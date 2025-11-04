# Multi-cluster Hub‑Spoke with Argo CD

## Overview

In a **Hub‑and‑Spoke GitOps model**, one Kubernetes cluster acts as the **Hub** hosting a central GitOps controller like **Argo CD**. This hub doesn't run applications itself; instead, it manages application deployments across multiple **Spoke clusters** (e.g., staging, production, dev). This structure allows organizations to scale deployments to many clusters while maintaining centralized governance.

Argo CD follows the GitOps model, where the desired state of your Kubernetes applications and environments lives in Git repositories. Argo CD continuously watches these Git repositories and ensures the cluster state always matches what’s declared in Git.

In short: If it’s not in Git, Argo CD won’t deploy it.
Git becomes the single trusted source for what should run in production.

> **Warning:** This PROJECT includes a workflow that exposes Argo CD using a NodePort — for production use secure LoadBalancers, TLS, and SSO.

---

## Architecture

* **Hub**: EKS cluster running Argo CD (manages apps, policies, and cluster connections).
* **Spokes**: One or more EKS clusters that run workloads. Argo CD performs deployments to these clusters.

Benefits: centralized deployment control, reproducible GitOps workflows, easier audit and RBAC. Tradeoffs: hub becomes a critical control plane and must be secured.

![WhatsApp Image 2025-11-04 at 3 47 37 PM](https://github.com/user-attachments/assets/7f621b48-ee9e-4362-b5e3-b8fa0b4383cb)


---

## Prerequisites

Install and configure these CLIs:

* **kubectl** — Kubernetes CLI (configure kubeconfig contexts for each cluster).

  * [https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)

* **eksctl** — EKS cluster management helper.

  * [https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)

* **AWS CLI** — AWS command line tool. Configure with `aws configure`.

  * Install: [https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)
  * Configure: [https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-config](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-config)

* **argocd CLI** — Argo CD client (used to register clusters and interact with Argo CD).

  * [https://argo-cd.readthedocs.io/en/stable/cli_installation/#installation](https://argo-cd.readthedocs.io/en/stable/cli_installation/#installation)

You will also need IAM permissions to create/delete EKS clusters and edit Security Groups.

---

## Create clusters

```bash
eksctl create cluster --name hub-cluster --region us-east-1
eksctl create cluster --name staging-cluster --region us-east-1
eksctl create cluster --name production-cluster --region us-east-1
```

Confirm contexts are present:

```bash
kubectl config get-contexts
```

---

## Switch to hub Cluster, Install Argo CD on the Hub cluster

1. Switch to the hub cluster context:

```bash
kubectl config use-context <hub-context-name>
```

2. Create namespace and install:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

3. Verify pods are ready:

```bash
kubectl get pods -n argocd
```

---

## Expose Argo CD server using NodePort (demo only)

Change the `argocd-server` service type to NodePort:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
kubectl get svc argocd-server -n argocd
```

Find node external IP(s):

```bash
kubectl get nodes -o wide
```

Access Argo CD UI: `https://<node-external-ip>:<nodePort>` 

Update AWS Security Group to allow the NodePort from trusted IPs.

---

## Retrieve initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode; echo
```

Login to the UI with username `admin` and the password above. Change the password immediately if this is not a demo.

---

## Install Argo CD CLI (client)

```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

---

## Add Spoke clusters to Argo CD (CLI)

> The Argo CD GUI cannot add clusters — use the `argocd` CLI.

1. Ensure your kubeconfig has contexts for each spoke cluster: `kubectl config get-contexts`.
2. Login to Argo CD with the CLI:

```bash
argocd login <ARGOCD_SERVER_HOST>:<PORT>
# example: argocd login 54.162.117.164:31306
```

3. Add each spoke cluster using its kubeconfig context name:

```bash
argocd cluster add <spoke-context-name> --server <ARGOCD_SERVER_HOST>:<PORT>
# example:
argocd cluster add sammy@production-cluster.us-east-1.eksctl.io --server 54.162.117.164:31306
argocd cluster add sammy@staging-cluster.us-east-1.eksctl.io --server 54.162.117.164:31306
```

This sets up a ServiceAccount and RBAC in the spoke cluster so Argo CD can deploy into it.

Verify clusters in the UI under **Settings → Clusters** or via CLI:

```bash
argocd cluster list
```

---

## Create Applications (per environment)

* Create applications via the UI (Applications → Create Application) or declaratively (YAML manifests).
* For each environment use unique names: `guest-book-staging`, `guest-book-prod`.
* Destination: choose the target spoke cluster and namespace.

Tip: Organize Git repos with environment folders (`environments/staging`, `environments/production`) or use ApplicationSets for many clusters.

---

## Verify & Troubleshooting

* Check Argo CD pods:

```bash
kubectl get pods -n argocd
```

* View logs:

```bash
kubectl logs -n argocd deploy/argocd-server
```

* Ensure:

  * Your kubeconfig context points to the target spoke cluster when running `argocd cluster add`.
  * You have cluster-admin permissions on the spoke clusters.
  * The Argo CD server is reachable from where you run the `argocd` CLI.

---

## Security recommendations (production)

* Use a **LoadBalancer** or **Ingress (ALB)** with TLS for Argo CD, and enable SSO (OIDC) and RBAC.
* Don’t expose Argo CD NodePorts to 0.0.0.0/0. Limit access via Security Groups.

---

## Delete clusters (cleanup)

```bash
eksctl delete cluster --name hub-cluster --region us-east-1
eksctl delete cluster --name staging-cluster --region us-east-1
eksctl delete cluster --name production-cluster --region us-east-1
```

---

## Checklist

* [ ] Create hub + spoke clusters with `eksctl`.
* [ ] Install Argo CD on hub and verify pods.
* [ ] Expose Argo CD (NodePort for demo) and open Security Group port (restricted).
* [ ] Install Argo CD CLI and login.
* [ ] Add spoke clusters (`argocd cluster add`).
* [ ] Create application per environment (unique names).



*Created by Osung.*
