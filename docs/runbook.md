# Runbook

Operational tasks for the platform. Each section is a recipe; not a tutorial.

## Add a new app — Pattern A (centralised manifests)

Use this when the new app's own repository is the right home for its Dockerfile and CI workflow.

1. **Scaffold manifests in this repo:**
   ```
   mkdir -p apps/<app>/{base,overlays/production}
   ```
   Add `deployment.yaml`, `service.yaml`, `ingress.yaml`, `kustomization.yaml` under `base/`. The `kustomization.yaml` should set `namespace: <app>`.
   Under `overlays/production/`, add a `kustomization.yaml` that lists `../../base` as resource and pins an initial image tag.
2. **Add the Dockerfile and CI workflow** in the source repo. The workflow needs to:
   - Build the image, push to `ghcr.io/alexmchughdev/<app>:sha-<short>`.
   - SSH-clone this repo using `MANIFEST_DEPLOY_KEY`.
   - Run `kustomize edit set image ghcr.io/alexmchughdev/<app>=ghcr.io/alexmchughdev/<app>:sha-<tag>` in `apps/<app>/overlays/production/`.
   - Commit and push.

   See `OT-edge-asset-tag-generator` and `alexmchugh-dev` for working examples.
3. **Register the ArgoCD Application:**
   ```
   kubectl apply -f - <<'EOF'
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: <app>
     namespace: argocd
     finalizers: [resources-finalizer.argocd.argoproj.io]
   spec:
     project: default
     source:
       repoURL: https://github.com/alexmchughdev/platform-engineering.git
       targetRevision: main
       path: apps/<app>/overlays/production
     destination:
       server: https://kubernetes.default.svc
       namespace: <app>
     syncPolicy:
       automated: { prune: true, selfHeal: true }
       syncOptions: [CreateNamespace=true, ServerSideApply=true]
   EOF
   ```
4. **Verify** with `kubectl get applications -n argocd` and `kubectl get pods -n <app>`.

## Add a new app — Pattern B (split infra repo)

Use this when the application source repo must stay free of Dockerfile / workflow / argocd.

1. Scaffold manifests in this repo as in Pattern A.
2. **Create a `<app>-infra` repository.** Mirror the structure of `fixmycampus-infra`:
   - `client/Dockerfile`, `client/nginx.conf`, `client/.dockerignore`
   - `server/Dockerfile`, `server/.dockerignore`
   - `argocd/application.yaml` pointing at `apps/<app>/overlays/production` in this repo.
   - `.github/workflows/build-and-deploy.yml` with `workflow_dispatch` trigger and an `app_ref` input.
3. **Generate a read-only deploy key** and register on the app repo:
   ```
   ssh-keygen -t ed25519 -N "" -f /tmp/<app>-builder
   gh repo deploy-key add /tmp/<app>-builder.pub -R alexmchughdev/<app> --title "<app>-infra-builder"
   gh secret set APP_REPO_DEPLOY_KEY -R alexmchughdev/<app>-infra < /tmp/<app>-builder
   gh secret set MANIFEST_DEPLOY_KEY -R alexmchughdev/<app>-infra < ~/.ssh/manifest-deploy-key
   rm /tmp/<app>-builder /tmp/<app>-builder.pub
   ```
4. **Apply the ArgoCD Application** from the public raw URL on the infra repo (assumes the repo is public; otherwise apply from a local checkout).
5. **First build:** `gh workflow run build-and-deploy.yml -R alexmchughdev/<app>-infra -f app_ref=main`.
6. **GHCR package visibility:** new packages default to private. Set both to public:
   ```
   # via the UI: github.com/users/alexmchughdev/packages/container/<package>/settings
   # → Danger Zone → Change visibility → Public
   ```
   Or restore via `gh api -X PATCH /user/packages/container/<package>/visibility -f visibility=public` (requires admin:packages scope).

## Apply a secret out-of-band

Used because the platform does not yet have SealedSecrets. The deployment manifest references the secret by name; the secret itself is not in git.

```
URI='actual-value-here'
kubectl -n <ns> create secret generic <name> --from-literal=KEY="$URI" \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl -n <ns> annotate secret <name> argocd.argoproj.io/compare-options=IgnoreExtraneous --overwrite
kubectl -n <ns> rollout restart deployment/<dep>
```

The annotation tells ArgoCD to treat this resource as not-tracked, so a manual sync does not overwrite it. Without the annotation, ArgoCD's force-sync will replace the data with whatever (or nothing) is in git.

## Debug a failed ArgoCD sync

```
kubectl -n argocd get application <app> -o jsonpath='{.status}' | jq
kubectl -n argocd describe application <app> | tail -40
argocd app get <app>            # if argocd CLI is installed and logged in
argocd app logs <app>            # streaming sync logs
```

Common causes and fixes:

- **`ComparisonError`** on a Secret → forgot the `IgnoreExtraneous` annotation. Apply it.
- **`ImagePullBackOff`** on new ReplicaSet pods → image was bumped in manifest but not actually pushed. Check GHCR. Re-run the workflow if needed.
- **`OutOfSync` with no apparent diff** → run `argocd app diff <app>` to see the actual server-side difference. ServerSideApply field-ownership conflicts can show as drift.
- **`Sync Status: Unknown`** → ArgoCD can't read the source path. Verify `spec.source.path` exists in the manifest repo at the `targetRevision`.

## Verify a deployment after CI completes

```
# 1. CI run finished, exit code 0
gh run list -R alexmchughdev/<repo> --limit 3

# 2. Manifest commit landed
git -C ~/path/to/platform-engineering pull
grep newTag apps/<app>/overlays/production/kustomization.yaml

# 3. ArgoCD reconciled
kubectl -n argocd get application <app>     # SYNC=Synced, HEALTH=Healthy
kubectl -n <ns> get pods                    # latest ReplicaSet running
kubectl -n <ns> rollout status deployment/<dep>

# 4. App responds
curl -fsS https://<hostname>/            # frontend
curl -fsS https://<hostname>/api/health  # API (if present)
```

## Rotate `MANIFEST_DEPLOY_KEY`

The same SSH keypair is used by every CI repo that pushes to this manifest repo. Rotation must update all repos that reference it; otherwise their next build will fail at the SSH clone step.

1. Generate a new keypair:
   ```
   ssh-keygen -t ed25519 -N "" -C "manifest-deploy-key" -f /tmp/manifest-key
   ```
2. Add the new public key as a deploy key (write access) on `alexmchughdev/platform-engineering`:
   ```
   gh repo deploy-key add /tmp/manifest-key.pub -R alexmchughdev/platform-engineering --title "manifest-deploy-key" --allow-write
   ```
3. Update the secret on every repo that uses it:
   ```
   for repo in OT-edge-asset-tag-generator alexmchugh-dev fixmycampus-infra; do
     gh secret set MANIFEST_DEPLOY_KEY -R alexmchughdev/$repo < /tmp/manifest-key
   done
   ```
4. Trigger a build on each to confirm the new key works.
5. Once all CI runs are green, delete the old deploy key from `platform-engineering` (Settings → Deploy keys).
6. `rm /tmp/manifest-key /tmp/manifest-key.pub`.

## Clean up orphaned GHCR packages

Listing packages tied to a user (needs `read:packages` on the gh token):

```
gh auth refresh -s read:packages,delete:packages
gh api '/users/alexmchughdev/packages?package_type=container' --jq '.[].name'
```

Delete a package permanently:

```
gh api -X DELETE /user/packages/container/<package-name>
```

This is irreversible. If any cluster pod is mid-pull or is restarted while the package is missing, that pod will hit `ImagePullBackOff`. Verify nothing in `apps/*/overlays/production/kustomization.yaml` references the package before deleting.

## Cloudflared: add a public hostname

```
# 1. Edit /etc/cloudflared/config.yml; add new rule above the catch-all.
sudo $EDITOR /etc/cloudflared/config.yml

# 2. Create the DNS record on Cloudflare
cloudflared tunnel route dns homelab <hostname>

# 3. Restart and verify
sudo systemctl restart cloudflared
sudo systemctl status cloudflared --no-pager | head -8
```

If `systemctl status` shows `failed`, the new config did not validate. Run `cloudflared tunnel --config /etc/cloudflared/config.yml ingress validate` to see the parse error. Most common cause is the catch-all `service: http_status:404` no longer being the last entry.
