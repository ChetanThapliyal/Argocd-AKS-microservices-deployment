# Microservice Deployment to AKS using Azure DevOps and ArgoCD

This project demonstrates the deployment of a microservice application to Azure Kubernetes Service (AKS) using Azure DevOps for CI/CD and ArgoCD for GitOps-based continuous deployment. 

For a detailed explanation of the steps involved, please refer to [this blog post](https://blog.chetan-thapliyal.cloud/micro-service-deployment-using-aks-and-argocd).

## Table of Contents

- [Microservice Deployment to AKS using Azure DevOps and ArgoCD](#microservice-deployment-to-aks-using-azure-devops-and-argocd)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Architecture](#architecture)
  - [Prerequisites](#prerequisites)
  - [Setup Instructions](#setup-instructions)
    - [Clone the Repository](#clone-the-repository)
    - [Configure Azure DevOps](#configure-azure-devops)
    - [Setup ArgoCD](#setup-argocd)
    - [Deploy to AKS](#deploy-to-aks)
  - [Usage](#usage)

## Introduction

This repository contains the source code and deployment scripts for a microservice application. The deployment process leverages Azure DevOps for building and deploying the application, and ArgoCD for managing Kubernetes resources in a GitOps manner.

## Architecture

![Architecture Diagram](/architecture/CICD2.png)

The architecture consists of the following components:
- **Microservice Application**: A sample microservice application. (example-voting-app)
- **Azure Kubernetes Service (AKS)**: Managed Kubernetes cluster in Azure.
- **Azure DevOps**: CI/CD platform for building and deploying the application.
- **ArgoCD**: GitOps continuous delivery tool for Kubernetes.

## Prerequisites

Before you begin, ensure you have the following:
- An Azure account with necessary permissions to create resources.
- Azure DevOps organization and project.
- AKS cluster.
- ArgoCD installed on your AKS cluster.
- `kubectl` configured to interact with your AKS cluster.

## Setup Instructions

### Clone the Repository

Clone this repository to your local machine using:

```bash
git clone https://github.com/ChetanThapliyal/Argocd-AKS-microservices-deployment
cd Argocd-AKS-microservices-deployment
```

### Configure Azure DevOps

1. Create a new pipeline in Azure DevOps.
2. Link the repository and configure the pipeline using the `azure-pipelines.yml` file provided in this repository.
3. Set up service connections for Azure and Docker Registry as required by the pipeline.

### Setup ArgoCD

1. Install ArgoCD on your AKS cluster if not already installed. Follow the [official documentation](https://argo-cd.readthedocs.io/en/stable/getting_started/) for installation instructions.
2. Create a new application in ArgoCD that points to this repository.

### Deploy to AKS

1. Once the pipeline is configured and run successfully, it will build the Docker images and push them to the container registry.
2. ArgoCD will automatically sync the Kubernetes manifests from the repository and deploy the application to AKS.

## Usage

After deployment, you can access the microservice application via the external IP address of the AKS service. Use `kubectl get services` to find the external IP.