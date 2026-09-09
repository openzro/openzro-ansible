# openzro-ansible

Ansible playbooks for installing the openZro server stack
(`management`, `signal`, `relay`, `dashboard`) on bare-metal Linux
hosts.

The roles wrap the native server packages produced by
[openzro/openzro](https://github.com/openzro/openzro):

| Component  | Debian / Ubuntu             | Fedora / RHEL / Rocky / Alma    | Source         |
|------------|-----------------------------|---------------------------------|----------------|
| management | `openzro-management` deb    | `openzro-management` rpm        | pkg.openzro.io |
| signal     | `openzro-signal` deb        | `openzro-signal` rpm            | pkg.openzro.io |
| relay      | `openzro-relay` deb         | `openzro-relay` rpm             | pkg.openzro.io |
| dashboard  | `openzro-dashboard` deb     | **no rpm** — container instead  | GHCR           |

That last row is the one asymmetry worth knowing before you start:
pkg.openzro.io's rpm repo publishes `openzro`, `openzro-management`,
`openzro-relay`, `openzro-signal` and `openzro-ui` — and no
dashboard. (`openzro-ui` is the desktop GUI client, not the web
dashboard; the names invite the mistake.) On RHEL-family hosts the
dashboard therefore runs as a container from
`ghcr.io/openzro/dashboard`, under **podman or docker**, with nginx
proxying to it. See [Dashboard install
methods](#dashboard-install-methods).

Server packages land in **v0.53.1-alpha.9** and onward — see
[`release_files/`](https://github.com/openzro/openzro/tree/main/release_files)
in the openzro repo for the systemd units, env files, and
example configs that these roles override.

## Supported platforms

| Platform | Status | Notes |
|---|---|---|
| Debian 12+ / Ubuntu 22.04+ | Supported | All four components from packages. |
| Fedora 41–44 | Supported | Dashboard via podman/docker. `dnf5` — needs ansible-core ≥ 2.18 on the control node. |
| RHEL / Rocky / AlmaLinux 9+ | Supported | Dashboard via podman/docker. certbot comes from EPEL, which the `common` role enables. |
| CentOS Stream 9+ | Should work | Same path as Rocky; not routinely exercised. |
| Arch | Untested | Roles install via `ansible.builtin.package`, so nothing blocks it, but there's no openzro package in the Arch repos. |

**Control node**: ansible-core **≥ 2.18**. That's not a style
preference — Fedora 41+ and RHEL 10 use dnf5, and 2.18 is the release
where `ansible.builtin.package` learned to dispatch to the `dnf5`
module. On an older ansible-core the RPM tasks fail (or, worse,
silently no-op). Collections: `ansible-galaxy install -r
requirements.yml`.

## Layout

```
inventories/
  lab/         # personal lab cluster (single host, all-in-one)
  production/  # multi-host, HA candidate
playbooks/
  site.yml         # full stack provisioning
  update.yml       # rolling update with cloud LB drain/undrain
  management.yml   # just management
  signal.yml       # just signal
  relay.yml        # just relay
  dashboard.yml    # just dashboard
roles/
  common/                # package repo + GPG key, EPEL on EL,
                         #   firewalld ports, SELinux booleans
  openzro_management/    # mgmt daemon, postgres DSN, OIDC, datastore,
                         #   cluster coordinator wiring (embedded NATS by default)
  openzro_signal/        # stateless rendezvous server
  openzro_relay/         # WireGuard relay (TURN-like)
  openzro_dashboard/     # package, podman or docker — see below
  openzro_nginx/         # nginx in front, certbot HTTP-01/DNS-01/BYO/self-signed
  openzro_nats_cluster/  # OPTIONAL — standalone nats-server cluster between
                         #   management hosts (alt to embedded NATS)
  openzro_redis_cluster/ # OPTIONAL — standalone redis (master/replica) between
                         #   management hosts
  aws_lb/                # ALB + NLB + ACM + Route53 (when openzro_cloud=aws)
  gcp_lb/                # HTTPS LB + NLB + managed cert + Cloud DNS (gcp)
```

## Quick start (lab)

```sh
# 0. Control node: ansible-core >= 2.18, plus the collections.
ansible-galaxy install -r requirements.yml
# 1. Inventory: edit inventories/lab/hosts.yml — point at your host(s).
#    The default expects a single all-in-one host called `openzro1`.
# 2. Group vars: copy + edit inventories/lab/group_vars/all.yml.example
#    to inventories/lab/group_vars/all.yml. Required values:
#      - openzro_public_domain (e.g. openzro.example.com)
#      - openzro_oidc_issuer / client_id / client_secret
#      - openzro_admin_email
#    See the file's comments for the rest.
# 3. Run:
ansible-playbook -i inventories/lab playbooks/site.yml
```

Nothing distro-specific to set: the roles detect the target's family
and pick package names, config paths, and the dashboard install
method to match. On a Fedora or RHEL host that also means firewalld
ports and the SELinux boolean nginx needs — see [RHEL / Fedora
notes](#rhel--fedora-notes).

The role idempotently:

- Adds the pkg.openzro.io APT or RPM repo + imports the signing key
- Enables EPEL on Enterprise Linux (certbot lives there)
- Opens the ports the host's inventory groups imply, in firewalld
- Sets `httpd_can_network_connect` where SELinux is enforcing
- Installs the three native packages
- Renders `/etc/openzro/management.json` from the role's template
- Drops `/etc/default/openzro-{management,signal,relay}` env files
- Installs the dashboard — package, podman or docker, per below
- `daemon-reload` + `enable --now` the systemd units

## Dashboard install methods

`openzro_dashboard_install_method` takes four values:

| Value | What runs | Where it works |
|---|---|---|
| `auto` *(default)* | `package` on Debian-family hosts, `podman` everywhere else | Everywhere — it encodes the table below so you don't have to |
| `package` | `openzro-dashboard` deb; nginx serves the static bundle off disk | **APT only.** There is no dashboard rpm; the role asserts rather than failing halfway |
| `podman` | `ghcr.io/openzro/dashboard` as a systemd quadlet, published on `127.0.0.1:8080`; nginx proxies to it | Anywhere with podman ≥ 4.4 (quadlet) |
| `docker` | Same image, same port, wrapped in a systemd unit the role writes | Anywhere with a docker daemon |

podman and docker are equally first-class: same unit name
(`openzro-dashboard.service`), same lifecycle, same env file, same
loopback publish, so `systemctl status openzro-dashboard`, the
handlers, and `update.yml` behave identically either way. podman is
the default under `auto` on RHEL-family hosts because it's in the
distro repos, is rootless-capable, and needs no daemon; pick `docker`
when the host already runs one and shouldn't grow a second container
runtime.

```yaml
# Explicitly choose docker on a host that already has it
openzro_dashboard_install_method: docker
# Only if you want an image tag that doesn't track openzro_version —
# by default it follows it, rewritten into the OCI spelling (`~` is
# not legal in an image tag), or "latest" when nothing is pinned.
openzro_dashboard_image_tag: "0.53.1-alpha.97"
# Only if you're changing the loopback port; must match
# openzro_nginx_dashboard_upstream
openzro_dashboard_publish: "127.0.0.1:8080:80"
```

The nginx role picks `static` vs `proxy` serving from the same rule,
so the two stay in sync without extra configuration. Override with
`openzro_nginx_dashboard_mode` if you're doing something unusual.

One caveat on updates: with the tag left at `latest`, a re-run
changes nothing in the unit, so nothing restarts and no newer image
is pulled — the container keeps running whatever it started with.
Pin `openzro_dashboard_image_tag` (or `openzro_version`) and the tag
change is what drives the restart and the pull. If you must track
`latest`, force it with `systemctl restart openzro-dashboard`; the
docker unit re-pulls on every start, and for podman add
`AutoUpdate=registry` to the quadlet.

A native dashboard rpm is on the openzro/openzro roadmap. When it
lands, RHEL-family hosts can switch to `package` by setting the var —
no role changes needed.

## RHEL / Fedora notes

Everything in this section is handled by the `common` role; it's
documented because these are the things that silently break a
Debian-shaped playbook when it first meets a RHEL-family host.

**Package manager.** Fedora 41+ and RHEL 10 report
`ansible_pkg_mgr == "dnf5"`. Every task in this repo installs through
`ansible.builtin.package`, which dispatches correctly; conditionals
are written against `ansible_os_family`. Gating on `ansible_pkg_mgr in
["dnf", "yum"]` — which this repo used to do — matches nothing on
those hosts, and the play then *succeeds* having installed nothing.

**nginx layout.** RHEL-family nginx has no
`sites-available`/`sites-enabled`; `nginx.conf` includes
`/etc/nginx/conf.d/*.conf`. The role writes
`/etc/nginx/conf.d/openzro.conf` there and skips the symlink step.

**firewalld** is enabled by default and drops everything this stack
needs. `common` opens the ports the host's groups imply — 80 + 443 on
the dashboard host, 33080 tcp+udp on relays, the cluster port between
management hosts — permanently and immediately, so the health checks
later in the play can actually reach the service. Set
`openzro_manage_firewall: false` to leave the host firewall alone,
`openzro_firewall_expose_backends: true` when management and signal
live on separate hosts and nginx has to cross the network to reach
them.

**SELinux.** With SELinux enforcing, nginx cannot open an outbound
socket, so every proxied request to management, signal, or the
dashboard container 502s — with nothing wrong in any config file.
`common` sets `httpd_can_network_connect` persistently on hosts in the
`dashboard` group. Disable with `openzro_manage_selinux: false`.

**EPEL.** certbot and its DNS plugins aren't in the Enterprise Linux
base repos, so `common` installs `epel-release` on EL (not on Fedora —
its repos carry current versions of both). On Rocky / Alma / CentOS
Stream this resolves from `extras`; on RHEL proper, either enable EPEL
per Red Hat's documentation first or point
`openzro_epel_release_package` at the release RPM's URL.

Note also that EPEL's certbot lags upstream far enough that recent DNS
plugins can reject its CLI arguments. The `openzro_relay` role works
around this on EL by installing certbot plus the plugin into a
`/opt/certbot` venv and symlinking the binary into `/usr/bin` — see
[`roles/openzro_relay/README.md`](roles/openzro_relay/README.md).
Fedora needs none of that.

**Redis.** Fedora ships **valkey**, not redis — the fork that followed
the licence change. `openzro_redis_cluster` selects package, service,
user and config path accordingly; the coordinator speaks the Redis
protocol, so valkey is a drop-in. EL uses `redis` from AppStream,
Debian `redis-server`.

**NATS.** Synadia publishes an apt repo but no rpm one, so RHEL-family
hosts get the upstream release tarball into `/usr/local/bin` (version
and optional checksum in `openzro_nats_version` /
`openzro_nats_checksum`).

## Topology assumptions

- `management` and `signal` need to be reachable from clients on
  ports 33073 (mgmt gRPC + REST) and 10000 (signal gRPC).
- `relay` needs an externally-resolvable hostname (`OZ_EXPOSED_ADDRESS`)
  so peers can dial it; UDP/TCP 33080 is the default.
- TLS termination happens at a load balancer / nginx in front of the
  hosts. The role can optionally generate a self-signed cert for
  dev/lab use — `openzro_tls_mode: self_signed`.

## High availability — what works today vs. pending

The four components have different statefulness, so HA support
varies:

| Component | Stateless? | Multi-replica today? | Notes |
|---|---|---|---|
| `signal` | ✅ | ✅ Yes | Drop N hosts in the `signal` group; the cloud LB role distributes traffic. Clients fall over via `Signal.URI` rotation in `management.json`. |
| `relay` | ✅ | ✅ Yes | Clients learn all relays via `management.json`'s `Relay.Addresses[]` and probe multiple in parallel. Add hosts to the `relay` group; the NLB load-balances. |
| `dashboard` | ✅ (static files) | ✅ Yes | Multiple controllers all serve identical static files; nginx does the work. |
| `management` | ❌ (writes datastore) | ✅ Yes via cluster coordinator | Multiple `management` hosts in the inventory get a coordinator wired automatically — distributed lock + state pub/sub. Choose backend with `openzro_cluster_backend` (see below). |

For **most deployments** (small to medium), the right shape is:

- 1 controller running `management` + `signal` + `dashboard` + nginx
- 2-3 relay hosts in different regions
- Postgres on RDS / Cloud SQL (managed; do not run on the same VM)

This is what `inventories/production/` is templated for. Single
`management` replica is the SPOF — for most teams that's fine
(brief outages during upgrades via the rolling-update playbook are
the only downtime).

For **HA / 24×7 SLAs**, set 2+ hosts in the `management` group and
pick a cluster coordinator backend (next section).

## Cluster coordinator (HA management)

When the `management` group has more than one host, the
coordinator is required so replicas don't corrupt shared state.
The role auto-defaults to **`embedded`** — no extra deps, fastest
to stand up. Override via `openzro_cluster_backend` in
`group_vars/all.yml`:

| `openzro_cluster_backend` | What gets installed | When to use |
|---|---|---|
| `none` (auto for single-replica) | Nothing — coordinator is nil | Single `management` host |
| `embedded` (auto for HA) | Nothing extra — `openzro-mgmt` boots an internal NATS+JetStream server, instances gossip on tcp/6222 | Default for HA. Zero extra deps. JetStream included. |
| `nats` | Standalone `nats-server` daemon on every management host (via the `openzro_nats_cluster` role) | Operators who want the broker as a separately-monitored process, separate restart cycle, dedicated logs |
| `redis` | Standalone `redis-server` master/replica across management hosts (via the `openzro_redis_cluster` role) | Teams already running Redis observability tooling |

**Embedded NATS** is the right call 90% of the time. Each
`openzro-mgmt` process boots a NATS+JetStream server bound to
loopback, joined to the cluster on a peer port (default 6222).
The role auto-derives the peer list from the inventory's
`management` group — no manual config beyond the inventory.

Firewall rule needed: tcp/6222 between management hosts (or the
port chosen via `openzro_cluster_peer_port`). On RHEL/Fedora the
`common` role opens this in firewalld automatically once the
`management` group has more than one host; elsewhere it's on you or
your cloud security group.

For external NATS or Redis (managed brokers, Elasticache /
Memorystore), set the backend to `nats` or `redis` and configure
`openzro_cluster_nats_url` / `openzro_cluster_redis_url`. The
optional install roles only run when you ask the playbook to put
the broker on the controller hosts themselves.

## What's NOT here yet

- **Postgres bootstrap** — assumed managed (RDS / Cloud SQL) or
  manually installed. No `openzro_postgres` role yet.
- **Backup / restore** for the management datastore.
- **Sentinel-based automatic failover** for `openzro_redis_cluster`.
  Today it's master/replica with manual promotion (set
  `openzro_redis_master_host` to the new master and re-run the
  playbook). For real HA Redis, use a managed service.

See [openzro/openzro/docs/adr/0008](https://github.com/openzro/openzro/blob/main/docs/adr/0008-k8s-deployment.md)
for the K8s/helm-chart parallel — the Ansible flow targets the same
production-shape but on bare metal.

## Rolling updates (zero-downtime)

For HA deployments behind an LB, use `playbooks/update.yml` —
**not** `site.yml` — to upgrade. The update playbook does the
drain/upgrade/undrain dance per host:

```sh
ansible-playbook -i inventories/prod playbooks/update.yml \
    -e openzro_version=v0.53.1-alpha.97
```

**Paste the version in whatever form you have it.** All of these mean
the same thing to the roles:

| you have | from |
|---|---|
| `v0.53.1-alpha.97` | a git tag, copied verbatim |
| `0.53.1-alpha.97` | a release page, without the `v` |
| `0.53.1~alpha.97` | the spelling the packages are published under |

A leading `v` is stripped, and a `-` before a pre-release word
(`alpha`, `beta`, `rc`) becomes `~`. That `~` is the character that
sorts *before* the final release in both dpkg and rpm; a plain `-`
sorts after, which would make `0.53.1-alpha.97` compare as **newer**
than `0.53.1` — the opposite of what a pre-release means. That's why
the packages use it, and why you no longer have to.

A Debian-style upstream revision is left alone: `0.53.1-1` and
`0.53.1-2ubuntu1` pass through untouched, since only a recognised
pre-release word is rewritten. That hyphen separates the upstream
version from the packaging revision and means something different.

The roles append the version with the separator each package manager
wants — `pkg=<v>` for apt, `pkg-<v>` for dnf — so `openzro_version`
carries the bare version either way. The dashboard container gets the
same version in the OCI spelling, since `~` is not legal in an image
tag.

Per host, in order:

1. Deregister the host from the cloud LB target pool
2. Wait for the `deregistration_delay` (AWS) or
   `connection_draining_timeout_sec` (GCP) — in-flight requests
   finish without being severed
3. Run the role tasks (package upgrade + systemd restart; on the
   dashboard host that's an image pull + container restart when
   the install method is podman or docker)
4. Wait for the local service to bind its port
5. Re-register the host with the LB target pool
6. Wait for the LB health check to mark the host `healthy`
7. Move to the next host

`serial: 1` on each play guarantees only one host is out of
rotation at a time — pace governed by your LB's drain timeout
plus the package install time. Plan ~2–3 minutes per host with
default settings.

The control plane (mgmt + signal + dashboard) updates first,
then the relay tier — separate play so the playbook doesn't take
mgmt down at the same time as a relay (peers reconnecting fall
back to other relays, but mgmt downtime affects new peer joins).

**Required per-host vars for the update:**

```yaml
# inventories/prod/hosts.yml — per host
controller1:
  ansible_host: …
  aws_instance_id: i-0123456789abcdef0   # for openzro_cloud=aws
  # OR
  gcp_instance_name: openzro-controller1 # for openzro_cloud=gcp
```

Without those, the drain/undrain steps skip with a warning and
the playbook upgrades in-place — which is the right behaviour
for the lab/single-host case (no LB to drain from anyway).

### What if a host fails mid-upgrade?

`any_errors_fatal: true` halts the play immediately. The current
host stays drained. Diagnose, fix the issue, then re-run the same
playbook — the host re-registers in the post_tasks of the next
run.

### Single-host (no LB) deployments

Re-run `playbooks/site.yml`. The roles' `notify: restart …`
handlers cause a brief restart; in-flight requests fail (~5–10s)
but everything else stays put. No drain logic is exercised because
there's nowhere to drain to.

## Recommended infrastructure sizing

The numbers below are starting points based on observed footprints
of the daemons under typical traffic. Tune up if your tracing /
metrics show pressure. **The three daemons — management, signal,
relay — run as native systemd units with no container layer**;
overhead is the binary itself, not a runtime + image. The dashboard
is a static SPA either way: served off disk under the `package`
method, or from a small nginx container under podman/docker, where
the runtime adds a few tens of MB of RAM and nothing meaningful in
CPU.

### Lab / proof-of-concept (< 100 peers)

A single all-in-one VM is plenty:

| Resource | Size |
|---|---|
| CPU | 2 vCPU |
| RAM | 4 GB |
| Disk | 40 GB SSD |
| Network | 1 Gbps |

Components colocated on that one host:
`management` + `signal` + `relay` + `dashboard` + `nginx` +
`SQLite` (no Postgres needed yet).

Use the `inventories/lab/` template. Self-signed cert is fine.

### Small production (100 – 1 000 peers)

Split control plane from data plane:

| Role | Count | CPU | RAM | Disk | Notes |
|---|---|---|---|---|---|
| Controller | 1 | 2 vCPU | 4 GB | 60 GB SSD | mgmt + signal + dashboard + nginx |
| Gateway (relay) | 2 | 1 vCPU | 2 GB | 20 GB | distributed across 2 regions / AZs for failover |
| Postgres | 1 | 2 vCPU | 4 GB | 50 GB SSD | managed (RDS / Cloud SQL) preferred over self-hosted |

The mgmt + signal + dashboard share one host because they're
control-plane (low traffic, mostly idempotent state). Relays are
data-plane (bandwidth-bound) and benefit from being independent
hosts on different networks — peers behind symmetric NAT fall
back to whichever relay they reach first.

Cert via certbot HTTP-01 (default `openzro_tls_mode`).

### Medium production (1 000 – 10 000 peers)

| Role | Count | CPU | RAM | Disk | Notes |
|---|---|---|---|---|---|
| Controller (HA pair) | 2 | 4 vCPU | 8 GB | 100 GB SSD | behind cloud LB; mgmt + signal share, dashboard on either |
| Gateway (relay) | 3 | 2 vCPU | 4 GB | 40 GB | one per region; multi-AZ within region |
| Postgres (HA) | 1 cluster | 4 vCPU | 8 GB | 100 GB SSD | RDS Multi-AZ or Patroni; daily snapshots |

The HA controller pair runs management + signal in active-active
mode behind a cloud LB — see the BACKLOG entry on HA. Dashboard
runs on either controller (stateless).

Use the `inventories/production/` template with `openzro_cloud:
aws` or `openzro_cloud: gcp` to wire up the LB role.

### Large production (10 000+ peers)

Custom — if you've grown past medium, the bottleneck is usually
Postgres + the bandwidth on the relay tier, not the management
service. Profile first, then scale the relay fleet horizontally
(adding more gateway hosts in the same region balances better
than scaling up).

### Per-component notes

| Component | What it does | Resource bound |
|---|---|---|
| `management` | gRPC + REST control API, peer/route/policy state, JWT validation against the IdP | CPU on JWT validation hot path; RAM scales with peer count |
| `signal` | Stateless rendezvous for WireGuard handshake messages | CPU on TLS termination on burst (peer reconnects); RAM minimal |
| `relay` | Forwards WireGuard between peers behind NAT (TURN-like) | Bandwidth-bound; RAM minimal |
| `dashboard` | Static SPA served by nginx | Trivial — 256 MB RAM serves thousands of admin sessions |

### Networking recommendations

- **Controller**: ports 80 + 443 public (nginx terminates TLS),
  443 forwards to mgmt gRPC + REST + signal gRPC + dashboard.
- **Relay**: UDP 33080 + TCP 33080 public. Direct, NOT through
  nginx — relay is L4.
- **Postgres**: never public. Private network only between
  controller and DB.
- **Cloud LB**: terminate TLS at LB OR pass-through to nginx;
  most operators terminate at LB for ACM/managed cert reasons.
  When passing through, nginx still does the TLS handshake.

## Routing peers (the `routing_peer` group)

Routing peers are Linux hosts whose only job is to expose private
CIDRs (VPC subnets, on-prem LANs) to the rest of the mesh. They
run the openzro client, NOT one of the control-plane components.
The CIDRs they attract are NOT declared in this Ansible repo —
they're configured on the management side as Network Resources
hung off the group attached to the routing peer's setup key.

So the responsibility split is:

| Where | What |
|---|---|
| **Ansible (this repo)** | Provision the host, install the openzro client, enable IP forwarding, log the peer in with a setup key |
| **Management (dashboard / API / GitOps)** | Define which CIDRs the peer's group exposes, tie ACL policies, manage peer membership |

This separation means scaling horizontally is purely an inventory
edit: add a host to the `routing_peer` group, re-run the
playbook, and the new replica joins the same router pool.

### Variables (group_vars/<env>/all.yml)

```yaml
openzro_management_url: "https://zt.example.com"

# Reusable, NON-ephemeral setup key whose auto_groups list
# contains the group bound to the network router on the
# management side. Vault this.
openzro_routing_peer_setup_key: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  ...
```

Per-host overrides go in `host_vars/<hostname>.yml` — typical
case is `openzro_routing_peer_hostname` so the dashboard shows
something more meaningful than the inventory FQDN.

### Capacity per host

A routing peer is CPU-bound on WireGuard encryption: roughly **1
vCPU per 1 Gbps of relayed traffic** with modern AES-NI. RAM and
disk are minimal — 2 GB / 20 GB covers a few thousand
simultaneous flows. For tighter sizing pick from the table at the
top of [Recommended infrastructure sizing](#recommended-infrastructure-sizing).

### Sample VM shapes

| Cloud | Up to 1 Gbps / host | Up to 5 Gbps / host |
|---|---|---|
| AWS | `t3a.medium` (2 vCPU, 4 GB) | `c6i.xlarge` (4 vCPU, 8 GB) |
| GCP | `e2-medium` (2 vCPU, 4 GB) | `c3-standard-4` (4 vCPU, 16 GB) |
| Azure | `B2s` (2 vCPU, 4 GB) | `D4s_v5` (4 vCPU, 16 GB) |
| Bare metal | 2 cores, 4 GB, 1 GbE | 4 cores, 8 GB, 10 GbE |

Network egress is the real bill driver — locate routing peers in
the same region as the workloads they front to keep egress
charges off your invoice.

## AWX / Ansible Tower

This repo is shaped to run as-is from AWX. Reference setup:

### 1. Project

- **SCM type**: `Git`
- **URL**: your fork of this repo
- **Branch**: `main`
- **Update revision on launch**: ✅ (so a job uses the most-recent
  push without an explicit project sync)

### 2. Inventory

One inventory per environment (`prod`, `staging`, `lab`). Sources:

- **From source control**: point at `inventories/<env>/hosts.yml`,
  same project as above.
- **OR cloud-dynamic**: gcp_compute / aws_ec2 plugin scoped by
  tag (`role=openzro-routing-peer`) — keeps the inventory in
  lock-step with the actual fleet.

### 3. Credentials

- **Machine**: SSH private key for `ansible_user`
- **Vault**: the password used to encrypt `all.yml` — AWX will
  unwrap automatically when the job references vault-encrypted
  vars

### 4. Job templates

| Name | Playbook | Inventory | Limit | Notes |
|---|---|---|---|---|
| `openzro-deploy-control-plane` | `playbooks/site.yml` | prod | (none) | Full stack — first-time bring-up |
| `openzro-deploy-relays` | `playbooks/relay.yml` | prod | `relay` | Adding/replacing relay tier |
| `openzro-deploy-routing-peers` | `playbooks/routing_peer.yml` | prod | `routing_peer` | Adding a host or rotating setup keys |
| `openzro-update` | `playbooks/update.yml` | prod | (none) | Rolling minor-version upgrade |

### 5. Surveys (optional)

Useful when you want operators to launch a job without touching
git. Example for `openzro-deploy-routing-peers`:

| Survey question | Variable | Default | Required |
|---|---|---|---|
| Target hostname (limit) | `target_host` |  | ✅ |
| openzro version | `openzro_version` | `v0.53.1-alpha.97` |  |

The setup key stays in vault — operators don't see or paste it.

### 6. Schedules

- Nightly `openzro-update` against staging at 03:00 UTC catches
  package-repo regressions before they hit prod.
- Weekly drift-detection run of `openzro-deploy-control-plane`
  with `--check --diff` flags surfaces config drift.
