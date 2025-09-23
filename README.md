# Botany Flux Example Repository

This repository shows how to structure Flux configuration so that every shoot cluster receives three clearly defined layers of manifests:

1. **Global** – resources applied to every cluster (shared tooling, platform defaults).
2. **Project** – objects shared by a Gardener Project (team namespaces, shared config).
3. **Shoot** – workload- or environment-specific overlays per cluster.

```
ref/botany-flux/
└── clusters/
    ├── global/                  # common resources for all shoots
    ├── projects/
    │   └── kew-dev/             # project-wide resources for the kew-dev team
    └── shoots/
        └── kew-dev/
            ├── ovh-test-1/      # OpenStack shoot overlay
            └── dg31b58jmn/      # GCP shoot overlay
```

## Layer Details

### Global layer (`clusters/global`)
Creates the `platform-tools` namespace and a `cluster-defaults` ConfigMap that every cluster consumes. Any new global policy or addon should be added here so that all shoots receive it automatically.

### Project layer (`clusters/projects/kew-dev`)
Adds the `kew-dev-shared` namespace plus a `team-preferences` ConfigMap. Additional manifests that should be shared by *all* kew-dev shoots (for example, RBAC roles, shared secrets managed by ESO, or monitoring rules) belong in this directory.

### Shoot layer (`clusters/shoots/...`)
Each shoot folder references the global and project layers and appends its own resources:

- `ovh-test-1` provisions a workload namespace, a sample Deployment, and a ConfigMap with OVH-specific settings.
- `dg31b58jmn` creates its workload namespace, a CronJob, and per-cluster configuration keyed for GCP.

Because each shoot `kustomization.yaml` includes:

```yaml
resources:
  - ../../../global
  - ../../../projects/kew-dev
  - resources/...
```

Flux will always reconcile the layers in order before applying the cluster-specific manifests.

## Using with Gardener `shoot-flux`

1. Point the Gardener project ConfigMap (`flux-config` in `garden-<project>` namespace) at this repository. Example values:
   - `repositoryUrl`: `https://github.com/dodwyer/botany-flux.git`
   - `repositoryBranch`: `main`
   - `kustomizationPath`: `clusters/shoots/kew-dev/ovh-test-1`

2. Add the Flux extension to your Shoot and set the `providerConfig` to reference the per-shoot path (as above). The extension bootstraps Flux controllers and applies the layered manifests automatically.

3. To create new shoots:
   - Add a folder under `clusters/shoots/<project>/<shoot-name>` with its own `kustomization.yaml`.
   - Reference the global/project layers exactly as shown.
   - Update the Shoot’s `providerConfig` to point at the new path.

This layout keeps common configuration in one place, lets teams share resources safely, and still gives individual clusters room for bespoke workloads.

### Verifying Each Layer

Every layer ships a ConfigMap with a simple `layer` key so you can confirm what landed on a shoot:

```bash
# Global resources (namespace platform-tools)
kubectl get configmap cluster-defaults -n platform-tools -o yaml | grep layer

# Project resources (namespace kew-dev-shared)
kubectl get configmap team-preferences -n kew-dev-shared -o yaml | grep layer

# Shoot resources (example shoot ovh-test-1)
kubectl get configmap cluster-overrides -n kew-dev-shared -o yaml | grep layer
```

For another shoot, replace the object names with those defined under `clusters/shoots/<project>/<shoot>/resources` (for example `cluster-settings` in the GCP demo).
