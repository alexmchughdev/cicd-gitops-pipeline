# ArgoCD

GitOps controller. Watches this repo and reconciles `apps/<app>/overlays/production` paths into the cluster.

## What is currently running

```
$ kubectl get pods -n argocd
argocd-application-controller-0          1/1 Running
argocd-applicationset-controller-...     0/1 CrashLoopBackOff   <-- see Known issues
argocd-dex-server-...                    1/1 Running
argocd-notifications-controller-...      1/1 Running
argocd-redis-...                         1/1 Running
argocd-repo-server-...                   1/1 Running
argocd-server-...                        1/1 Running
```

Three Applications registered:

```
$ kubectl get applications -n argocd
NAME                SYNC STATUS   HEALTH STATUS
alexmchugh-dev      Synced        Healthy
dfx-tag-generator   Unknown       Healthy   <-- see Known issues
fixmycampus         Synced        Healthy
```

## Install method

Installed once via the upstream `install.yaml` manifest into the `argocd` namespace, not via Helm. No manifests for ArgoCD itself live in this repo — the install is treated as a one-shot cluster bootstrap step.

To re-bootstrap on a fresh cluster:

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Then register Applications individually (see `decisions/0002` and the runbook).

## Application registration

Applications are registered manually via the ArgoCD UI or by `kubectl apply`-ing the Application YAML stored alongside the app:

- `apps/fixmycampus/argocd-application.yaml` (publicly accessible, applied via raw URL).
- `alexmchughdev/alexmchugh-dev` repo, `argocd/application.yaml`.
- `dfx-tag-generator` was registered in the UI, no committed YAML.

## Sync policy

All three Applications use:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
    - CreateNamespace=true
    - ServerSideApply=true
```

`fixmycampus` additionally uses `RespectIgnoreDifferences=true`. Drift is corrected within minutes.

## Known issues

- **`argocd-applicationset-controller` is in CrashLoopBackOff.** Has been since install. Not a blocker because no ApplicationSet resources are in use; Applications are managed manually. Fix or remove the controller is on the roadmap.
- **`dfx-tag-generator` Application reports `Unknown` sync status.** Its source path will be migrated from `apps/dfx-tag-generator/` to `apps/ot-edge-asset-tag-generator/` (this repo's rename); the migration is captured in the runbook.

## Access

ArgoCD UI is not exposed publicly. Reach it via port-forward:

```
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

Then browse `https://localhost:8080`.
