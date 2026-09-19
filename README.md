# AutoLink — Containerized App Deployment on AWS EKS

A car-focused social platform prototype, containerized with Docker and deployed to a real Kubernetes cluster on AWS EKS, provisioned entirely with Terraform.

Built to actually learn Docker and Kubernetes hands-on instead of just watching tutorials. Everything under the app — the container, the cluster, the networking — is the real point of this project.

**Stack:** Docker · Kubernetes · AWS EKS · Terraform · GitHub Actions (in progress)

## Architecture

```
Internet
   │
   ▼
AWS Load Balancer (created automatically by a Kubernetes Service)
   │
   ▼
2x EC2 Worker Nodes (t3.micro, across two subnets/AZs)
   │
   ▼
2x Pods (Deployment, replicas: 2) — each running the AutoLink container
```

## Stack Details

- Dockerfile builds the app on `nginx:alpine`
- Kubernetes Deployment (2 replicas) and Service (`type: LoadBalancer`)
- EKS cluster + node group, IAM roles, VPC, subnets, Internet Gateway, and route tables — all in Terraform
- GitHub Actions pipeline to automate build → push → deploy (in progress)

## Notes from the build

- Image built on my Mac (Apple Silicon/arm64) but EKS nodes run on amd64 — pods sat in `ImagePullBackOff` until I rebuilt with `docker build --platform linux/amd64`
- Rebuilding on the correct platform still left a rollout stuck: `0/2 nodes are available: 2 Too many pods`, since the old and new pods briefly needed to run at once and my two `t3.micro` nodes didn't have room. Scaled the old ReplicaSet to zero to force it to hand over the space
- Node group failed to launch the first time, default instance type wasn't Free Tier eligible on this account, set `instance_types = ["t3.micro"]` explicitly
- Couldn't connect `kubectl` to a cluster I'd just created, newer EKS versions don't auto-grant the creator admin access, needed `bootstrap_cluster_creator_admin_permissions = true`
- That last setting can't be changed in place, updating it forces Terraform to destroy and recreate the whole cluster

## Testing

Confirmed pods healthy with `kubectl get pods`, then hit the Load Balancer's public URL from `kubectl get services` directly in a browser to confirm the app was actually reachable end to end.

## Teardown

```
terraform destroy
```

The EKS control plane and worker nodes are the cost drivers here, so this gets destroyed between sessions.

## Running it yourself

Needs AWS CLI, Terraform, kubectl, and Docker installed.

```
git clone https://github.com/jdrakegit/docker-kubernetes-eks-cicd.git
cd docker-kubernetes-eks-cicd

terraform init
terraform apply
```

Point kubectl at the new cluster:

```
aws eks update-kubeconfig --region us-east-1 --name eks_cluster
```

Deploy the app:

```
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

Grab the public URL from `kubectl get services` once the Load Balancer finishes provisioning.

## What's next

Finishing the GitHub Actions pipeline so a push to `main` automatically builds, pushes, and redeploys, no manual steps.

---

Built by [Jordan Drake](https://github.com/jdrakegit) · [LinkedIn](https://www.linkedin.com/in/jordan-drake-a95471397)
