# gitops

GitOps repository for the duynhlab platform — the single source of truth for
what runs in every environment. Owned by **DevOps/SRE** (CODEOWNERS); developers
never push here directly. Changes arrive as pull requests from
[Backstage self-service templates](https://github.com/duynhlab/backstage) or
from service CI (dev auto-deploy).

## Architecture

```mermaid
flowchart TD
    subgraph sources ["Change sources"]
        CI["checkout-service CI\n(merge to main)"]
        BS["Backstage templates\n(onboard / update / promote)"]
    end

    subgraph repo ["duynhlab/gitops (main)"]
        DevDir["apps/dev/*"]
        ProdDir["apps/prod/*"]
    end

    subgraph devcluster ["Kind cluster: dev"]
        FluxDev["Flux (dev)"]
        DevNS["ns: checkout-dev"]
    end
    subgraph prodcluster ["Kind cluster: prod"]
        FluxProd["Flux (prod)"]
        ProdNS["ns: checkout-prod"]
    end

    CI -->|"direct commit\n(bump image tag)"| DevDir
    BS -->|"pull request\nDevOps review + merge"| ProdDir
    FluxDev -->|"sync clusters/dev (1m)"| repo
    FluxProd -->|"sync clusters/prod (1m)"| repo
    DevDir --> DevNS
    ProdDir --> ProdNS
```

## Promotion flow

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant CI as checkout-service CI
    participant Git as gitops repo
    participant Ops as DevOps/SRE
    participant Flux
    Dev->>CI: merge PR to main
    CI->>CI: test, build, scan, sign → ghcr sha-tag
    CI->>Git: commit bump apps/dev (auto)
    Flux->>Git: reconcile → dev updated
    Dev->>Ops: verified in dev, request promotion (Backstage Update Service, env=prod)
    Ops->>Git: review + merge PR → prod updated
```

## Layout

```
clusters/<env>/           # Flux entrypoint per environment cluster
├── sources.yaml          # OCIRepository: shared mop Helm chart
└── apps.yaml             # Kustomization → ./apps/<env>

apps/
├── base/<service>/       # Environment-invariant HelmRelease (chart, probes, resources)
└── <env>/<service>/      # namespace + kustomization + release-patch.yaml
                          # (replicas, image tag, FULL env var list — source of truth per env)

catalog/<service>.yaml    # Backstage catalog entity (discovered, never applied by Flux)
```

Design notes:

- **Per-env `release-patch.yaml` is the full desired state** of a service in that
  environment: image tag, replicas and the complete env var contract
  (`SERVICE_NAME`, `ENV`, `LOG_LEVEL`, `OTEL_*`, `SHUTDOWN_TIMEOUT`, …).
  Diffs stay small and reviews are self-contained.
- **`apps/base`** holds what never differs per environment (chart ref, probes,
  resources, service ports).
- **`catalog/`** sits outside every Flux sync path, so Backstage entities are
  never applied to the cluster; Backstage's GitHub provider discovers them.
- Each environment is its own Kind cluster running its own Flux
  (flux-operator + FluxInstance syncing `clusters/<env>`). Backstage and its
  database live on a separate mgmt cluster.

## Environments

| Env | Namespace | Update path | ENV | LOG_LEVEL | Replicas |
|-----|-----------|-------------|-----|-----------|----------|
| dev | checkout-dev | CI auto-commit on every main build | development | debug | 1 |
| prod | checkout-prod | Backstage PR + DevOps review | production | warn | 2 |

## Guard rails

- `CODEOWNERS`: DevOps/SRE review required on every PR
- Branch protection on `main`: 1 approving review; admins (DevOps + the CI
  deploy token) can bypass for the dev auto-deploy lane
- Images are immutable (`sha-<short>` / semver) — promotion means moving a tag
  through env files, never rebuilding
