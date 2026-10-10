# kubernetes-php

The umbrella Helm chart for deploying a FrankenPHP/Laravel application stack on
Kubernetes — k3s for local development, any production cluster via Flux GitOps.
It composes every infrastructure dependency (Traefik, cert-manager, PostgreSQL,
Redis) and the FrankenPHP application subchart into one installable unit.

The same chart serves both environments. Environment differences are expressed
purely through values files and HelmRelease overlays, never through template
branching. Local is a single-node k3s cluster on the workstation; production is
a Flux-managed cluster — both reconciled from the same fleet repo.

Related repos:

- [kubernetes-base](https://github.com/99linesofcode/kubernetes-base) — the generic reusable charts.
- [kubernetes-fleet](https://github.com/99linesofcode/kubernetes-fleet) — the Flux GitOps fleet (releases, secrets).

## What the chart composes

The umbrella is named `laravel-stack` and owns five subcharts plus three
infrastructure templates.

Subcharts:

- **Traefik** (v36.2.0) — the Gateway API controller. Owns the Gateway and
  GatewayClass, terminates TLS, routes HTTP. Runs in hostNetwork mode on k3s to
  bind ports 80/443 directly; no Service is created for it.
- **cert-manager** (v1.17.2) — TLS certificate lifecycle. Local uses a
  self-signed CA chain; production uses Let's Encrypt with a Cloudflare DNS-01
  challenge.
- **PostgreSQL** (Bitnami, v16.7.2) — conditional (`postgresql.enabled`). Local:
  no persistence. Production: persistent volumes, password from SOPS.
- **Redis** (Bitnami, v27.0.18) — conditional (`redis.enabled`). Local: no auth,
  no persistence. Production: auth + persistence, password from SOPS.
- **FrankenPHP** (`charts/frankenphp/`) — the application: a Deployment, a
  ClusterIP Service, and an HTTPRoute on the Gateway's `websecure` listener.
  Local mounts the host code directory via hostPath for live reloading;
  production bakes the code into the image.

Infrastructure templates (owned by the umbrella, not a subchart):

- **ClusterIssuers** — three files, each gated by a values toggle:
  `self-signed-issuer.yaml` (bootstraps a self-signed issuer), `local-ca-issuer.yaml`
  (creates a CA Certificate, then a CA ClusterIssuer), `cloudflare-issuer.yaml`
  (Let's Encrypt ACME with Cloudflare DNS-01). Only the enabled issuers render.
- **Certificate** (`certificate.yaml`) — requests a TLS cert for
  `global.domain` from `global.issuer.name`; the resulting secret
  (`frankenphp-tls`) backs the Gateway's HTTPS listener.
- **HTTP→HTTPS redirect** (`httproute-redirect.yaml`) — a portable Gateway API
  `RequestRedirect` on the `web` listener.

Traffic: client → host port 80/443 (Traefik hostNetwork) → 80 redirects to 443
→ TLS terminated with the cert-manager cert → HTTPRoute to the FrankenPHP
Service → Laravel served from `/app/public`.

## Secrets

Secrets never sit in git as plaintext. SOPS encrypts `secrets/<environment>/*.yaml`
with age; each environment has its own recipient in `.sops.yaml`, so production
secrets cannot be decrypted with the local key. Flux's kustomize-controller
decrypts in-cluster using the age private key stored as a Secret in
`flux-system`. The chart references the decrypted secret via `existingSecret`
(Bitnami) or `secretKeyRef` (app env vars) — never a value in `values.yaml`.

## Configuration

Values layer, they don't branch:

1. `values.yaml` — shared defaults (Gateway, domain, Traefik, cert-manager,
   PostgreSQL/Redis defaults disabled, FrankenPHP defaults).
2. `values.local.yaml` — local overlay: PostgreSQL/Redis enabled without
   persistence, self-signed + local CA issuers, `hostPath` for live reloading.
3. `values.production.yaml` — production overlay: persistence, Cloudflare
   DNS-01 issuer, production domain and image, no hostPath.
4. The fleet HelmRelease adds the final layer per environment.

| Setting | Local | Production |
|---|---|---|
| `global.domain` | `frankenphp.99linesofcode.test` | `frankenphp.99linesofcode.nl` |
| `global.issuer.name` | `local-ca-issuer` | `cloudflare-dns01-issuer` |
| `issuers.*.enabled` | selfSigned + localCA | cloudflare |
| `postgresql.primary.persistence` | off | on (10Gi) |
| `redis.auth.enabled` | off | on |
| `frankenphp.hostPath` | source tree | (omitted — baked image) |
| `frankenphp.image` | `dunglas/frankenphp:php8.5-alpine` | `ghcr.io/99linesofcode/frankenphp:<tag>` |

## Usage

Prerequisites: `helm`, `kubectl`, `flux`, `k3s`, `sops`, `age`. The Gateway API
CRDs are **not** bundled — install them before the chart.

```bash
# Fetch and vendor the external chart dependencies
helm repo add traefik https://traefik.github.io/charts
helm repo add jetstack https://charts.jetstack.io
helm repo add bitnami https://charts.bitnami.com/bitnami
helm dependency build

# Local: decrypt the secret directly (no Flux), then install
sops --decrypt secrets/local/secrets.yaml | kubectl apply -f -
helm install laravel-stack . -f values.local.yaml

# Render / lint
helm template . -f values.production.yaml
helm lint .
```

Production is installed by Flux from the fleet repo, not by hand. To publish a
new chart version, tag the repo (`vX.Y.Z`); the `publish` workflow packages and
pushes the chart to `oci://ghcr.io/99linesofcode/charts`.

## Contributing

Please review the [Contribution Guidelines](https://github.com/99linesofcode/.github/blob/main/.github/CONTRIBUTING.md).

## License

This software is open source and licensed under the [MIT license](https://github.com/99linesofcode/kubernetes-php?tab=MIT-1-ov-file).
