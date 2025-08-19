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

* Helm chart for a three-tier app (frontend, backend, database).
* ArgoCD Gitops (auto-sync, prune, self-heal).
* **NetworkPolicies**: default-deny + explicit allow rules.
* **PDBs**: keep replicas available during voluntary disruptions.
* **HPAs**: scale frontend and backend on CPU (optional memory).
* Namespace **Pod Security Admission** labels (baseline/restricted).
* Optional **ExternalDNS** for Route53 from Ingress hosts.
* AWS Load Balancer Controller annotations (TLS, access logs, WAF).
* Optional **Karpenter** NodePool/EC2NodeClass under GitOps.

## Project Structure

```plaintext
kube-proj-manifest/
├── argocd-sync.yaml
└── manifest/
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        ├── namespaces/  serviceaccount/  ingress/
        ├── networkpolicies/  hpas/  pdbs/
        ├── backend/  frontend/  database/
        └── karpenter/   # optional
```

## Prerequisites

* EKS cluster (from the KubleOps infra repo).
* ArgoCD installed with access to this repo.
* AWS Load Balancer Controller if you use Ingress/ALB.
* ExternalDNS if you want Route53 records from Ingress hosts.

## Installation

1. Clone:

```bash
git clone https://github.com/firassBenNacib/KubleOps-manifest.git
cd KubleOps-manifest
```

2. Edit `manifest/values.yaml`:

* Set image repositories/tags.
* Configure Ingress hosts and the ACM certificate ARN for TLS on ALB.
* (Optional) enable ExternalDNS annotations if you want Route53 records.

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

* **Node IAM (simple):** let nodes pull from ECR with their instance role; don’t set `imagePullSecrets`.
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

Set hosts under `ingress.hosts.*` in `values.yaml`. If ExternalDNS is installed, it reads hosts and annotations to manage Route53. You can also set `external-dns.alpha.kubernetes.io/hostname`.

### ALB settings

Common Ingress annotations:

* `alb.ingress.kubernetes.io/target-type: ip`
* `alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...`
* `alb.ingress.kubernetes.io/load-balancer-attributes` for access logs
* `alb.ingress.kubernetes.io/wafv2-acl-arn` to attach a WAF (optional)

### Pod Security Admission (namespaces)

Label namespaces to enforce Pod Security Standards:

```yaml
metadata:
  labels:
    pod-security.kubernetes.io/enforce: "restricted"
    pod-security.kubernetes.io/audit: "restricted"
    pod-security.kubernetes.io/warn: "restricted"
```

### NetworkPolicies

* Each namespace starts with default-deny.
* Allow only what you need (frontend → backend, backend → db, DNS, HTTPS).

### HPAs

* HPAs for frontend and backend (autoscaling/v2).
* CPU by default; memory is optional; behaviors are configurable in values.

### PDBs

* PDBs for all tiers.
* Set `minAvailable` to avoid full drain during maintenance.

### Karpenter (optional)

* Manage capacity with GitOps under `templates/karpenter/`.
* Define **NodePools** (constraints, limits, consolidation) and **EC2NodeClass** (AWS settings).

## Monitoring

Monitoring (Prometheus/Grafana) lives in the main [KubleOps](https://github.com/firassBenNacib/KubleOps) repo.

## License

This project is licensed under the [MIT License](./LICENSE).

## Author

Created and maintained by [Firas Ben Nacib](https://github.com/firassBenNacib) - [bennacibfiras@gmail.com](mailto:bennacibfiras@gmail.com)
