# EKS Cluster Autoscaler Setup

This repo documents the setup of [Kubernetes Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler) on an Amazon EKS cluster, using IRSA (IAM Roles for Service Accounts) for AWS permissions and Helm for deployment.

## Overview

Cluster Autoscaler automatically adjusts the size of the EKS node group's Auto Scaling Group (ASG) based on pod scheduling demand — scaling up when pods are unschedulable due to resource constraints, and scaling down when nodes are underutilized.

**Environment used in this setup:**
- Cluster name: `my-eks`
- Region: `ap-south-1`
- AWS Account ID: `930171196802`

---

## Prerequisites

- An existing EKS cluster with an IAM OIDC provider associated (required for IRSA)
- `awscli`, `eksctl`, and `kubectl` installed and configured
- `helm` installed (see [Installing Helm](#installing-helm) below)

Verify the OIDC provider is set up for the cluster:

```bash
aws eks describe-cluster --name my-eks --region ap-south-1 \
  --query "cluster.identity.oidc.issuer" --output text

aws iam list-open-id-connect-providers
```

---

## 1. Create the IAM Policy

Cluster Autoscaler needs permission to describe and manage the ASG.

```bash
cat <<EOF > cluster-autoscaler-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeTags",
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "ec2:DescribeLaunchTemplateVersions"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name AmazonEKSClusterAutoscalerPolicy \
  --policy-document file://cluster-autoscaler-policy.json
```

Note the returned policy ARN — you'll need it in the next step.

---

## 2. Create the IAM Service Account (IRSA)

This creates an IAM role trusted by the cluster's OIDC provider, attaches the policy above, and creates a matching Kubernetes `ServiceAccount`.

```bash
eksctl create iamserviceaccount \
  --cluster my-eks \
  --region ap-south-1 \
  --namespace kube-system \
  --name cluster-autoscaler \
  --attach-policy-arn arn:aws:iam::970009400005:policy/AmazonEKSClusterAutoscalerPolicy \
  --approve \
  --override-existing-serviceaccounts
```

> `--override-existing-serviceaccounts` updates an existing `ServiceAccount` with the IAM role annotation instead of failing if one already exists.

---

## 3. Tag the Auto Scaling Group

Cluster Autoscaler auto-discovers node groups using tags. Ensure the ASG backing your node group has:

| Key | Value |
|---|---|
| `k8s.io/cluster-autoscaler/enabled` | `true` |
| `k8s.io/cluster-autoscaler/my-eks` | `owned` |

---

## Installing Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
helm version
```

See [helm.sh/docs/intro/install](https://helm.sh/docs/intro/install/) for alternative install methods (apt, snap).

---

## 4. Add the Cluster Autoscaler Helm Repo

```bash
helm repo add cluster-autoscaler https://kubernetes.github.io/autoscaler
helm repo update
```

---

## 5. Deploy via Helm

Configuration is managed in [`autoscaler_values.yaml`](./autoscaler_values.yaml). Key values used:

```yaml
autoDiscovery:
  clusterName: "my-eks"
awsRegion: "ap-south-1"
cloudProvider: aws
rbac:
  serviceAccount:
    create: false
    name: cluster-autoscaler
extraArgs:
  balance-similar-node-groups: "true"
  scale-down-utilization-threshold: "0.5"
  scale-down-delay-after-add: "2m"
  scale-down-unneeded-time: "5m"
  skip-nodes-with-system-pods: "false"
```

> `rbac.serviceAccount.create: false` + matching `name` tells the chart to reuse the IRSA-linked `ServiceAccount` from step 2, instead of creating a new one without AWS permissions.

Install/upgrade:

```bash
helm upgrade --install cluster-autoscaler cluster-autoscaler/cluster-autoscaler \
  -f autoscaler_values.yaml \
  -n kube-system
```

---

## Verifying It's Working

```bash
# Check pod status
kubectl get pods -n kube-system | grep cluster-autoscaler

# Watch autoscaler decision logs
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-cluster-autoscaler --tail=100 -f

# Check the ASG's current desired/min/max
aws autoscaling describe-auto-scaling-groups --region ap-south-1 \
  --query "AutoScalingGroups[].{Name:AutoScalingGroupName,Min:MinSize,Desired:DesiredCapacity,Max:MaxSize,Instances:length(Instances)}"
```

A healthy scale-down cycle will show log lines like `Scale down status: ... scaleDownInCooldown=true` right after a node is removed — this is the cooldown period, not an error.

---

## Troubleshooting

| Symptom | Likely Cause |
|---|---|
| Pod in `CrashLoopBackOff` | Malformed `autoscaler_values.yaml` (check for trailing whitespace in string values like `awsRegion`), wrong `cloudProvider`, or missing `autoDiscovery.clusterName` |
| `helm repo add` fails with arg count error | Command was duplicated/malformed in the shell — re-run cleanly |
| `eksctl` returns `ResourceNotFoundException` for the cluster | Wrong `--cluster` name/region, or AWS credentials pointing at the wrong account — check with `aws sts get-caller-identity` and `aws eks list-clusters --region <region>` |
| Nodes not scaling down | Check for leftover pods on the node, PodDisruptionBudgets, `safe-to-evict: "false"` annotations, or the ASG already at `MinSize` |
| ASG `Instances` count doesn't match `Desired` | Give it a cycle — `scale-down-unneeded-time` (5m) + `scale-down-delay-after-add` (2m) cooldowns apply before action is taken |

---

## Teardown

```bash
# 1. Remove the Helm release
helm uninstall cluster-autoscaler -n kube-system

# 2. Remove the IAM service account / role
eksctl delete iamserviceaccount \
  --cluster my-eks \
  --region ap-south-1 \
  --namespace kube-system \
  --name cluster-autoscaler

# 3. Remove the IAM policy
aws iam delete-policy \
  --policy-arn arn:aws:iam::930171196802:policy/AmazonEKSClusterAutoscalerPolicy

# 4. (Optional) Delete the whole cluster
eksctl delete cluster --name my-eks --region ap-south-1
```

Verify nothing was left behind:

```bash
aws eks list-clusters --region ap-south-1
aws iam list-open-id-connect-providers
aws autoscaling describe-auto-scaling-groups --region ap-south-1 --query "AutoScalingGroups[].AutoScalingGroupName"
```

---

## References

- [Cluster Autoscaler on AWS — official docs](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/README.md)
- [Cluster Autoscaler FAQ / CLI flags](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md)
- [IAM Roles for Service Accounts (IRSA)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [Helm install docs](https://helm.sh/docs/intro/install/)
