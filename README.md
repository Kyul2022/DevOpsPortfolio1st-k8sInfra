# DevOpsPortfolio1st-k8sInfra

**GitOps** repository holding the Kubernetes manifests used by **ArgoCD** to deploy and continuously reconcile the backend application + database on a **k3s** cluster.

This repo describes the **desired state of the infrastructure**. It pairs with [`k3s2on10-back`](https://github.com/Kyul2022/k3s2on10-back), which holds the API source code and its CI pipeline (building and pushing the Docker image consumed here).

## How it works

An ArgoCD `Application` resource (`application.yaml`) points to this repo's `dev/` folder. ArgoCD continuously watches that folder and automatically applies any change to the cluster:

```yaml
syncPolicy:
  syncOptions:
    - CreateNamespace=true
  automated:
    selfHeal: true   # reverts any manual drift on the cluster
    prune: true       # removes resources that were deleted from the repo
```

The application's target namespace is `myapp`, while the ArgoCD resources themselves live in the `argocd` namespace.

## Structure

```
application.yaml     # ArgoCD "Application" resource (source = this repo, path=dev)
dev/
├── storageClass.yaml # "local-storage" StorageClass (no-provisioner, default)
├── volume.yaml         # local PersistentVolume, bound to the "slave" node
├── pvc.yaml              # PVC for MySQL data
├── secrets.yaml           # DB credentials (Opaque Secret)
├── config.yaml             # ConfigMap (DB host/port/name)
├── mysql.yaml                # MySQL Deployment + Service
└── backend.yaml                # Spring Boot API Deployment + Service
```

## Deployed components

- **MySQL**: a pod backed by a persistent volume (`mysql-pvc`), exposed as `ClusterIP` on port 3306.
- **Backend**: the Spring Boot API (image `kyul1234/devops-portfolio:<sha>`, tagged by commit for traceable, immutable deployments), exposed as `ClusterIP` on port 8082 → 8080. Its environment variables (`DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`) are injected from the `ConfigMap` and `Secret`.
- **Storage**: a local `PersistentVolume` (path `/mnt/disks/ssd1`) restricted to the `slave` node via `nodeAffinity`, consumed by MySQL through a `PersistentVolumeClaim` and the `local-storage` `StorageClass` (`WaitForFirstConsumer` binding mode).

## Prerequisites

- A **k3s** cluster with at least one node named `slave` (referenced by the `PersistentVolume`'s node affinity).
- **ArgoCD** installed on the cluster.

## Deployment

```bash
kubectl apply -f application.yaml
```

Once the `Application` resource is created, ArgoCD takes over: it clones this repo, applies all manifests in `dev/`, creates the `myapp` namespace if needed, and keeps the cluster state in sync with this repo on every commit (self-heal + prune).

## Relation to the backend repo

The image consumed by `dev/backend.yaml` is produced by the CI pipeline in [`k3s2on10-back`](https://github.com/Kyul2022/k3s2on10-back). This repo only declares *which* image should run and *how* — the core idea of GitOps: separating CI (building/pushing the image) from CD (ArgoCD reconciling the state declared here).
