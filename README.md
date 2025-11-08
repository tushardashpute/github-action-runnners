***

# GitHub Actions Self-Hosted Runner on AWS EKS

This guide walks you through setting up a self-hosted GitHub Actions runner on your AWS EKS cluster using the [Actions Runner Controller (ARC)].[1]

## Prerequisites

- AWS EKS cluster configured and `kubectl`/`helm` installed.[1]
- GitHub repository and Personal Access Token (PAT) with repo access.[1]

## Steps

### 1. Install cert-manager on EKS

```shell
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.2/cert-manager.yaml
kubectl get pods --namespace cert-manager
```
Verify that cert-manager pods are running.[1]

### 2. Create GitHub Personal Access Token & Kubernetes Secret

- Generate a PAT: GitHub → Settings → Developer settings → Personal access tokens → Generate new token (select `repo` scope).[1]
- Create namespace and secret:

```shell
kubectl create ns actions-runner-system
kubectl create secret generic controller-manager -n actions-runner-system --from-literal=github_token=<YOUR_GITHUB_PAT>
```

### 3. Install Actions Runner Controller (ARC)

```shell
helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller
helm repo update
helm upgrade --install --namespace actions-runner-system \
  --create-namespace --wait actions-runner-controller \
  actions-runner-controller/actions-runner-controller \
  --set syncPeriod=1m
kubectl get all -n actions-runner-system
```
Confirm ARC resources are present.[1]

### 4. Deploy GitHub Actions Runner

Create `runner.yml` with the following configuration:

```yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: k8s-action-runner
  namespace: actions-runner-system
spec:
  replicas: 1
  template:
    spec:
      repository: <your-org>/<your-repo>
      labels:
        - "eks_runner"
```

Apply deployment:

```shell
kubectl create -f runner.yml
kubectl get pod -n actions-runner-system | grep k8s-action-runner
```
Check your runner appears under GitHub → Settings → Actions → Runners.[1]

### 5. Test with a GitHub Actions Workflow

Create `.github/workflows/test.yml`:

```yaml
name: Testing

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: eks_runner
    container:
      image: ubuntu:latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v2
        with:
          ref: main
      - name: Echo Message
        run: echo "Hello World"
```
Push to `main` ⇒ workflow triggers on your EKS runner.[1]

***
- Full guide: [devopscube.com/github-actions-runner-aws-eks/][1]

[1](https://devopscube.com/github-actions-runner-aws-eks/)
