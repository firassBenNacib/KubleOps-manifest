# KubleOps Manifest

GitOps-driven Kubernetes deployment using Helm and ArgoCD.

## Overview

This repository contains the Helm charts and ArgoCD configurations for deploying a production-grade, multi-tier application to Amazon EKS.

This repository is part of the [KubleOps](https://github.com/firassBenNacib/KubleOps.git) project. While the parent repo handles the infrastructure setup (EKS, VPC, IAM, Bastion, etc.), this repo is dedicated to deploying applications to the Kubernetes cluster using **Helm** and **ArgoCD** with a GitOps approach.

## Table of Contents

- [Features](#features)
- [Project Structure](#project-structure)
- [Installation](#installation)
- [Usage](#usage)
- [Monitoring](#monitoring)
- [License](#license)
- [Author](#author)

## Features

- Helm-based chart structure for Kubernetes manifests.
- ArgoCD sync automation via GitOps.
- Support for production-ready multi-tier application deployment.
- Works with AWS ECR using IAM Roles for Service Accounts (`irsaRoleArn`).

## Project Structure

```plaintext
kube-proj-manifest/
├── .gitignore
├── argocd-sync.yaml         
└── manifest/
    ├── Chart.yaml           
    ├── values.yaml           
    ├── templates/           
    ├── charts/              
    └── .helmignore
````

## Installation

1. Clone the repository:

   ```bash
   git clone https://github.com/firassBenNacib/KubleOps-manifest.git
   cd KubleOps-manifest
   ```

2. Review or edit `manifest/values.yaml`:

   * Replace `irsaRoleArn` with your AWS IAM Role ARN if using ECR.
   * Or set up a Kubernetes `imagePullSecret` for Docker Hub if using public/private Docker images.

3. Configure your ArgoCD application to point to the Helm chart path:

   ```yaml
   spec:
     source:
       repoURL: https://github.com/firassBenNacib/KubleOps-manifest.git
       path: manifest
       targetRevision: HEAD
       helm:
         valueFiles:
           - values.yaml
   ```

## Usage

* Update container images, configs, or Kubernetes specs in `values.yaml` or `templates/`.
* Commit and push your changes.
* ArgoCD automatically syncs the changes to your EKS cluster.

To customize IAM access for ECR image pulls, use the `serviceAccountName` and `irsaRoleArn` settings like this:

```yaml
backend:
  serviceAccountName: backend-service-account
  irsaRoleArn: arn:aws:iam::<account-id>:role/eks-ecr-access
```

Or use Docker Hub and add imagePullSecrets:

```yaml
imagePullSecrets:
  - name: dockerhub-secret
```

## Monitoring

Observability is handled in the main [KubleOps](https://github.com/firassBenNacib/KubleOps) repo using Prometheus and Grafana, integrated into the infrastructure setup.

## License

This project is licensed under the [MIT License](./LICENSE).

## Author

Created and maintained by [Firas Ben Nacib](https://github.com/firassBenNacib) - [bennacibfiras@gmail.com](mailto:bennacibfiras@gmail.com)

