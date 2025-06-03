## Prerequisites
- **AWS CLI** installed and configured
- **kubectl** installed
- **eksctl** installed
- AWS account with appropriate permissions

## 1. Create EKS Cluster
```bash
eksctl create cluster --name demo-cluster --region us-east-1
```

## 2. Configure IAM OIDC Provider
```bash
eksctl utils associate-iam-oidc-provider --cluster demo-cluster --approve
```

## 3. Setup ALB Add-on

### Download IAM Policy
```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```

### Create IAM Policy
```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

### Create IAM Role and Service Account
Replace placeholders with your values:
- `<your-cluster-name>` - Your EKS cluster name
- `<your-aws-account-id>` - Your AWS account ID

```bash
eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

## 4. Deploy ALB Controller

### Add Helm Repository
```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```

### Install ALB Controller
Replace placeholders:
- `<YOUR_CLUSTER_NAME>` - Your EKS cluster name
- `<YOUR_AWS_REGION>` - AWS region
- `<YOUR_VPC_ID>` - Your VPC ID

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=<YOUR_CLUSTER_NAME> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<YOUR_AWS_REGION> \
  --set vpcId=<YOUR_VPC_ID>
```

### Verify Deployment
```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

## 5. EBS CSI Plugin Configuration

### Create IAM Role and Service Account
Replace placeholders:
- `<YOUR-CLUSTER-NAME>` - Your EKS cluster name

```bash
eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster <YOUR-CLUSTER-NAME> \
    --role-name AmazonEKS_EBS_CSI_DriverRole \
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve
```

### Create EBS CSI Addon
Replace placeholders:
- `<YOUR-CLUSTER-NAME>` - Your EKS cluster name
- `<AWS-ACCOUNT-ID>` - Your AWS account ID

```bash
eksctl create addon --name aws-ebs-csi-driver \
  --cluster <YOUR-CLUSTER-NAME> \
  --service-account-role-arn arn:aws:iam::<AWS-ACCOUNT-ID>:role/AmazonEKS_EBS_CSI_DriverRole \
  --force
```

**Note**: For AWS GovCloud regions, replace `arn:aws:` with `arn:aws-us-gov:`

## 6. Deploy Application

### Clone Repository
```bash
git clone <my-url>
cd three-tier-architecture-demo/EKS/helm
```

### Deploy Using Helm
```bash
kubectl create ns robot-shop
helm install robot-shop --namespace robot-shop .
```

### Install Ingress
```bash
kubectl apply -f ingress.yaml
```

### Get Application URL
```bash
kubectl get svc -n robot-shop
```
Wait approximately 5 minutes for the ALB to become active before accessing the URL.

## References
- [EKS Persistent Storage Knowledge Center](https://repost.aws/knowledge-center/eks-persistent-storage)
- AWS Documentation for ALB Controller and EBS CSI Driver

---