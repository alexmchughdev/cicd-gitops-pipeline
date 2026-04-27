# 0003 — Per-app deployment scaffolding can live in either the app repo or a sidecar repo

Date: 2026-04-27
Status: Accepted

## Context

A cluster app needs three things in addition to its source code: a Dockerfile, a CI workflow that builds and pushes the image, and an ArgoCD Application pointing at this manifest repo. There are two sensible places to put these:

1. **Inside the application repo.** Source, Dockerfile, and `.github/workflows/build-and-deploy.yml` live next to each other. Simple to onboard.
2. **In a sidecar `<app>-infra` repo.** The application repo holds source only; the sidecar holds Dockerfile, workflow, and Application manifest. Used when the application repo must remain free of deployment scaffolding (third-party contributors, audited source, coursework deliverables).

Both end up at the same place: `kustomize edit set image` against `apps/<app>/overlays/production/` in this manifest repo, ArgoCD reconciles.

## Decision

Allow either, decided per app at onboarding time. Default to scaffolding-in-app-repo unless one of these is true:

- The app's source repo is read-only or governed externally.
- The contributors to the source repo are not the same as the people who own its deployment.
- The source repo is being submitted as a deliverable that should not include deployment artefacts.

If none of those apply, put the Dockerfile and workflow in the application repo.

## Consequences

The runbook covers both onboarding paths in roughly the same number of steps, so the cost of supporting both is low. The sidecar pattern adds one extra GitHub repo and one extra deploy key (read-only on the source repo) per app, which is acceptable in exchange for a clean source tree.

If a future ApplicationSet auto-discovers `apps/*/overlays/production/`, both patterns continue to work — the manifest repo is the single source of truth ArgoCD reads, regardless of which CI flavour wrote it.
