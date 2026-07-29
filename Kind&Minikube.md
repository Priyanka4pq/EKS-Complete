# kind & minikube — Local Kubernetes for Learning Helm/kubectl

When you're learning Helm or Kubernetes, you don't need a real cloud cluster (like EKS, GKE, or AKS). A local cluster running on your own machine behaves the same way from `kubectl`'s and `helm`'s point of view — it's just faster, free, and easier to tear down and recreate.

Two popular tools for this: **kind** and **minikube**.

## Prerequisite: Docker

Both tools need a container runtime. Docker is the simplest option:

```bash
sudo apt update
sudo apt install -y docker.io
sudo usermod -aG docker $USER
newgrp docker   # or log out/in for group change to take effect
docker version
```

---

## Option 1: kind (Kubernetes IN Docker)

`kind` runs an entire Kubernetes cluster inside Docker containers. It's lightweight and quick to spin up/tear down — great for fast iteration.

### Install

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind version
```

### Create a cluster

```bash
kind create cluster --name learning
```

This automatically updates your kubeconfig (`~/.kube/config`) and switches context to the new cluster.

### Verify

```bash
kubectl cluster-info --context kind-learning
kubectl get nodes
```

### Delete the cluster when done

```bash
kind delete cluster --name learning
```

---

## Option 2: minikube

`minikube` runs Kubernetes in a VM or container on your machine. It's slightly heavier than kind but ships with more built-in add-ons (dashboard, ingress, metrics-server, etc.), which can be handy for exploring beyond the basics.

### Install

```bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube
sudo mv minikube /usr/local/bin/
minikube version
```

### Start a cluster

```bash
minikube start
```

By default this uses Docker as the driver if available. You can specify a different driver explicitly:

```bash
minikube start --driver=docker
```

### Verify

```bash
kubectl cluster-info
kubectl get nodes
```

### Useful extras

```bash
minikube dashboard      # opens the Kubernetes web dashboard
minikube addons list     # see available add-ons
```

### Stop / delete the cluster when done

```bash
minikube stop
minikube delete
```

---

## Using Helm once your cluster is up

Whether you used kind or minikube, once `kubectl get nodes` shows a node in `Ready` state, Helm works exactly the same:

```bash
# Add a chart repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Install a chart
helm install nginxv1 bitnami/nginx

# Check it deployed
kubectl get pods
kubectl get svc

# Uninstall when done
helm uninstall nginxv1
```

---

## kind vs minikube — quick comparison

| | kind | minikube |
|---|---|---|
| Runs on | Docker containers | VM or container driver |
| Startup speed | Very fast | Fast, slightly slower than kind |
| Built-in add-ons | Minimal | Many (dashboard, ingress, metrics-server, etc.) |
| Best for | CI pipelines, quick throwaway clusters, multi-node testing | Exploring Kubernetes features interactively |
| Multi-node clusters | Easy via config file | Supported, less common use case |

For pure Helm practice, either works fine. `kind` is great if you want something fast and disposable; `minikube` is great if you want to poke around a fuller-featured cluster (dashboard, add-ons) while you learn.

---

## Common gotcha

If `kubectl` or `helm` ever gives you:

```
The connection to the server localhost:8080 was refused
```

it means your kubeconfig isn't pointing at a running cluster. Check:

```bash
kubectl config current-context
kubectl config get-contexts
```

If nothing is running, start (or recreate) your kind/minikube cluster and try again.
