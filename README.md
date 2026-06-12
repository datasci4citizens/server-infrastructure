# Server Infrastructure

K8s manifests and ArgoCD applications for student projects on the UNICAMP IC cluster (`ak8s.ic.unicamp.br`).

## Structure

```
.
├── apps/                         # Deployed applications
│   └── mih-server/
│       ├── base/                 # Kustomize base — shared manifests
│       └── overlays/
│           └── prod/             # Production overlay (namespace, secrets, image tag)
├── argocd/                       # ArgoCD Application definitions (one per app)
└── templates/
    ├── app/                      # Template for new apps (copy this)
    │   ├── base/
    │   └── overlays/prod/
    ├── argocd/                   # ArgoCD Application template
    │   └── argocd-app.yaml
    └── workflows/                # GitHub Actions templates for app repos
```

## Adding a new app

1. **Copy template**:

   ```bash
   cp -r templates/app apps/<your-app>
   ```

2. **Replace placeholders** in all files under `apps/<your-app>/`:
   - `<APP_NAME>` — your app name (e.g. `my-api`)
   - `<NAMESPACE>` — K8s namespace (e.g. `my-api`)
   - `<GH_USER_OR_ORG>` — GitHub user/org owning the image
   - `<APP_PORT>` — container port (e.g. `8000`)

3. **Configure your app**:
   - Add env vars to `base/configmap.yaml`
   - If app needs DB, copy `apps/mih-server/base/postgres.yaml` as reference and uncomment in `base/kustomization.yaml`

4. **Create ArgoCD Application**:

   ```bash
   cp templates/argocd/argocd-app.yaml argocd/<your-app>-app.yaml
   ```

   Replace placeholders in the new file.

5. **Commit + push** infra repo.

6. **Apply secret manually on cluster** (one-time, secret is NOT in git):

   ```bash
   cp templates/app/overlays/prod/secret-example.yaml secret.yaml   # local only
   # fill secret.yaml with real values
   kubectl create namespace <NAMESPACE>
   kubectl apply -f secret.yaml -n <NAMESPACE>
   ```

7. **Apply ArgoCD app on cluster**:
   ```bash
   kubectl apply -f argocd/<your-app>-app.yaml
   ```

## Deploy flow

```
app repo                    infra repo (this)            cluster
─────────                   ──────────────────           ────────
push to main
   │
   ▼
build image (CI)
   │
   ▼
push to ghcr.io
   │
   ▼
trigger workflow ───────► bump image tag in
                          overlays/prod/
                          kustomization.yaml
                             │
                             ▼
                          commit + push
                             │
                             ▼
                          ArgoCD detects ───────────► sync to cluster
```

## App repo setup

In each app repo, copy `templates/workflows/deploy-k8s.yml` to `.github/workflows/`.
One job: builds the Docker image, pushes to GHCR, then bumps the tag in this repo
(ArgoCD syncs from there). Frontends uncomment the `build-args` block for their
build-time `VITE_*` vars.

Required in app repo settings (Settings → Secrets and variables → Actions):

- Secret `INFRA_APP_ID` — GitHub App ID (see "GitHub App setup" below)
- Secret `INFRA_APP_PRIVATE_KEY` — full `.pem` file contents
- Variable `APP_NAME` — folder name under `apps/` (e.g. `mih-server`)

## GitHub App setup (one-time, org admin)

The infra repo is updated by a GitHub App so commits don't rely on any individual user's PAT.

1. `github.com/organizations/datasci4citizens/settings/apps → New GitHub App`
2. Name: `infra-deployer`
3. Webhook: disable
4. Repository permissions → **Contents: Read and write**
5. Where installable: "Only on this account"
6. Create → note **App ID** + generate **Private Key** (downloads `.pem`)
7. Install App on `server-infrastructure` repo
8. Share App ID and `.pem` contents with team members; each app repo stores them as secrets `INFRA_APP_ID` and `INFRA_APP_PRIVATE_KEY`

## Secrets management

Secrets are NOT stored in git. Workflow:

- `secret-example.yaml` (committed) — template showing required keys
- `secret.yaml` (gitignored) — real values, applied manually to cluster once
- ArgoCD manages everything else; references the existing `<APP_NAME>-secret` in namespace

For production-grade secret management in git, look into [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) or [SOPS](https://github.com/getsops/sops).

## Apply manually (without ArgoCD)

```bash
kubectl apply -k apps/<your-app>/overlays/prod
```
