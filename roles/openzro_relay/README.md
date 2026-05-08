# openzro_relay

Installs the [openZro relay](https://github.com/openzro/openzro/tree/main/relay)
binary on a Linux host as a systemd unit. The relay is the bytes-forwarder
peers fall back to when direct WireGuard punching fails (symmetric NAT,
strict corporate egress, etc).

By default this role only manages the daemon — TLS for the relay's
public endpoint is assumed to be terminated upstream (cloud LB,
[`openzro_nginx`](../openzro_nginx/README.md), or a manually-staged
cert). For deployments where the relay terminates TLS directly on
host:443 — typically the multi-region pattern where each pod IS the
LB target — the role can also issue + renew a Let's Encrypt cert via
DNS-01 challenge. See [Certbot pipeline](#certbot-pipeline) below.

## Required variables

| Variable | Used | Notes |
|---|---|---|
| `openzro_relay_auth_secret` | always | HMAC shared with the management server (`relay.secret` in the chart). Vault. |
| `openzro_relay_public_address` | always | FQDN peers connect to. Falls back to `openzro_public_domain`. |
| `openzro_relay_acme_email` | when `cert_provider != none` | ACME registration email. |

## Common knobs

| Variable | Default | Notes |
|---|---|---|
| `openzro_relay_listen_address` | `0.0.0.0:33080` | What the systemd unit binds to. |
| `openzro_relay_port` | `33080` | Public port emitted in the exposed-address URL. |
| `openzro_relay_group` | `openzro` | Posix group the daemon runs as (matches the package's post_install). |
| `openzro_relay_cluster_headless` | `""` | Optional second SAN — the cluster-headless DNS name when running multiple relay pods behind a round-robin record (see ADR-0014 in `openzro/openzro`). |

## Certbot pipeline

Set `openzro_relay_cert_provider` to one of the values below to have
the role issue + renew a Let's Encrypt cert via DNS-01:

| Provider | Auth path | Default propagation |
|---|---|---|
| `none` *(default)* | — | — |
| `certbot_dns_google` | GCE metadata service account (zero on-disk creds) | 60s |
| `certbot_dns_route53` | EC2 instance role *or* static AWS access keys | 30s |
| `certbot_dns_cloudflare` | API token at `/etc/letsencrypt/cloudflare.ini` | 10s |

When set to anything other than `none`, the role:

1. Installs certbot + the matching DNS plugin
   (apt path: distro packages; RHEL/Rocky path: a `/opt/certbot`
   venv with the binary symlinked to `/usr/bin/certbot` because
   the EPEL `certbot` RPM's Python isolation can't load
   pip-installed plugins).
2. Issues the cert with `--expand` and both
   `openzro_relay_public_address` + `openzro_relay_cluster_headless`
   as `-d` SANs.
3. Wires a deploy-hook at
   `/etc/letsencrypt/renewal-hooks/deploy/openzro-relay.sh` that
   on every successful renewal re-applies `chgrp openzro` /
   `chmod 0640` to the new privkey files (certbot lands them at
   `0600` by default) and runs
   `systemctl reload-or-restart openzro-relay` so the daemon picks
   up the rotated cert.

The renewal timer (`certbot.timer` on Debian, `certbot-renew.timer`
on RHEL) ships with the certbot package itself; the role only makes
sure it's enabled.

### Provider-specific knobs

#### Google Cloud DNS

```yaml
openzro_relay_cert_provider: certbot_dns_google
openzro_relay_acme_email: ops@example.com

# Optional: only when the Cloud DNS zone is in a different GCP
# project than the VM. Empty = use the VM's metadata project.
openzro_relay_certbot_dns_google_project: "infra-prod-dns"
```

The VM's instance service account needs `roles/dns.admin` on the
zone (or a custom role with `changes.create` + `changes.get` +
`managedZones.list`).

#### AWS Route53

```yaml
openzro_relay_cert_provider: certbot_dns_route53
openzro_relay_acme_email: ops@example.com

# Path A: EC2 instance role attached — leave the access key vars empty.
# Path B: static keys (off-AWS or no instance role available):
openzro_relay_certbot_dns_route53_access_key_id: "{{ vault_aws_access_key_id }}"
openzro_relay_certbot_dns_route53_secret_access_key: "{{ vault_aws_secret_access_key }}"
openzro_relay_certbot_dns_route53_region: "us-east-1"
```

IAM permissions required: `route53:ChangeResourceRecordSets` and
`route53:GetChange` on the hosted zone covering
`openzro_relay_public_address`, plus `route53:ListHostedZones`
project-wide so the plugin can locate the zone.

#### Cloudflare

```yaml
openzro_relay_cert_provider: certbot_dns_cloudflare
openzro_relay_acme_email: ops@example.com
openzro_relay_certbot_dns_cloudflare_api_token: "{{ vault_cloudflare_token }}"
```

Generate the token at
<https://dash.cloudflare.com/profile/api-tokens> with **Zone:DNS:Edit**
scoped to the zone(s) covering `openzro_relay_public_address`. Don't
use the legacy "Global API Key" — that's a primary credential with
full account access.

### Bring-your-own-cert / external certbot

If you issue the relay's cert via some other means (corporate PKI,
shared certbot cron managed elsewhere, etc) but want the renewal
auto-restart behaviour, set:

```yaml
openzro_relay_cert_provider: none           # default — role doesn't issue
openzro_relay_certbot_renew_hook: true      # but DOES install the hook
```

The hook only fires when `RENEWED_LINEAGE` matches
`/etc/letsencrypt/live/<openzro_relay_public_address>` so unrelated
renewals on the same host don't bounce the relay.

### Force a one-shot reissue

```yaml
openzro_relay_certbot_force_renew: true
```

Useful when rotating the SAN list (e.g. adding a cluster headless
name later). Leave `false` after the one-shot completes — certbot
renews automatically via the timer.
