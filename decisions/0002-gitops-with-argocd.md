# 0002 — GitOps with ArgoCD watching a central manifest repo

Date: 2026-04-27
Status: Accepted

## Context

Three applications need to land on the cluster. The options were:

1. **Imperative**: SSH onto a node, `kubectl apply` directly, possibly with a small wrapper script.
2. **Push-based CD from CI**: GitHub Actions runs `kubectl apply` against the cluster using a kubeconfig stored as a secret.
3. **Pull-based GitOps**: a controller in the cluster watches a git repo and reconciles desired state continuously.

Imperative drifts the moment anyone touches the cluster manually. Push-based CD requires cluster credentials in CI, expands the attack surface, and gives no continuous reconciliation — drift caused by a one-off `kubectl edit` is invisible until someone spots it.

## Decision

Pull-based GitOps with **ArgoCD**. The cluster runs an ArgoCD installation (UI registered Applications, no ApplicationSet yet). Each Application points at a Kustomize overlay path in this repository (`apps/<app>/overlays/production`). ArgoCD reconciles every three minutes plus on every push (via webhook when configured).

Image promotions happen by tag bump: CI builds the image, pushes to GHCR, then commits a `kustomize edit set image` change to the overlay. ArgoCD picks up the commit and applies the new image.

Sync policy is automated with `prune: true` and `selfHeal: true`. Drift is corrected; resources removed from git are pruned.

## Consequences

**Positive**

- Cluster credentials never leave the cluster. CI pushes to git; the cluster pulls from git.
- Continuous reconciliation closes the drift loop. `kubectl edit` in anger is reverted within minutes.
- Manifest history is in git, with the same review and rollback affordances as application code.
- Kustomize overlays keep base manifests reusable; production-specific config (image tags, env injection) lives in `overlays/production/`.

**Negative**

- The CI loop is two-stage: image build + manifest commit. A failure between the two leaves the cluster on the previous image without obvious signal. Mitigation is the workflow's `update-manifest` job committing only after a successful push.
- Secrets are awkward. The chosen pattern (see 0003 and the runbook) is to apply secrets out-of-band with `kubectl` and annotate them so ArgoCD treats them as extraneous. Long-term roadmap is SealedSecrets or sops.
- Applications are registered manually via the UI. An ApplicationSet that auto-discovers `apps/*/overlays/production/` is the standard fix, deferred until adding an app becomes painful.

## Roadmap link

ApplicationSet auto-discovery and SealedSecrets/sops are listed in the README roadmap.
