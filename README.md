# server-k3s

GitOps manifests for the `kingkiller` single-node k3s cluster, managed with
**Kustomize** and applied with **mise**. Domain: `neleoz.com` (Cloudflare DNS).

Companion repo: `server-iac` provisions the host, ZFS pool (`/tank/...`), and
installs k3s itself.

---

## Layers (applied in order)

| Layer | Namespace(s) | Contents |
|-------|--------------|----------|
| `00-infra` | `infra` (+ creates all namespaces) | sealed-secrets, reflector, cert-manager, cloudflared |
| `10-auth` | `auth` | lldap, authelia |
| `20-media` | `media` | jellyfin, sonarr, radarr, prowlarr, bazarr, seerr |
| `25-downloads` | `downloads` | qbittorrent, shelfmark |
| `30-cloud` | `cloud` | nextcloud (+db), immich (+db, redis, ml) |
| `40-web` | `web` | website |
| `50-apps` | `apps` | bambustudio, grimmory |
| `60-games` | `games` | minecraft |

### Namespace design
Grouped by **function / trust boundary**, not one-per-app, so NetworkPolicy,
RBAC, and quotas apply per group:
- `infra` — cluster plumbing/controllers
- `auth` — identity/SSO (other apps depend on it)
- `media` — streaming + *arr management
- `downloads` — internet-facing fetchers (qbittorrent, shelfmark), isolated
- `cloud` — personal data / PII (nextcloud, immich)
- `web` — public website
- `apps` — misc self-hosted apps
- `games` — game servers

---

## Ingress & exposure

- **Traefik** (k3s built-in) serves every app an internal ingress at
  `<app>.neleoz.com` with **cert-manager** Let's Encrypt TLS (DNS-01 via
  Cloudflare).
- **cloudflared** tunnel exposes only selected apps publicly:
  `shelfmark`, `seerr`, `grimmory` (nextcloud/immich ready to enable later).
  Edit `00-infra/cloudflared/configmap.yaml` to change the public set.
- **minecraft** is exposed via a k3s `LoadBalancer` on `25565` (raw TCP).

### Authentication (Authelia SSO)

Authelia is the single identity gateway, backed by **lldap**. Apps integrate in
one of two ways:

**Native OIDC / LDAP** (preferred — no proxy):
- **nextcloud** — `user_oidc` app, OIDC client `nextcloud`.
- **immich** — built-in OAuth, OIDC client `immich` (configure in admin UI).
- **seerr** (Jellyseerr) — built-in OIDC, client `jellyseerr`.
- **jellyfin** — LDAP plugin pointed straight at lldap (native media clients
  break behind a forward-auth proxy, so it must not be proxied).

OIDC clients are declared in `10-auth/authelia/configmap.yaml`. Each client
secret (hashed) plus the provider `oidc-hmac-secret` and
`oidc-issuer-private-key` live in the `authelia-secrets` SealedSecret.

**Traefik forwardAuth proxy** (apps with no native SSO): radarr, sonarr,
prowlarr, bazarr, qbittorrent, shelfmark, grimmory, bambustudio, lldap. These
opt in via the `traefik.ingress.kubernetes.io/router.middlewares:
<namespace>-authelia@kubernetescrd` annotation; each namespace ships its own
`authelia-middleware.yaml`.

Excluded from auth entirely: the public **website** and the **authelia** portal.

---

## Storage

- **Media / config / user data** → `hostPath` into the ZFS pool (`/tank/app/*`,
  `/tank/media/*`, `/tank/data/*`).
- **Databases** (nextcloud, immich, lldap, authelia) → `local-path` on the boot
  SSD for speed. Back up to `/tank/backup/db/*` (job not yet added).

---

## Prerequisites

1. k3s running on `kingkiller` (via `server-iac`).
2. Kubeconfig on your laptop. Copy from the server and point mise at it:
   ```bash
   mkdir -p .kube
   scp kingkiller:/etc/rancher/k3s/k3s.yaml .kube/config
   # edit .kube/config: replace 127.0.0.1 with the server IP/Tailscale name
   ```
   `.mise.toml` sets `KUBECONFIG` to `.kube/config`.
3. `mise install` (gets kubectl + kustomize).

---

## Secrets (sealed-secrets)

All secrets are committed as **SealedSecrets** (safe in git). They must be
generated against the live controller with `kubeseal` after `00-infra` is
applied. General flow:

```bash
# after mise run infra, once sealed-secrets controller is running:
kubectl create secret generic <name> \
  --namespace <ns> \
  --from-literal=<key>=<value> \
  --dry-run=client -o yaml \
  | kubeseal --format yaml > <path>/sealed-secret.yaml
```

Secrets to create (uncomment the `sealed-secret.yaml` line in each app's
`kustomization.yaml` once generated):

| Secret | Namespace | Keys |
|--------|-----------|------|
| `cloudflare-api-token` | `cert-manager` | `api-token` (Zone:DNS:Edit) |
| `cloudflared-credentials` | `infra` | tunnel `credentials.json` |
| `lldap-secrets` | `auth` | `jwt-secret`, `admin-password` |
| `authelia-secrets` | `auth` | `jwt-secret`, `session-secret`, `storage-encryption-key`, `ldap-password`, `postgres-password`, `oidc-hmac-secret`, `oidc-issuer-private-key`, `oidc-nextcloud-secret`, `oidc-immich-secret`, `oidc-jellyseerr-secret` |
| `nextcloud-secrets` | `cloud` | `admin-password`, `db-password` |
| `immich-secrets` | `cloud` | `db-password` |
| `grimmory-secrets` | `apps` | `db-password`, `db-root-password` |

---

## Usage

```bash
mise run build      # render all manifests locally (no apply)
mise run diff       # diff against the live cluster
mise run all        # apply everything, in order

# or per layer:
mise run infra
mise run auth
mise run media
mise run downloads
mise run cloud
mise run web
mise run apps
mise run games
```

**Order matters on first apply:** run `infra` first (installs CRDs for
cert-manager + sealed-secrets), create the SealedSecrets, then the rest.

---

## TODO / confirm before applying

- Generate all SealedSecrets and uncomment their references.
- Create the cloudflared tunnel + Cloudflare DNS records.
- Add DB backup CronJobs (dump SSD Postgres/MariaDB → `/tank/backup/db/*`).

### App notes (from upstream docs)
- **shelfmark** `ghcr.io/calibrain/shelfmark`, port 8084, health `/api/health`.
- **grimmory** `ghcr.io/grimmory-tools/grimmory`, port 6060, needs its own
  MariaDB (included in `db.yaml`), health `/api/v1/healthcheck`.
- **bambustudio** `lscr.io/linuxserver/bambustudio`, HTTPS on 3001 required —
  ingress talks HTTPS to the backend and skips cert verification; `/dev/shm`
  backed by 1Gi tmpfs.
