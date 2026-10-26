# AWS EKS with Terraform, ArgoCD and Helm (GitOps)

An Amazon EKS cluster and its VPC provisioned entirely with Terraform, with applications deployed through ArgoCD using a GitOps workflow. ArgoCD itself is installed with Helm.

## What This Project Does

The project is built in three layers:

1. **Infrastructure:** Terraform builds a custom VPC across two Availability Zones, with private subnets for the worker nodes, and an EKS cluster with a managed node group inside it. One `terraform apply` creates 56 resources.
2. **Access:** kubectl connects to the cluster through the AWS CLI using `aws eks update-kubeconfig`.
3. **Deployment:** ArgoCD is installed into the cluster with Helm. It watches this Git repository and keeps the cluster matching it. ArgoCD deploys my own Helm chart, hello-app, straight from the `charts/hello-app` folder. Push a change and it deploys automatically. Change the cluster by hand and ArgoCD puts it back.

## Architecture

```
GitHub repo (source of truth)
        |
        v
ArgoCD (installed via Helm, runs in the argocd namespace)
        |
        v
EKS cluster (Kubernetes 1.35, managed node group, 2 worker nodes)
        |
        v
Custom VPC (2 Availability Zones, private subnets)
```

## Tech Stack

- **Terraform** with the `terraform-aws-modules` VPC and EKS modules
- **Amazon EKS** (Kubernetes 1.35)
- **kubectl** and the **AWS CLI**
- **Helm** (v4)
- **ArgoCD**

## Repository Layout

| File / Folder | Purpose |
| --- | --- |
| `Provider.tf` | AWS provider and Terraform settings |
| `VPC.tf` | Custom VPC, subnets, and networking across two AZs |
| `EKS.tf` | EKS cluster and managed node group |
| `Outputs.tf` | Cluster details and the kubectl configuration command |
| `charts/hello-app` | My Helm chart, deployed to the cluster by ArgoCD |
| `Screen Shots` | Proof of the working build |

## How to Run It

```bash
# 1. Build the infrastructure
terraform init
terraform plan
terraform apply

# 2. Connect kubectl to the cluster
aws eks update-kubeconfig --region us-east-1 --name james-eks
kubectl get nodes

# 3. Install ArgoCD with Helm
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd --namespace argocd --create-namespace
kubectl get pods -n argocd

# 4. In ArgoCD, create an app pointing at this repo, path charts/hello-app

# 5. Tear everything down when finished
terraform destroy
```

## Results

**Both worker nodes Ready, one in each Availability Zone, running Kubernetes 1.35**

![Nodes ready](Screen%20Shots/01-nodes-ready.png)

**ArgoCD dashboard running in the cluster**

![ArgoCD dashboard](Screen%20Shots/02-argocd-dashboard.png)

**Guestbook sample app deployed from GitHub by ArgoCD, Healthy and Synced**

![Guestbook synced](Screen%20Shots/03-guestbook-synced.png)

**My own hello-app Helm chart, deployed by ArgoCD straight from this repo, Healthy and Synced**

![Hello app synced](Screen%20Shots/04-hello-app-synced.png)

## Troubleshooting

### 1. Terraform "Invalid count argument" in the EKS module

**Problem:** `terraform plan` threw an "Invalid count argument" error inside the EKS module's node group logic. Terraform could not work out how many resources to create before apply because no Kubernetes version was set.

**Fix:** I pinned `kubernetes_version = "1.35"` in `EKS.tf`. I chose one version below my installed kubectl (1.36) to stay within Kubernetes' version skew policy. After that, `terraform plan` ran cleanly with 56 resources to add.

### 2. Helm would not install on an Intel Mac

**Problem:** `brew install helm` failed because Homebrew no longer provides prebuilt packages for Intel Macs, and building from source failed on outdated Command Line Tools.

**Fix:** I bypassed Homebrew and used Helm's official install script, which installed Helm v4 directly.

## Cost Note

The cluster and its node group cost money while running. Run `terraform destroy` when you are done so nothing is left behind.

## Author

**James Victor**, Cloud Engineer
[LinkedIn](https://www.linkedin.com/in/james-victor-a9460b21/) | [GitHub](https://github.com/JamesVCloudEngineer) | [Blog](https://jamesanthonyjourney.wordpress.com)