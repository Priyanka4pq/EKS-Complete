# Install Helm on Ubuntu

A quick reference for installing [Helm](https://helm.sh/) (Kubernetes package manager) on Ubuntu.

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

## Uninstall / Cleanup

Helm stores its files in these default locations on Linux:

| Type | Path |
|------|------|
| Cache | `$HOME/.cache/helm` |
| Config | `$HOME/.config/helm` |
| Data | `$HOME/.local/share/helm` |

To fully remove Helm, delete the binary along with the above folders.

---

## Reference

- Official docs: https://helm.sh/docs/intro/install/
