# Install Helm on Ubuntu

Official installer script method (recommended — always fetches the latest stable version):

\`\`\`bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
\`\`\`

Verify installation:

\`\`\`bash
helm version
\`\`\`

---

## Alternative: Apt (Debian/Ubuntu)

\`\`\`bash
sudo apt-get install curl gpg apt-transport-https --yes

curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey > "${TMPDIR:-/tmp}/helm.gpg"

cat "${TMPDIR:-/tmp}/helm.gpg" | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

sudo apt-get update
sudo apt-get install helm
\`\`\`

---

## Alternative: Snap

\`\`\`bash
sudo snap install helm --classic
\`\`\`
