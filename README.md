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

## What's NOT here yet

- HA story: multi-replica with Postgres + cluster coordinator. The
  `production` inventory has the host groups (`management`, `signal`,
  `relay`, `dashboard`) but the role currently writes a single-node
  config. Postgres + Redis bootstrap is a follow-up.
- Cert-manager-style cert lifecycle — for now bring your own cert
  via `openzro_tls_cert_path` / `openzro_tls_key_path`.
- Backup / restore for the management datastore.

See [openzro/openzro/docs/adr/0008](https://github.com/openzro/openzro/blob/main/docs/adr/0008-k8s-deployment.md)
for the K8s/helm-chart parallel — the Ansible flow targets the same
production-shape but on bare metal.
