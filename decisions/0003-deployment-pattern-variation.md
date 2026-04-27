# 0003 — Tolerate two deployment patterns until a third app justifies standardising

Date: 2026-04-27
Status: Accepted (with technical debt acknowledged)

## Context

Three apps live on the cluster. They were onboarded over a few months as the platform evolved, and they end up using two different deployment patterns:

**Pattern A — centralised manifests**
- App: `OT-edge-asset-tag-generator`, `alexmchugh-dev`.
- The application repo holds source code, Dockerfile, and the build workflow.
- The workflow builds an image, pushes to GHCR, then SSH-clones this manifest repo and runs `kustomize edit set image` against the app's overlay.
- ArgoCD watches this manifest repo and reconciles.

**Pattern B — split infra repo**
- App: `fixmycampus`.
- The application repo holds only source code. No Dockerfile. No workflow.
- A second repo (`fixmycampus-infra`) holds the Dockerfiles, the build workflow, and the ArgoCD Application manifest.
- The infra workflow checks out app source via a read-only deploy key, builds the image, and updates the same central manifest repo.
- ArgoCD watches this manifest repo (same as Pattern A).

Pattern B was introduced because `fixmycampus` was a coursework deliverable that needed to keep its source repo free of deployment scaffolding.

## Decision

Document both patterns honestly and defer standardisation. This ADR is the record that the variation was a conscious response to a constraint, not an oversight.

The current platform-engineering README, architecture.md, and runbook describe both patterns side by side. Apps in the manifest repo continue to share a single layout (`apps/<app>/{base,overlays/production}/`) regardless of which pattern produced them.

## Consequences

**Positive**

- Fast onboarding for apps with a separation requirement (coursework, audited source, third-party contributors).
- Pattern A still works for any app that happily owns its own deployment scaffolding.
- A reader can compare the two and see the trade-offs without ambiguity.

**Negative**

- Two onboarding paths means two runbook entries, two failure modes, two sets of secrets per app under Pattern B (`APP_REPO_DEPLOY_KEY` is extra).
- A future engineer (or me, in six months) will have to read the ADR to know which pattern to pick for a new app.
- Drift risk: if a Pattern A app outgrows the constraint that put it in Pattern A, migrating to Pattern B is a deliberate refactor.

## Standardisation criteria

Standardise on Pattern B (split infra repo) when **any** of the following becomes true:

1. A third or fourth app needs Pattern B for the same source-purity reason as fixmycampus.
2. A reusable workflow can express both patterns with a single template.
3. An ApplicationSet is introduced (per ADR 0002) that requires a single shape for all apps.

Until then, the cost of standardising preemptively exceeds the cost of carrying two patterns.
