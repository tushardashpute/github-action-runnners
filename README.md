***

# GitHub Actions Self-Hosted Runner on AWS EKS

This guide walks you through setting up a self-hosted GitHub Actions runner on your AWS EKS cluster using the Actions Runner Controller (ARC). You’ll find theoretical context, a practical installation guide, and key insights on ARC and self-hosted runner architecture.

***

## Background: GitHub Actions and Runners

**GitHub Actions** is a powerful CI/CD platform that automates build, test, and deployment workflows using a “runner”—a machine that executes the jobs defined in workflow files.

- **GitHub-hosted runners** are managed by GitHub, ephemeral, and require no maintenance, but may be limited in customization and connectivity.[4][7]
- **Self-hosted runners** are infrastructure you or your team provide (VM, bare metal, cloud, or Kubernetes), giving full control over software, resources, and network access. They are ideal for scenarios requiring special tools, increased parallelism, persistent caches, or access to private infrastructure.[2][5][1]

When you use self-hosted runners, your hardware or cloud resources run jobs via the GitHub Actions runner application, which connects outbound to GitHub. This means control over scaling, security, and the runtime environment is up to you.[9][1]

***

## What is Actions Runner Controller (ARC)?

**Actions Runner Controller (ARC)** is an open-source Kubernetes operator that manages GitHub Actions runners as native Kubernetes resources. ARC automates:

- **Provisioning**: Automatically spawns runners as Kubernetes pods on-demand.
- **Scaling**: Adjusts runner pods based on workflow job load, supporting high efficiency and cost savings.[2]
- **Maintenance & Resilience**: Integrates with cluster security, monitoring, and autoscaling for robust enterprise environments.
- **Native Integration**: ARC-managed runners have all the advantages of a Kubernetes-native workflow—declarative resource management, security policies, scaling, and seamless integration with other workloads.[5][2]

This makes ARC perfect for DevOps teams wanting to maximize performance by bringing CI/CD right into their AWS EKS or other Kubernetes clusters.

***

## Step-by-Step Guide

### Prerequisites

- AWS EKS cluster and `kubectl`/`helm` configured.[10]
- GitHub repo and personal access token (PAT) with `repo` scope.[10]

### 1. Install cert-manager on Your EKS Cluster

```shell
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.2/cert-manager.yaml
kubectl get pods --namespace cert-manager
```
Ensure all cert-manager pods are running before proceeding.[10]

### 2. Create GitHub Personal Access Token & Kubernetes Secret

- Generate a PAT via GitHub → Settings → Developer settings → Personal access tokens (classic), grant at least `repo` scope.[10]
- Create the namespace and the secret:

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
Check for ARC resources to be active and running.[10]

### 4. Deploy the GitHub Actions Runner

Create `runner.yml` using your org/repo:

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
Apply and verify:

```shell
kubectl create -f runner.yml
kubectl get pod -n actions-runner-system | grep k8s-action-runner
```
You should see the runner in your repo’s GitHub → Settings → Actions → Runners tab.[10]

***

## Testing: Example Workflow

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
Pushing to `main` triggers workflow jobs executed on your EKS-based runner pod.[10]

***

## References

- [How to Setup GitHub Actions Runner on AWS EKS Cluster (DevOpsCube)][10]
- [Self-hosted runners documentation][1]
- [ARC and Kubernetes integration][5][2]

***

This README provides both theory and hands-on practice for running your own scalable, resilient GitHub Actions runners in AWS EKS using ARC.[1][2][5][10]

[1](https://docs.github.com/actions/hosting-your-own-runners)
[2](https://docs.github.com/en/actions/reference/runners/self-hosted-runners)
[3](https://docs.github.com/articles/getting-started-with-github-actions)
[4](https://github.blog/enterprise-software/ci-cd/when-to-choose-github-hosted-runners-or-self-hosted-runners-with-github-actions/)
[5](https://aws.amazon.com/blogs/devops/best-practices-working-with-self-hosted-github-action-runners-at-scale-on-aws/)
[6](https://www.youtube.com/watch?v=2wJU225zKOw)
[7](https://docs.github.com/actions/using-github-hosted-runners/about-github-hosted-runners)
[8](https://www.arnica.io/blog/github-hosted-or-self-hosted-runners)
[9](https://stackoverflow.com/questions/64457842/how-does-github-action-self-hosted-runner-work)
[10](https://devopscube.com/github-actions-runner-aws-eks/)
