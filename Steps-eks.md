# EKS Cluster Setup (AWS Free-Tier Account)

> ⚠️ **Cost note:** EKS control plane and worker nodes are **not** Free Tier eligible. Running this will incur charges (~$0.10/hr for the cluster + EC2 costs for nodes). Delete the cluster when done (`eksctl delete cluster --name=my-eks22 --region=ap-south-1`) to avoid ongoing charges.

## 1. Create an IAM User

1. Go to **IAM → Users → Create user** (any name works, e.g. `jenkins-eks-user`)
2. Attach the following **AWS managed policies**:
   - `AmazonEC2FullAccess`
   - `AmazonEKS_CNI_Policy`
   - `AmazonEKSClusterPolicy`
   - `AmazonEKSWorkerNodePolicy`
   - `AWSCloudFormationFullAccess`
   - `IAMFullAccess`

3. Create one **custom inline policy** (name it e.g. `eks-full-access`) with this content, and attach it too:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "eks:*",
            "Resource": "*"
        }
    ]
}
```

4. Generate an **Access Key / Secret Key** for this user (needed for `aws configure` below).

---

## 2. Install AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install
aws configure
```
When prompted, enter the Access Key/Secret Key of the IAM user created above, and set region to `ap-south-1` (Mumbai).

---

## 3. Install kubectl

```bash
curl -o kubectl https://s3.us-west-2.amazonaws.com/amazon-eks/1.30.0/2024-05-12/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --client
```
*(Updated to a current 1.30 kubectl build — the original 1.19.6 URL is outdated and won't match your cluster version.)*

---

## 4. Install eksctl

```bash
curl --silent --location "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```
*(Note: the `eksctl` project moved from `weaveworks/eksctl` to `eksctl-io/eksctl`; the old URL may 404.)*

---

## 5. Create the EKS Cluster (cost-minimized for testing)

```bash
eksctl create cluster --name=my-eks \
                      --region=ap-south-1 \
                      --zones=ap-south-1a,ap-south-1b \
                      --version=1.31 \
                      --without-nodegroup

eksctl utils associate-iam-oidc-provider \
    --region ap-south-1 \
    --cluster my-eks \
    --approve

eksctl create nodegroup --cluster=my-eks \
                       --region=ap-south-1 \
                       --name=node2 \
                       --node-type=t3.small \
                       --nodes=1 \
                       --nodes-min=1 \
                       --nodes-max=2 \
                       --node-volume-size=20 \
                       --ssh-access \
                       --ssh-public-key=YOUR_KEY_PAIR_NAME \
                       --managed \
                       --asg-access \
                       --external-dns-access \
                       --full-ecr-access \
                       --appmesh-access \
                       --alb-ingress-access
```

> 💡 Reduced from `t3.medium` × 2–4 nodes to `t3.small` × 1–2 nodes to keep this affordable for testing. Bump these up later if you need more capacity. Replace `YOUR_KEY_PAIR_NAME` with an existing EC2 key pair name in `ap-south-1`.

---

## 6. Open Inbound Traffic

In the **additional Security Group** eksctl creates for the nodegroup, open the ports your app needs (e.g., NodePort range `30000–32767`, or your app's specific port) under **Inbound rules**.

---

## 7. Create Service Account, Role, RoleBinding & Token

### Service Account
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: webapps
```

### Role
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: webapps
rules:
  - apiGroups:
      - ""
      - apps
      - autoscaling
      - batch
      - extensions
      - policy
      - rbac.authorization.k8s.io
    resources:
      - pods
      - secrets
      - componentstatuses
      - configmaps
      - daemonsets
      - deployments
      - events
      - endpoints
      - horizontalpodautoscalers
      - ingress
      - jobs
      - limitranges
      - namespaces
      - nodes
      - persistentvolumes
      - persistentvolumeclaims
      - resourcequotas
      - replicasets
      - replicationcontrollers
      - serviceaccounts
      - services
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

### RoleBinding
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-rolebinding
  namespace: webapps
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-role
subjects:
  - namespace: webapps
    kind: ServiceAccount
    name: jenkins
```

### Generate a Token for the Service Account
```bash
kubectl create token jenkins -n webapps --duration=8760h
```
*(Use `kubectl create token` — the older Kubernetes docs page for creating a long-lived secret-based token is deprecated as of K8s 1.24+; this is the current supported method.)*

---

### Cleanup (important on Free Tier)
```bash
eksctl delete cluster --name=my-eks22 --region=ap-south-1
```
Run this once you're done to stop charges.
