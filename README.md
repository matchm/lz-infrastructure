# lz-infrastructure

Landing-zone infrastructure claims, reconciled by Crossplane on the mgmt cluster.
Synced by the `infrastructure` ApplicationSet in **lz-argocd-config** — a matrix of
`{environments/*, shared} x {org, network, projects, clusters}`, one ArgoCD app per
combination.

## Layer semantics

| Layer | Wave (intent) | Contents |
|---|---|---|
| `org` | 0 | Folder hierarchy, org policies |
| `network` | 5 | Shared VPC, subnets, NAT, PSC, DNS hub |
| `projects` | 10 | GCP projects (XProject XRs) |
| `clusters` | 15 | workload GKE clusters |

**Waves document intent only.** Generated apps sync independently — cross-layer
ordering is convergence by retry (an XR applied before its XRD/provider exists fails
and retries until it succeeds). Health checks make status truthful meanwhile.

## Hard conventions

- **Full grid**: every env dir must contain every layer dir (empty + `.gitkeep`
  counts). A missing path fails that app.
- **New environment** = `cp -r environments/dev environments/<env>` and adjust.
- **Namespaces**: namespaced XRs (Crossplane v2) land in `lz-<env>` /
  `lz-shared` via the app destination — never set `metadata.namespace` in files here.
- **Cluster-scoped MRs** (folders today) carry labels:
  `app.kubernetes.io/{name: crossplane, component: infra, part-of: platform}`.
- XRs reference folders **by Folder resource name** (`folderName`), never by numeric
  ID — the composition resolves it via `folderIdRef`.
- Project IDs are globally unique in GCP. If a sync fails with "already exists",
  pick another `projectId` — do NOT reuse someone else's.

## Current contents

- `shared/org/folders.yaml` — the LZ folder hierarchy (core / security / platform /
  workloads{prod,non-prod}), owned by Crossplane. Terraform (Day-0) stays seed-only.
- `environments/dev/projects/sandbox.yaml` — first XProject, the end-to-end proof.
