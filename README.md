# openzro-ansible

Ansible playbooks for installing the openZro server stack
(`management`, `signal`, `relay`, `dashboard`) on bare-metal Linux
hosts (Debian/Ubuntu, RHEL/Rocky/Fedora).

The roles wrap the native server packages produced by
[openzro/openzro](https://github.com/openzro/openzro):

| Component  | Package                                              | Source           |
|------------|------------------------------------------------------|------------------|
| management | `openzro-management_<v>_amd64.deb` / `.rpm`          | pkg.openzro.io   |
| signal     | `openzro-signal_<v>_amd64.deb` / `.rpm`              | pkg.openzro.io   |
| relay      | `openzro-relay_<v>_amd64.deb` / `.rpm`               | pkg.openzro.io   |
| dashboard  | `ghcr.io/openzro/dashboard:<v>` (container, for now) | GHCR             |

Server packages land in **v0.53.1-alpha.9** and onward — see
[`release_files/`](https://github.com/openzro/openzro/tree/main/release_files)
in the openzro repo for the systemd units, env files, and
example configs that these roles override.

## Layout

```
inventories/
  lab/         # personal lab cluster (single host, all-in-one)
  production/  # multi-host, HA candidate
playbooks/
  site.yml         # full stack on the inventory
  management.yml   # just management
  signal.yml       # just signal
  relay.yml        # just relay
  dashboard.yml    # just dashboard
roles/
  common/              # apt/yum repo setup, GPG key import
  openzro_management/  # postgres DSN, dex grpc, JWT, datastore
  openzro_signal/      # stateless rendezvous server
  openzro_relay/       # WireGuard relay (TURN-like)
  openzro_dashboard/   # nginx + container OR future deb/rpm pkg
```

## Quick start (lab)

```sh
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

The role idempotently:

- Adds pkg.openzro.io APT/YUM repo + imports the signing key
- Installs the three native packages
- Renders `/etc/openzro/management.json` from the role's template
- Drops `/etc/default/openzro-{management,signal,relay}` env files
- `daemon-reload` + `enable --now` the systemd units

The dashboard role currently runs the upstream container via
`docker compose` (single-host) or a `Pod` manifest (multi-host).
A native server package is on the openzro/openzro roadmap — when it
lands, `roles/openzro_dashboard/tasks/main.yml` switches to apt/dnf
install with the same template-driven approach as the other three.

## Topology assumptions

- `management` and `signal` need to be reachable from clients on
  ports 33073 (mgmt gRPC + REST) and 10000 (signal gRPC).
- `relay` needs an externally-resolvable hostname (`OZ_EXPOSED_ADDRESS`)
  so peers can dial it; UDP/TCP 33080 is the default.
- TLS termination happens at a load balancer / nginx in front of the
  hosts. The role can optionally generate a self-signed cert for
  dev/lab use — `openzro_self_signed_tls: true`.

## High availability — what works today vs. pending

The four components have different statefulness, so HA support
varies:

| Component | Stateless? | Multi-replica today? | Notes |
|---|---|---|---|
| `signal` | ✅ | ✅ Yes | Drop N hosts in the `signal` group; the cloud LB role distributes traffic. Clients fall over via `Signal.URI` rotation in `management.json`. |
| `relay` | ✅ | ✅ Yes | Clients learn all relays via `management.json`'s `Relay.Addresses[]` and probe multiple in parallel. Add hosts to the `relay` group; the NLB load-balances. |
| `dashboard` | ✅ (static files) | ✅ Yes | Multiple controllers all serve identical static files; nginx does the work. |
| `management` | ❌ (writes datastore) | ⚠️ **Pending** — needs cluster coordinator | Today the role writes a single-node `management.json`. Multiple management replicas writing to the same Postgres without a distributed lock (Redis / NATS / pg-advisory) corrupts state. |

For **most deployments** (small to medium), the right shape is:

- 1 controller running `management` + `signal` + `dashboard` + nginx
- 2-3 relay hosts in different regions
- Postgres on RDS / Cloud SQL (managed; do not run on the same VM)

This is what `inventories/production/` is templated for. Single
`management` replica is the SPOF — for most teams that's fine
(brief outages during upgrades via the rolling-update playbook are
the only downtime).

For **large deployments** or 24×7 SLAs the management tier needs
2+ replicas + a cluster coordinator. That work is tracked under
the HA story below.

## What's NOT here yet

- **HA management** with cluster coordinator. The `production`
  inventory has the `management` group but the role's
  `management.json.j2` template doesn't emit a `Cluster:` block
  yet — adding it (Redis-backed, mirrors the helm chart) is ~6-8h
  of work + integration testing.
- **Postgres bootstrap** — assumed managed (RDS / Cloud SQL) or
  manually installed. No `openzro_postgres` role yet.
- **Backup / restore** for the management datastore.

See [openzro/openzro/docs/adr/0008](https://github.com/openzro/openzro/blob/main/docs/adr/0008-k8s-deployment.md)
for the K8s/helm-chart parallel — the Ansible flow targets the same
production-shape but on bare metal.

## Rolling updates (zero-downtime)

For HA deployments behind an LB, use `playbooks/update.yml` —
**not** `site.yml` — to upgrade. The update playbook does the
drain/upgrade/undrain dance per host:

```sh
ansible-playbook -i inventories/prod playbooks/update.yml \
    -e openzro_version=0.53.1-alpha.X
```

Per host, in order:

1. Deregister the host from the cloud LB target pool
2. Wait for the `deregistration_delay` (AWS) or
   `connection_draining_timeout_sec` (GCP) — in-flight requests
   finish without being severed
3. Run the role tasks (apt/yum/pacman upgrade + systemd restart)
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
metrics show pressure. **All daemons run as native systemd units
with no container layer** — overhead is the binary itself, not a
runtime + image.

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
