# rideshare-argo

GitOps repository for the RideShare platform. Argo CD reconciles the manifests
in this repo into the EKS cluster. **The cluster is never modified directly.**

## Layout

```
apps/
  <service>/
    base/                    # plain k8s manifests (no env-specific values)
    overlays/
      dev/                   # patches: namespace, image tag, replicas, config, secrets
      staging/
argocd/
  applications/              # Argo CD Application CRs (one per service-env)
platform/
  namespaces/                # rideshare-dev, rideshare-staging
```

## Services in scope

| Service                       | Port | Source repo                                                       |
| ----------------------------- | ---- | ----------------------------------------------------------------- |
| `rideshare-frontend`          | 3000 | https://github.com/Ekene-Chris/rideshare-frontend                 |
| `rideshare-matching-service`  | 3004 | https://github.com/Ekene-Chris/rideshare-matching-service         |

Other services follow the same pattern: `https://github.com/Ekene-Chris/rideshare-<service>`.

## What lives where

- **App code, Dockerfile, CI build** → in each app repo
- **Container image** → AWS ECR, tagged with the git SHA
- **Desired k8s state** → this repo
- **Cluster reconciliation** → Argo CD (already installed in `argocd-chris` namespace)

## Sync policies

| Application                          | Sync           | Self-heal | Prune |
| ------------------------------------ | -------------- | --------- | ----- |
| `platform-namespaces`                | automated      | yes       | no    |
| `rideshare-frontend-dev`             | automated      | yes       | yes   |
| `rideshare-matching-service-dev`     | automated      | yes       | yes   |
| `rideshare-frontend-staging`         | **manual**     | n/a       | n/a   |
| `rideshare-matching-service-staging` | **manual**     | n/a       | n/a   |

Dev reconciles automatically — push to `main` and the cluster catches up.
Staging requires an explicit sync (CLI or UI), which is the human gate before
promotion lands.

## Environments

| Env     | Namespace               | ECR registry path                                                          |
| ------- | ----------------------- | -------------------------------------------------------------------------- |
| dev     | `rideshare-dev-chris`     | `596484147832.dkr.ecr.eu-north-1.amazonaws.com/chris/<service>`            |
| staging | `rideshare-staging-chris` | `596484147832.dkr.ecr.eu-north-1.amazonaws.com/chris/<service>`            |

The `-chris` suffix scopes these namespaces to a single engineer's workspace on
a shared cluster. Other engineers use the same repo with their own suffix.

## First-time setup

Two things still need real values before the first sync:

1. **Secret store / AWS secret names** — replace `aws-secretsmanager-{dev,staging}`
   and `rideshare/{dev,staging}/<service>` placeholders in
   `apps/<svc>/overlays/*/external-secret-patch.yaml` with the real
   ClusterSecretStore name and AWS Secrets Manager key paths.
2. **Initial image tags** — replace `dev-placeholder` / `staging-placeholder` in
   each overlay's `kustomization.yaml` with a real ECR tag (a git SHA from the
   app repo's CI).

Then push this repo to GitHub and apply the Application CRs once:

```bash
kubectl apply -f argocd/applications/
```

`platform-namespaces` should reconcile first — it creates `rideshare-dev-chris`
and `rideshare-staging-chris` so the service Apps have somewhere to land.

After that, all changes flow through PRs to this repo.

## Promoting a release

1. App repo CI builds and pushes `<ECR>/rideshare-frontend:<sha>`.
2. Open a PR here that bumps the dev overlay's image tag:

   ```bash
   cd apps/rideshare-frontend/overlays/dev
   kustomize edit set image \
     rideshare-frontend=596484147832.dkr.ecr.eu-north-1.amazonaws.com/chris/rideshare-frontend:<sha>
   ```

3. Merge → Argo CD auto-syncs dev within ~3 minutes (or click *Refresh*).
4. Verify in dev. To promote, open a second PR bumping the same tag in
   `overlays/staging/kustomization.yaml`.
5. Merge → click *Sync* on `rideshare-frontend-staging` in the Argo CD UI
   (or `argocd app sync rideshare-frontend-staging`).

## Rolling back

Rollback is a `git revert` on the PR that bumped the tag. Argo CD reconciles
back to the previous tag automatically (dev) or on next manual sync (staging).
The audit trail is the git history of this repo.

```bash
git revert <bad-promotion-sha>
git push
```

## Verifying locally

Render any overlay to plain YAML before committing:

```bash
kustomize build apps/rideshare-frontend/overlays/dev
kustomize build apps/rideshare-frontend/overlays/staging
kustomize build apps/rideshare-matching-service/overlays/dev
kustomize build apps/rideshare-matching-service/overlays/staging
kustomize build platform/namespaces
```

## App-repo CI changes (out of scope here, do later)

In each app repo's `.github/workflows/deploy.yml`:

- **Keep** the `build` job (build → push to ECR with `:${{ github.sha }}`).
- **Delete** the `deploy` job entirely — that's the kubectl-apply-from-CI
  anti-pattern this repo replaces.
- **Add** a step that opens a PR against this repo bumping
  `apps/<svc>/overlays/dev/kustomization.yaml`'s `newTag`.

Until that automation is in place, the bump is a manual `kustomize edit`
followed by a regular PR.
