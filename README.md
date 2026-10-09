# lz-infrastructure

Landing-zone infrastructure claims, reconciled by Crossplane on the mgmt cluster.
Synced by the `infrastructure` ApplicationSet in **lz-argocd-config** — a matrix of
`{environments/*, shared} x {org, network, projects, clusters}`, one ArgoCD app per
combination.

## Bootstrap (once, by hand)

The Crossplane provider bills all GCP API calls to the seed project
(`lz-platform-seed`). Before anything can reconcile, enable the two APIs the
provider itself needs — Crossplane can't do it itself (chicken-egg: enabling
APIs requires the Service Usage API):

```sh
gcloud services enable serviceusage.googleapis.com cloudresourcemanager.googleapis.com \
  --project lz-platform-seed
```

`shared/org/seed-apis.yaml` then keeps them enabled declaratively (adopts the
manual state; enabling an enabled API is a no-op).

## Layer semantics

| Layer | Wave (intent) | Contents |
|---|---|---|
| `org` | 0 | Seed APIs, org policies |
| `network` | 5 | Shared VPC, subnets, NAT, PSC, DNS hub |
| `projects` | 10 | Resource hierarchy claims (folders + GCP projects) |
| `clusters` | 15 | workload GKE clusters |

**Waves document intent only.** Generated apps sync independently — cross-layer
ordering is convergence by retry (an XR applied before its XRD/provider exists fails
and retries until it succeeds). Health checks make status truthful meanwhile.

## Hard conventions

- **Full grid**: every env dir must contain every layer dir (empty + `.gitkeep`
  counts). A missing path fails that app.
- **New environment** = `cp -r environments/dev environments/<env>` and adjust.
- **Namespaces**: namespaced XRs/MRs (Crossplane v2) land in `lz-<env>` /
  `lz-shared` via the app destination — never set `metadata.namespace` in files here.
- **Label contract**: everything carries the `platform.lzaas/*` contract labels
  (`owner`, `env`, `cost-center`, `data-classification`). XRs get them from spec
  fields via the composition; hand-written MRs set them statically. Details:
  `lz-crossplane-core/docs/composition-standards.md`.
- **Composition rollouts**: claims pin a channel via
  `spec.crossplane.compositionRevisionSelector` + `compositionUpdatePolicy: Manual`
  — no silent fleet-wide updates.
- Project IDs are globally unique in GCP. If a sync fails with "already exists",
  pick another name — do NOT reuse someone else's.

## Current contents

- `shared/org/seed-apis.yaml` — provider-critical APIs on the seed project.
- `shared/org/org-policies.yaml` — org policy baseline (XOrgPolicyBundle claim).
- `shared/projects/resource-hierarchy.yaml` — the LZ folder/project skeleton
  (XResourceHierarchy claim: folders + projects + per-project APIs).
