# KubleOps Manifest

GitOps-driven Kubernetes deployment using Helm and ArgoCD.

## Overview

This repository contains the Helm chart and ArgoCD configuration for deploying a multi-tier app to Amazon EKS.

It belongs to the parent project [KubleOps](https://github.com/firassBenNacib/KubleOps.git). The parent repo provisions infra (EKS, VPC, IAM, NAT). This repo deploys apps with **Helm** and **ArgoCD**.

## Table of Contents

* [Features](#features)
* [Project Structure](#project-structure)
* [Prerequisites](#prerequisites)
* [Installation](#installation)
* [Usage](#usage)
* [Monitoring](#monitoring)
* [License](#license)
* [Author](#author)

## Features

* Helm chart for a three-tier app (frontend, backend, database) with per-tier **ServiceAccounts** and optional **IRSA** via `irsaRoleArn`.
* **Namespaces** created with **Pod Security Admission** labels (baseline/restricted per tier).
* **NetworkPolicies**: default-deny + explicit allow rules (frontend to backend, backend to MySQL, DNS, egress).
* **HPAs** (autoscaling/v2): per-tier CPU (optional memory) with tuned scale up/down behaviors.
* **PDBs**: per-tier budgets to survive voluntary disruptions.
* **Topology spread** constraints across zones and nodes; **nodeSelector** and **tolerations** for tier/capacity placement.
* **Ingress/ALB** with annotations for target-type **ip**, **TLS** (certificate ARN), **SSL policy** (TLS 1.3), **ALB group name**, **load balancer name**, and optional **ExternalDNS**.
* **ECR pulls** via node IAM (default) or registry **imagePullSecrets** (templates provided for frontend/backend).
* **MySQL as StatefulSet** with a **gp3 WaitForFirstConsumer** StorageClass (`gp3-wffc`) and PVC (demo setup).
* Optional **Karpenter**: NodePools and EC2NodeClass definitions under GitOps, split by tier and capacity (on-demand/spot).
* Optional **Argo CD sync waves** per resource type to control apply order.

## Project Structure

```plaintext
kube-proj-manifest/
├── argocd-sync.yaml
└── manifest/
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        ├── namespaces/
        ├── serviceaccount/
        ├── secrets/                 # optional ECR dockerconfigjson for frontend/backend
        ├── ingress/                 # ALB annotations (group name, LB name, TLS, WAF-ready)
        ├── networkpolicies/
        ├── hpas/
        ├── pdbs/
        ├── frontend/                # Deployment, Service, ConfigMap (nginx)
        ├── backend/                 # Deployment, Service, ConfigMap, example Secret refs
        ├── database/                # StorageClass (gp3-wffc), StatefulSet, Services, PVC, ConfigMap
        └── karpenter/               # EC2NodeClass + NodePools (frontend spot/ondemand, backend, database)
````

## Prerequisites

* EKS cluster (from the KubleOps infra repo).
* ArgoCD installed with access to this repo.
* AWS Load Balancer Controller for Ingress/ALB.
* (Optional) **ExternalDNS** to manage Route53 records from Ingress hosts.
* (Optional) **Karpenter** CRDs/controllers if you enable NodePools/EC2NodeClass.
* An ACM certificate in the cluster’s region for TLS on ALB.

## Installation

1. Clone:

```bash
git clone https://github.com/firassBenNacib/KubleOps-manifest.git
cd KubleOps-manifest
```

2. Edit `manifest/values.yaml`:

* Set **image repositories/tags** for `frontend` and `backend`.
* Configure **Ingress**:

  * `ingress.hosts.frontend`, `ingress.hosts.backend`
  * `ingress.certificateArn` (ACM)
  * (Optional) `ingress.groupName`, `ingress.loadBalancerName`, `ingress.sslPolicy`, `ingress.ipAddressType`
  * Set `ingress.externalDNS: true` if ExternalDNS is installed.
* (Optional) **IRSA**: set `*.irsaRoleArn` per tier; otherwise rely on node IAM.
* (Optional) **Registry secrets**: enable under `templates/secrets/` and reference via `imagePullSecrets`.
* **Scheduling**: confirm `nodeSelector`, `tolerations`, and **topologySpread** constraints match your cluster labels.
* **MySQL**: adjust `persistence.size`, `storageClassName`, or point to an `existingClaim`.
* (Optional) **Karpenter**: set `karpenter.clusterName`, `karpenter.zones`, and attach `nodeRoleArn` if needed.
* (Optional) **Argo sync waves**: set `argo.syncWaves.enabled: true` to enforce apply order.

3. Create the ArgoCD `Application`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kubleops
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/firassBenNacib/KubleOps-manifest.git
    path: manifest
    targetRevision: HEAD
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Tip: you can also inline overrides with `source.helm.valuesObject`.

## Usage

### Update and sync

* Change images, env, resources, ingress, or policies in `values.yaml` or `templates/`.
* Commit and push. ArgoCD syncs the changes.

### ECR auth (two options)

* **Node IAM (simple):** let nodes pull from ECR; do not set `imagePullSecrets`.
* **IRSA (scoped):** set `serviceAccountName` and `irsaRoleArn` per app when pods need AWS APIs.

Example:

```yaml
backend:
  serviceAccountName: backend-service-account
  irsaRoleArn: arn:aws:iam::<account-id>:role/eks-ecr-access
  image:
    repository: <account-id>.dkr.ecr.<region>.amazonaws.com/backend
    tag: "TAG"
```

Or use a registry secret:

```yaml
imagePullSecrets:
  - name: dockerhub-secret
```

### Ingress and DNS

Set hosts under `ingress.hosts.*`. If ExternalDNS is installed and `ingress.externalDNS: true`, Route53 records are created automatically from Ingress annotations.

Common ALB annotations already templated:

* `alb.ingress.kubernetes.io/target-type: ip`
* `alb.ingress.kubernetes.io/certificate-arn: ...`
* `alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS13-1-2-2021-06`
* `alb.ingress.kubernetes.io/group.name` / `alb.ingress.kubernetes.io/load-balancer-name`
* `alb.ingress.kubernetes.io/ip-address-type` (default `ipv4`)
* `alb.ingress.kubernetes.io/ssl-redirect: "443"`

### Pod Security Admission (namespaces)

Namespaces are created with PSS labels from `values.yaml`:

```yaml
podSecurity:
  frontend: restricted
  backend: restricted
  database: baseline
```

### NetworkPolicies

Default-deny per namespace + explicit allow flows:

* frontend to backend (HTTP)
* backend to mysql (TCP 3306)
* DNS egress to `kube-system` (CoreDNS) and required egress as needed

### HPAs

CPU-based HPAs for frontend and backend with configurable min/max replicas and scale behaviors; optional memory metric.

### PDBs

Budgets for all tiers (e.g., `frontend.maxUnavailable`, `backend.minAvailable`, `mysql.maxUnavailable`) to avoid full drain during maintenance.

### Scheduling & topology

Per-tier placement:

* `nodeSelector` on `workload-tier: {frontend|backend|database}`
* `tolerations` for tier/capacity
* `topologySpreadConstraints` across zones and nodes

### Storage (MySQL demo)

A `gp3-wffc` StorageClass (WaitForFirstConsumer) is included for locality-friendly PVC binding. Tune `mysql.persistence.*` or use `existingClaim`. For production, consider a managed database.

### Karpenter (optional)

GitOps definitions under `templates/karpenter/`:

* **EC2NodeClass** for AMI/subnets/security groups
* **NodePools** per tier and capacity (frontend-spot, frontend-ondemand, backend, database)
  Enable only if Karpenter is installed in the cluster.

## Monitoring

Monitoring (Prometheus/Grafana) lives in the main [KubleOps](https://github.com/firassBenNacib/KubleOps) repo.

## License

This project is licensed under the [MIT License](./LICENSE).

## Author

Created and maintained by [Firas Ben Nacib](https://github.com/firassBenNacib) - [bennacibfiras@gmail.com](mailto:bennacibfiras@gmail.com)