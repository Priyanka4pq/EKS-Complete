# Install Helm on Ubuntu

Quick reference for installing [Helm](https://helm.sh/) (the Kubernetes package manager) on your Ubuntu instance, needed before installing Cluster Autoscaler via Helm chart.

---

## Method 1: Official Installer Script (Recommended)

Always fetches the latest stable release.

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
```

Verify installation:

```bash
helm version
```

---

## Method 2: Apt (Debian/Ubuntu)

```bash
sudo apt-get install curl gpg apt-transport-https --yes
curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey > "${TMPDIR:-/tmp}/helm.gpg"
cat "${TMPDIR:-/tmp}/helm.gpg" | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

---

## Method 3: Snap

```bash
sudo snap install helm --classic
```

---

## Next Step: Add the Cluster Autoscaler Helm Repo

Once `helm version` confirms Helm is installed, add the official autoscaler chart repo:

```bash
helm repo add cluster-autoscaler https://kubernetes.github.io/autoscaler
helm repo update
```

Then install it against your EKS cluster (`my-eks`, region `ap-south-1`), pointing at the IAM-linked ServiceAccount created earlier via `eksctl`:

```bash
helm install cluster-autoscaler cluster-autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=my-eks \
  --set awsRegion=ap-south-1 \
  --set rbac.serviceAccount.create=false \
  --set rbac.serviceAccount.name=cluster-autoscaler
```

> **Note:** `rbac.serviceAccount.create=false` + `rbac.serviceAccount.name=cluster-autoscaler` tell the chart to reuse the existing ServiceAccount (with the IAM role already attached via IRSA) instead of creating a new one without permissions.

---

## Uninstall / Cleanup

Helm stores its files in these default locations on Linux:

| Type   | Path                     |
|--------|--------------------------|
| Cache  | `$HOME/.cache/helm`      |
| Config | `$HOME/.config/helm`     |
| Data   | `$HOME/.local/share/helm`|

To fully remove Helm, delete the binary along with the above folders.

---

## Reference

- Official docs: https://helm.sh/docs/intro/install/
