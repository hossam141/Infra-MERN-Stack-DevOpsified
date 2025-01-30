# AWS Load Balancer Controller Setup Guide

## 🎯 **Aim**  
Deploy the AWS Load Balancer Controller to **automatically provision and manage Application/Network Load Balancers (ALB/NLB)** in your AWS account based on Kubernetes `Ingress` and `Service` resources.

---

## 📋 Prerequisites
- AWS CLI configured with credentials
- `eksctl`, `kubectl`, and `helm` installed
- Existing EKS cluster (`dev-medium-eks-cluster` in this example)
- IAM permissions to create policies/roles

---

## 🚀 Deployment Steps

### 1. Create IAM Policy
```bash
# Download policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json

# Create policy
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json \
  --region us-east-1
```

---

### 2. Create IAM Role & Service Account
```bash
eksctl create iamserviceaccount \
  --cluster=dev-medium-eks-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<YOUR-AWS-ACCOUNT-ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve \
  --region us-east-1
```

> 🔑 Replace `<YOUR-AWS-ACCOUNT-ID>` with your actual AWS account ID.

---

### 3. Install Helm Chart
```bash
# Add EKS Helm repo
helm repo add eks https://aws.github.io/eks-charts

# Update repos
helm repo update

# Install controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=dev-medium-eks-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

---

### 4. Verify Installation
```bash
# Check deployment status
kubectl get deployment -n kube-system aws-load-balancer-controller

# View logs
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

---

## 🧪 Test with Sample Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: your-service
                port:
                  number: 80
```

---

## 🔍 Troubleshooting
| Issue                  | Solution                                  |
|------------------------|-------------------------------------------|
| Controller not starting | Check IAM role ARN and cluster name       |
| ALB not created        | Verify Ingress annotations and pod logs   |
| 403 Forbidden errors   | Validate IAM policy attachments           |

---

## 🧹 Cleanup
```bash
# Uninstall controller
helm uninstall aws-load-balancer-controller -n kube-system

# Delete IAM resources
eksctl delete iamserviceaccount \
  --cluster=dev-medium-eks-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller

aws iam delete-policy \
  --policy-arn arn:aws:iam::<YOUR-AWS-ACCOUNT-ID>:policy/AWSLoadBalancerControllerIAMPolicy
```

---

📚 [Official AWS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html)  
📦 [GitHub Repository](https://github.com/kubernetes-sigs/aws-load-balancer-controller)