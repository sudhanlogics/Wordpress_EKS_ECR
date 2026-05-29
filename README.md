# 🏗️ WordPress on Amazon EKS — Production Deployment Guide

> **Stack:** WordPress + PHP 8.1-FPM + Nginx · Amazon RDS MySQL 8.0 · Amazon EFS · EKS · ECR · ALB + ACM (HTTPS)
> **Region:** `ap-south-2` (Hyderabad) · Migrated from AWS ECS Fargate

---

## 🧭 Table of Contents

1. [Stack Overview](#1-stack-overview)
2. [Prerequisites](#2-prerequisites)
3. [VPC and Networking](#3-vpc-and-networking)
4. [Security Groups](#4-security-groups)
5. [EKS Cluster](#5-eks-cluster)
6. [AWS Load Balancer Controller](#6-aws-load-balancer-controller)
7. [EFS CSI Driver and Persistent Storage](#7-efs-csi-driver-and-persistent-storage)
8. [Amazon RDS MySQL](#8-amazon-rds-mysql)
9. [Secrets Manager and ExternalSecrets Operator](#9-secrets-manager-and-externalsecrets-operator)
10. [ECR — Container Registry](#10-ecr--container-registry)
11. [Kubernetes Manifests](#11-kubernetes-manifests)
12. [ACM Certificate and DNS](#12-acm-certificate-and-dns)
13. [Logging with Fluent Bit](#13-logging-with-fluent-bit)
14. [Apply Manifests and Verify](#14-apply-manifests-and-verify)
15. [Useful Commands](#15-useful-commands)
16. [Security Checklist](#16-security-checklist)
17. [Cleanup](#17-cleanup)

---

## 1. Stack Overview

| Component | ECS Design | EKS Design |
|---|---|---|
| Compute | ECS Fargate tasks | EKS Pods (managed node groups) |
| Ingress | Manual ALB + listeners | AWS Load Balancer Controller (Ingress resource) |
| WordPress | Nginx + PHP 8.1-FPM container | Same container image — no change |
| Database | MySQL 8.0 sidecar container | **Amazon RDS MySQL 8.0 (Multi-AZ)** |
| Persistent storage | EFS via task definition volumes | EFS CSI Driver + PersistentVolumeClaim (RWX) |
| Secrets | ECS secrets from Secrets Manager | ExternalSecrets Operator or AWS ASCP |
| IAM | ECS Task Role | IRSA (IAM Roles for Service Accounts) |
| Scaling | ECS Service desired count | HorizontalPodAutoscaler (min 2 / max 10) |
| Logging | `awslogs` driver → CloudWatch | Fluent Bit DaemonSet → CloudWatch Container Insights |
| Deployment | `aws ecs update-service` | `kubectl rollout` / `helm upgrade` / GitOps |

### Architecture

```
Internet (HTTPS :443)
        │
        ▼
  ALB (public subnets) ◄── ACM Certificate
  HTTP :80 → HTTPS :443 redirect
  X-Forwarded-Proto: https
        │
        ▼
  EKS Ingress (AWS Load Balancer Controller)
        │
        ▼
  Kubernetes Service (ClusterIP :80)
        │
        ▼
  Deployment — replicas: 2+  (private subnets)
  ┌──────────────────────────────────────────────┐
  │  Pod A                    Pod B              │
  │  ┌──────────────────┐  ┌──────────────────┐  │
  │  │ wordpress :80    │  │ wordpress :80    │  │
  │  │ Nginx+PHP-FPM    │  │ Nginx+PHP-FPM    │  │
  │  └──────────────────┘  └──────────────────┘  │
  │         │                      │             │
  │         └──────────┬───────────┘             │
  └──────────────────────────────────────────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
     Amazon EFS    Amazon RDS    Secrets Manager
     (wp-content   MySQL 8.0     (credentials via
      PVC RWX)     Multi-AZ)      ExternalSecrets)

  Public Subnets:  ALB + NAT Gateway
  Private Subnets: EKS Nodes + EFS Mount Targets + RDS
```

> **Key change from ECS:** MySQL moves from a sidecar container to Amazon RDS. With multiple pod replicas, each pod would otherwise spin up its own MySQL instance and corrupt the shared EFS data directory. RDS Multi-AZ is the correct pattern for horizontally scalable WordPress.

---

## 2. Prerequisites

- **AWS CLI v2** — `aws configure` with credentials and region
- **kubectl** — Kubernetes CLI
- **eksctl** — EKS cluster provisioner
- **Helm 3** — for Load Balancer Controller and EFS CSI charts
- **Docker** — local image build and push to ECR
- A registered **domain name** with DNS access

```bash
REGION="ap-south-2"
CLUSTER_NAME="wordpress-eks"
aws configure set region $REGION
aws sts get-caller-identity
```

---

## 3. VPC and Networking

The VPC layout is identical to the ECS design — two public subnets for the ALB and NAT Gateway, two private subnets for workloads. EKS requires additional subnet tags so the Load Balancer Controller can discover where to place ALBs.

### 3.1 Create VPC

```bash
VPC_CIDR="10.0.0.0/16"

VPC_ID=$(aws ec2 create-vpc \
  --cidr-block $VPC_CIDR --region $REGION \
  --query 'Vpc.VpcId' --output text)

aws ec2 create-tags --resources $VPC_ID \
  --tags Key=Name,Value=wordpress-eks-vpc \
         Key=kubernetes.io/cluster/$CLUSTER_NAME,Value=shared

aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support

echo "✅ VPC: $VPC_ID"
```

### 3.2 Subnets

```bash
# Public subnets — ALB, NAT Gateway
PUB_SUBNET1=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 --availability-zone ${REGION}a \
  --query 'Subnet.SubnetId' --output text)
PUB_SUBNET2=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.2.0/24 --availability-zone ${REGION}b \
  --query 'Subnet.SubnetId' --output text)

# Required tag: tells ALB controller these are public-facing
for s in $PUB_SUBNET1 $PUB_SUBNET2; do
  aws ec2 create-tags --resources $s --tags \
    Key=kubernetes.io/role/elb,Value=1 \
    Key=kubernetes.io/cluster/$CLUSTER_NAME,Value=shared
done

# Private subnets — EKS nodes/pods, EFS, RDS
PRI_SUBNET1=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.3.0/24 --availability-zone ${REGION}a \
  --query 'Subnet.SubnetId' --output text)
PRI_SUBNET2=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.4.0/24 --availability-zone ${REGION}b \
  --query 'Subnet.SubnetId' --output text)

# Required tag: tells ALB controller these are internal
for s in $PRI_SUBNET1 $PRI_SUBNET2; do
  aws ec2 create-tags --resources $s --tags \
    Key=kubernetes.io/role/internal-elb,Value=1 \
    Key=kubernetes.io/cluster/$CLUSTER_NAME,Value=shared
done

echo "✅ Public:  $PUB_SUBNET1, $PUB_SUBNET2"
echo "✅ Private: $PRI_SUBNET1, $PRI_SUBNET2"
```

### 3.3 Internet Gateway, NAT Gateway, Route Tables

Private subnets route `0.0.0.0/0` via NAT — not the IGW. Without this, pods cannot pull images from ECR or reach Secrets Manager.

```bash
# Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
  --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID

# Public route table
PUB_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $PUB_RT \
  --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --route-table-id $PUB_RT --subnet-id $PUB_SUBNET1
aws ec2 associate-route-table --route-table-id $PUB_RT --subnet-id $PUB_SUBNET2

# NAT Gateway (must be in PUBLIC subnet)
EIP=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)
NAT_GW=$(aws ec2 create-nat-gateway \
  --subnet-id $PUB_SUBNET1 --allocation-id $EIP \
  --query 'NatGateway.NatGatewayId' --output text)
echo "⏳ Waiting for NAT Gateway..."
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW

# Private route table → NAT (not IGW)
PRI_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $PRI_RT \
  --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_GW
aws ec2 associate-route-table --route-table-id $PRI_RT --subnet-id $PRI_SUBNET1
aws ec2 associate-route-table --route-table-id $PRI_RT --subnet-id $PRI_SUBNET2

echo "✅ IGW: $IGW_ID | NAT: $NAT_GW"
```

---

## 4. Security Groups

| Security Group | Inbound | Purpose |
|---|---|---|
| `wp-alb-sg` | 80, 443 from `0.0.0.0/0` | Internet-facing ALB |
| `wp-node-sg` | 80, 443 from `wp-alb-sg` | EKS worker nodes / pod ENIs |
| `wp-efs-sg` | 2049 (NFS) from `wp-node-sg` | EFS mount targets |
| `wp-rds-sg` | 3306 from `wp-node-sg` | RDS MySQL |

```bash
# ALB
ALB_SG=$(aws ec2 create-security-group --group-name wp-alb-sg \
  --description 'ALB HTTP/HTTPS' --vpc-id $VPC_ID --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $ALB_SG \
  --ip-permissions \
  '[{"IpProtocol":"tcp","FromPort":80,"ToPort":80,"IpRanges":[{"CidrIp":"0.0.0.0/0"}]},
    {"IpProtocol":"tcp","FromPort":443,"ToPort":443,"IpRanges":[{"CidrIp":"0.0.0.0/0"}]}]'

# EKS Nodes
NODE_SG=$(aws ec2 create-security-group --group-name wp-node-sg \
  --description 'EKS nodes' --vpc-id $VPC_ID --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $NODE_SG \
  --protocol tcp --port 80 --source-group $ALB_SG
aws ec2 authorize-security-group-ingress --group-id $NODE_SG \
  --protocol tcp --port 443 --source-group $ALB_SG

# EFS
EFS_SG=$(aws ec2 create-security-group --group-name wp-efs-sg \
  --description 'EFS NFS' --vpc-id $VPC_ID --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $EFS_SG \
  --protocol tcp --port 2049 --source-group $NODE_SG

# RDS
RDS_SG=$(aws ec2 create-security-group --group-name wp-rds-sg \
  --description 'RDS MySQL' --vpc-id $VPC_ID --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $RDS_SG \
  --protocol tcp --port 3306 --source-group $NODE_SG

echo "✅ ALB: $ALB_SG | Nodes: $NODE_SG | EFS: $EFS_SG | RDS: $RDS_SG"
```

---

## 5. EKS Cluster

### 5.1 Create Cluster with eksctl

```bash
eksctl create cluster \
  --name $CLUSTER_NAME \
  --region $REGION \
  --version 1.30 \
  --vpc-private-subnets $PRI_SUBNET1,$PRI_SUBNET2 \
  --vpc-public-subnets  $PUB_SUBNET1,$PUB_SUBNET2 \
  --nodegroup-name wordpress-nodes \
  --node-type t3.medium \
  --nodes-min 2 \
  --nodes-max 6 \
  --node-private-networking \
  --asg-access \
  --with-oidc \
  --managed

kubectl get nodes
```

> `--with-oidc` provisions the OIDC provider required for IRSA. Without it, pods cannot assume IAM roles.

### 5.2 Configure kubectl

```bash
aws eks update-kubeconfig --region $REGION --name $CLUSTER_NAME
kubectl get svc   # should show default/kubernetes ClusterIP
```

---

## 6. AWS Load Balancer Controller

This Helm chart watches Kubernetes `Ingress` resources and automatically provisions and configures the ALB — replacing all the manual listener and target group CLI steps from the ECS guide.

### 6.1 IAM Policy

```bash
curl -o alb-iam-policy.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://alb-iam-policy.json
```

### 6.2 IRSA for the Controller

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

eksctl create iamserviceaccount \
  --cluster $CLUSTER_NAME \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve
```

### 6.3 Install via Helm

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=$REGION \
  --set vpcId=$VPC_ID

kubectl get deployment -n kube-system aws-load-balancer-controller
```

---

## 7. EFS CSI Driver and Persistent Storage

WordPress stores themes, plugins, and uploads in `/var/www/html/wp-content`. This must be shared across all pods using a `ReadWriteMany` PersistentVolumeClaim backed by Amazon EFS.

### 7.1 Create EFS Filesystem

```bash
EFS_ID=$(aws efs create-file-system \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --encrypted \
  --tags Key=Name,Value=wordpress-eks-efs \
  --region $REGION \
  --query 'FileSystemId' --output text)

# Mount targets in private subnets
aws efs create-mount-target \
  --file-system-id $EFS_ID --subnet-id $PRI_SUBNET1 --security-groups $EFS_SG
aws efs create-mount-target \
  --file-system-id $EFS_ID --subnet-id $PRI_SUBNET2 --security-groups $EFS_SG
echo "⏳ Waiting 30s for mount targets..."
sleep 30

# Access Point (www-data = UID/GID 33)
EFS_AP=$(aws efs create-access-point \
  --file-system-id $EFS_ID \
  --posix-user Uid=33,Gid=33 \
  --root-directory 'Path=/wp-content,CreationInfo={OwnerUid=33,OwnerGid=33,Permissions=755}' \
  --tags Key=Name,Value=wp-content-ap \
  --region $REGION \
  --query 'AccessPointId' --output text)

echo "✅ EFS: $EFS_ID | Access Point: $EFS_AP"
```

### 7.2 Install EFS CSI Driver

```bash
# As EKS managed addon (recommended)
aws eks create-addon \
  --cluster-name $CLUSTER_NAME \
  --addon-name aws-efs-csi-driver \
  --region $REGION
```

### 7.3 StorageClass and PVC

Save as `efs-storageclass.yaml` — replace `<EFS_ID>` before applying:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: <EFS_ID>
  directoryPerms: "755"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-content-pvc
  namespace: wordpress
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 10Gi
```

---

## 8. Amazon RDS MySQL

> ⚠️ **This replaces the MySQL sidecar from the ECS design.** With multiple pod replicas, each pod would otherwise launch its own MySQL instance and corrupt the shared EFS data directory. RDS Multi-AZ is the correct pattern for horizontally scalable WordPress.

```bash
# RDS subnet group
aws rds create-db-subnet-group \
  --db-subnet-group-name wordpress-rds-subnets \
  --db-subnet-group-description 'WordPress RDS subnet group' \
  --subnet-ids $PRI_SUBNET1 $PRI_SUBNET2

# Create Multi-AZ RDS instance
aws rds create-db-instance \
  --db-instance-identifier wordpress-db \
  --db-instance-class db.t3.medium \
  --engine mysql \
  --engine-version 8.0 \
  --master-username dbadmin \
  --master-user-password 'YourStrongPassword123!' \
  --db-name wordpress \
  --db-subnet-group-name wordpress-rds-subnets \
  --vpc-security-group-ids $RDS_SG \
  --multi-az \
  --storage-type gp3 \
  --allocated-storage 20 \
  --storage-encrypted \
  --backup-retention-period 7 \
  --no-publicly-accessible \
  --region $REGION

# Wait (~10 min) then capture endpoint
echo "⏳ Waiting for RDS..."
aws rds wait db-instance-available \
  --db-instance-identifier wordpress-db --region $REGION

RDS_HOST=$(aws rds describe-db-instances \
  --db-instance-identifier wordpress-db \
  --query 'DBInstances[0].Endpoint.Address' --output text)
echo "✅ RDS endpoint: $RDS_HOST"
```

---

## 9. Secrets Manager and ExternalSecrets Operator

### 9.1 Store Credentials

```bash
aws secretsmanager create-secret \
  --name wordpress/config \
  --region $REGION \
  --secret-string "{
    \"MYSQL_HOST\": \"$RDS_HOST\",
    \"MYSQL_DATABASE\": \"wordpress\",
    \"MYSQL_USER\": \"dbadmin\",
    \"MYSQL_PASSWORD\": \"YourStrongPassword123!\",
    \"WP_TABLE_PREFIX\": \"wp_\",
    \"WP_DEBUG\": \"false\",
    \"DOMAIN\": \"your-domain.com\"
  }"
```

### 9.2 IRSA for WordPress Pods

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

cat > wordpress-pod-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue", "kms:Decrypt"],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "elasticfilesystem:ClientMount",
        "elasticfilesystem:ClientWrite"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name WordpressPodPolicy \
  --policy-document file://wordpress-pod-policy.json

eksctl create iamserviceaccount \
  --cluster $CLUSTER_NAME \
  --namespace wordpress \
  --name wordpress-sa \
  --attach-policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/WordpressPodPolicy \
  --approve
```

### 9.3 Install ExternalSecrets Operator

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace
```

### 9.4 SecretStore and ExternalSecret Manifests

```yaml
# secretstore.yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secretsmanager
  namespace: wordpress
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-south-2
      auth:
        jwt:
          serviceAccountRef:
            name: wordpress-sa
---
# externalsecret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: wordpress-config
  namespace: wordpress
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore
  target:
    name: wordpress-secret
  data:
    - secretKey: MYSQL_HOST
      remoteRef: { key: wordpress/config, property: MYSQL_HOST }
    - secretKey: MYSQL_DATABASE
      remoteRef: { key: wordpress/config, property: MYSQL_DATABASE }
    - secretKey: MYSQL_USER
      remoteRef: { key: wordpress/config, property: MYSQL_USER }
    - secretKey: MYSQL_PASSWORD
      remoteRef: { key: wordpress/config, property: MYSQL_PASSWORD }
    - secretKey: DOMAIN
      remoteRef: { key: wordpress/config, property: DOMAIN }
```

---

## 10. ECR — Container Registry

The WordPress image and `wp-config.php` fix are identical to the ECS guide.

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ECR_URI="$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/docker-wordpress"

aws ecr create-repository \
  --repository-name docker-wordpress \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=AES256

aws ecr get-login-password --region $REGION | \
  docker login --username AWS --password-stdin \
  "$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com"

docker build -t docker-wordpress .
docker tag docker-wordpress:latest ${ECR_URI}:latest
docker push ${ECR_URI}:latest

echo "✅ Image pushed: ${ECR_URI}:latest"
```

> ⚠️ The `wp-config.php` fix is still required. Add this **before** `require_once ABSPATH . 'wp-settings.php'`:
>
> ```php
> if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
>     $_SERVER['HTTPS'] = 'on';
> }
> define('FORCE_SSL_ADMIN', true);
> ```
>
> The ALB terminates HTTPS and forwards plain HTTP to pods — same as ECS.

---

## 11. Kubernetes Manifests

Create a namespace first:

```bash
kubectl create namespace wordpress
```

### 11.1 Deployment

Save as `wordpress-deployment.yaml` — replace `<ACCOUNT_ID>`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: wordpress
spec:
  replicas: 2
  selector:
    matchLabels:
      app: wordpress
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      serviceAccountName: wordpress-sa    # IRSA
      containers:
        - name: wordpress
          image: <ACCOUNT_ID>.dkr.ecr.ap-south-2.amazonaws.com/docker-wordpress:latest
          ports:
            - containerPort: 80
          envFrom:
            - secretRef:
                name: wordpress-secret    # synced by ExternalSecrets
          volumeMounts:
            - name: wp-content
              mountPath: /var/www/html/wp-content
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 1Gi
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 60
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 30
            periodSeconds: 10
      volumes:
        - name: wp-content
          persistentVolumeClaim:
            claimName: wp-content-pvc
```

### 11.2 Service

```yaml
# wordpress-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: wordpress
spec:
  type: ClusterIP
  selector:
    app: wordpress
  ports:
    - port: 80
      targetPort: 80
```

### 11.3 Ingress (ALB)

Replace `<ACM_CERT_ARN>` and `<ALB_SG_ID>` before applying:

```yaml
# wordpress-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wordpress
  namespace: wordpress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: <ACM_CERT_ARN>
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/security-groups: <ALB_SG_ID>
    alb.ingress.kubernetes.io/healthcheck-path: /
    alb.ingress.kubernetes.io/success-codes: 200-399
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '30'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/target-group-attributes: deregistration_delay.timeout_seconds=30
spec:
  rules:
    - host: your-domain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: wordpress
                port:
                  number: 80
```

### 11.4 HorizontalPodAutoscaler

```yaml
# wordpress-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: wordpress-hpa
  namespace: wordpress
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: wordpress
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

---

## 12. ACM Certificate and DNS

```bash
CERT_ARN=$(aws acm request-certificate \
  --domain-name "your-domain.com" \
  --subject-alternative-names "www.your-domain.com" \
  --validation-method DNS \
  --region $REGION \
  --query 'CertificateArn' --output text)

echo "✅ Certificate ARN: $CERT_ARN"

# Get DNS validation CNAMEs and add them to your DNS provider
aws acm describe-certificate \
  --certificate-arn $CERT_ARN --region $REGION \
  --query 'Certificate.DomainValidationOptions[].ResourceRecord'

aws acm wait certificate-validated --certificate-arn $CERT_ARN --region $REGION
echo "✅ Certificate ISSUED"

# After applying the Ingress manifest, get the ALB hostname:
kubectl get ingress -n wordpress wordpress \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
# Point your domain CNAME (or Route 53 Alias A record) to this hostname.
```

| Type | Name | Value |
|------|------|-------|
| `CNAME` | `@` | ALB hostname from above |
| `CNAME` | `www` | ALB hostname from above |

> Route 53 users: use an `A` record with **Alias → Application Load Balancer** for the apex domain.

---

## 13. Logging with Fluent Bit

Replaces the `awslogs` driver from ECS. Fluent Bit runs as a DaemonSet and ships container logs to CloudWatch Container Insights.

```bash
ClusterName=$CLUSTER_NAME
RegionName=$REGION
FluentBitHttpPort='2020'
FluentBitReadFromHead='Off'

curl -s https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml \
  | sed "s/{{cluster_name}}/${ClusterName}/;s/{{region_name}}/${RegionName}/;s/{{http_server_toggle}}/${FluentBitHttpPort}/;s/{{read_from_head}}/${FluentBitReadFromHead}/" \
  | kubectl apply -f -

# Logs appear in CloudWatch under:
# /aws/containerinsights/$CLUSTER_NAME/application
```

---

## 14. Apply Manifests and Verify

```bash
# Apply in order
kubectl apply -f efs-storageclass.yaml
kubectl apply -f wp-content-pvc.yaml
kubectl apply -f secretstore.yaml
kubectl apply -f externalsecret.yaml
kubectl apply -f wordpress-deployment.yaml
kubectl apply -f wordpress-service.yaml
kubectl apply -f wordpress-ingress.yaml
kubectl apply -f wordpress-hpa.yaml

# Watch rollout
kubectl rollout status deployment/wordpress -n wordpress

# Confirm pods running
kubectl get pods -n wordpress -o wide

# Check Ingress / ALB provisioning (~2 min)
kubectl get ingress -n wordpress

# Tail logs
kubectl logs -n wordpress -l app=wordpress --follow
```

### Post-Deployment

1. Visit `https://your-domain.com` — the WordPress installer will appear
2. Fill in site title, admin username, password, and email
3. Log in at `https://your-domain.com/wp-admin`
4. Go to **Settings → General** — both WordPress Address and Site Address must show `https://`

---

## 15. Useful Commands

```bash
# Exec into a running pod
kubectl exec -it -n wordpress deploy/wordpress -- /bin/bash

# Describe a pod (events, resource usage)
kubectl describe pod -n wordpress <pod-name>

# Force redeployment (pulls latest image)
kubectl rollout restart deployment/wordpress -n wordpress

# Scale manually
kubectl scale deployment/wordpress -n wordpress --replicas=4

# View HPA status
kubectl get hpa -n wordpress

# Port-forward to localhost for debugging
kubectl port-forward -n wordpress svc/wordpress 8080:80

# Check EFS PVC binding
kubectl get pvc -n wordpress

# View namespace events sorted by time
kubectl get events -n wordpress --sort-by=.lastTimestamp

# Tail CloudWatch logs
aws logs tail /aws/containerinsights/$CLUSTER_NAME/application --follow --region $REGION

# Check ExternalSecret sync status
kubectl get externalsecret -n wordpress

# Verify IRSA annotation on service account
kubectl describe sa wordpress-sa -n wordpress
```

---

## 16. Security Checklist

- [ ] EKS private API endpoint enabled; nodes in private subnets
- [ ] Node SG allows only 80/443 from ALB SG — not from `0.0.0.0/0`
- [ ] RDS in private subnets; `no-publicly-accessible`; SG allows only 3306 from node SG
- [ ] Secrets in Secrets Manager; delivered via ExternalSecrets; never hardcoded in manifests
- [ ] IRSA scopes pod permissions — no broad EC2 node instance role for secret access
- [ ] EFS encrypted at rest; transit encryption enabled on CSI driver
- [ ] ACM certificate covers both `domain.com` and `www.domain.com`
- [ ] ALB HTTP → HTTPS redirect active; TLS 1.3 policy
- [ ] `wp-config.php` includes `HTTP_X_FORWARDED_PROTO` fix — WordPress generates `https://` links
- [ ] WordPress Address and Site Address both set to `https://your-domain.com`
- [ ] ECR scan-on-push enabled; use git SHA tags in production, not `latest`
- [ ] Fluent Bit log group retention set to 30 days
- [ ] HPA minReplicas ≥ 2 for high availability
- [ ] RDS automated backups enabled with ≥ 7-day retention

---

## 17. Cleanup

```bash
# Delete Kubernetes resources (triggers ALB deprovisioning)
kubectl delete namespace wordpress

# Delete EKS cluster and node groups (~10 min)
eksctl delete cluster --name $CLUSTER_NAME --region $REGION

# Delete RDS
aws rds delete-db-instance \
  --db-instance-identifier wordpress-db \
  --skip-final-snapshot \
  --region $REGION

# Delete EFS
for mt in $(aws efs describe-mount-targets \
  --file-system-id $EFS_ID \
  --query 'MountTargets[].MountTargetId' --output text); do
  aws efs delete-mount-target --mount-target-id $mt
done
sleep 30
aws efs delete-file-system --file-system-id $EFS_ID

# Delete ECR
aws ecr delete-repository --repository-name docker-wordpress --force

# Delete ACM certificate
aws acm delete-certificate --certificate-arn $CERT_ARN

# Delete networking (NAT Gateway takes ~2 min)
aws ec2 delete-nat-gateway --nat-gateway-id $NAT_GW
echo "⏳ Waiting for NAT Gateway deletion..."
sleep 120
aws ec2 release-address --allocation-id $EIP

for sg in $ALB_SG $NODE_SG $EFS_SG $RDS_SG; do
  aws ec2 delete-security-group --group-id $sg
done

aws ec2 detach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID

for rt in $PUB_RT $PRI_RT; do aws ec2 delete-route-table --route-table-id $rt; done

for s in $PUB_SUBNET1 $PUB_SUBNET2 $PRI_SUBNET1 $PRI_SUBNET2; do
  aws ec2 delete-subnet --subnet-id $s
done

aws ec2 delete-vpc --vpc-id $VPC_ID

echo "🧹 All resources deleted."
```

---
