# 🚀 Manual AWS EKS Cluster Setup — From Scratch

This guide documents how to create an **Amazon EKS cluster manually from scratch**, without using `eksctl`.

The goal is to understand the complete EKS infrastructure:

```text
AWS Account
   │
   ├── VPC
   │    ├── Internet Gateway
   │    ├── Public Subnets
   │    │    ├── NAT Gateway
   │    │    ├── NAT Gateway
   │    │    └── NAT Gateway
   │    │
   │    └── Private Subnets
   │         ├── EKS Worker Node
   │         ├── EKS Worker Node
   │         └── EKS Worker Node
   │
   └── EKS Control Plane
            │
            └── Kubernetes Worker Nodes
```

---

# 1. Architecture

## Region

```text
ap-south-1
```

## Cluster

```text
Cluster Name: prod-eks
Kubernetes Version: 1.36
```

## VPC

```text
VPC CIDR: 10.0.0.0/16
```

## Availability Zones

```text
ap-south-1a
ap-south-1b
ap-south-1c
```

## Public Subnets

```text
10.0.1.0/24
10.0.2.0/24
10.0.3.0/24
```

## Private Subnets

```text
10.0.11.0/24
10.0.12.0/24
10.0.13.0/24
```

## Worker Nodes

```text
3 x EC2 instances

Private Subnet 1
10.0.11.0/24

Private Subnet 2
10.0.12.0/24

Private Subnet 3
10.0.13.0/24
```

---

# 2. Prerequisites

Run these commands from an Ubuntu management EC2 instance.

Required tools:

```bash
aws
kubectl
```

Verify:

```bash
aws --version
kubectl version --client
```

Configure AWS:

```bash
aws configure
```

Enter:

```text
AWS Access Key ID
AWS Secret Access Key
Default region: ap-south-1
Output format: json
```

Verify:

```bash
aws sts get-caller-identity
```

---

# 3. Configure Environment Variables

Create a persistent environment file.

```bash
cat > ~/eks-env.sh <<'EOF'
export AWS_REGION=ap-south-1
export CLUSTER_NAME=prod-eks
export VPC_CIDR=10.0.0.0/16

export PUBLIC_SUBNET_1=10.0.1.0/24
export PUBLIC_SUBNET_2=10.0.2.0/24
export PUBLIC_SUBNET_3=10.0.3.0/24

export PRIVATE_SUBNET_1=10.0.11.0/24
export PRIVATE_SUBNET_2=10.0.12.0/24
export PRIVATE_SUBNET_3=10.0.13.0/24
EOF
```

Load it:

```bash
source ~/eks-env.sh
```

Verify:

```bash
echo $AWS_REGION
echo $CLUSTER_NAME
```

Expected:

```text
ap-south-1
prod-eks
```

### Important

Do not rely only on:

```bash
export CLUSTER_NAME=prod-eks
```

because shell variables disappear when you disconnect from SSH.

This caused one of our issues where:

```bash
$CLUSTER_NAME
```

became empty.

---

# 4. Create VPC

Create the VPC:

```bash
export VPC_ID=$(aws ec2 create-vpc \
  --cidr-block $VPC_CIDR \
  --region $AWS_REGION \
  --query 'Vpc.VpcId' \
  --output text)
```

Enable DNS:

```bash
aws ec2 modify-vpc-attribute \
  --vpc-id $VPC_ID \
  --enable-dns-support "{\"Value\":true}"

aws ec2 modify-vpc-attribute \
  --vpc-id $VPC_ID \
  --enable-dns-hostnames "{\"Value\":true}"
```

Tag:

```bash
aws ec2 create-tags \
  --resources $VPC_ID \
  --tags Key=Name,Value=prod-eks-vpc
```

Verify:

```bash
echo $VPC_ID
```

---

# 5. Create Internet Gateway

Create:

```bash
export IGW_ID=$(aws ec2 create-internet-gateway \
  --query 'InternetGateway.InternetGatewayId' \
  --output text)
```

Attach:

```bash
aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $VPC_ID
```

Verify:

```bash
aws ec2 describe-internet-gateways \
  --internet-gateway-ids $IGW_ID
```

---

# 6. Create Public Subnets

Create three public subnets.

### AZ 1

```bash
export PUBLIC_SUBNET_ID_1=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 \
  --availability-zone ap-south-1a \
  --query 'Subnet.SubnetId' \
  --output text)
```

### AZ 2

```bash
export PUBLIC_SUBNET_ID_2=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.2.0/24 \
  --availability-zone ap-south-1b \
  --query 'Subnet.SubnetId' \
  --output text)
```

### AZ 3

```bash
export PUBLIC_SUBNET_ID_3=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.3.0/24 \
  --availability-zone ap-south-1c \
  --query 'Subnet.SubnetId' \
  --output text)
```

Enable public IP assignment:

```bash
aws ec2 modify-subnet-attribute \
  --subnet-id $PUBLIC_SUBNET_ID_1 \
  --map-public-ip-on-launch

aws ec2 modify-subnet-attribute \
  --subnet-id $PUBLIC_SUBNET_ID_2 \
  --map-public-ip-on-launch

aws ec2 modify-subnet-attribute \
  --subnet-id $PUBLIC_SUBNET_ID_3 \
  --map-public-ip-on-launch
```

---

# 7. Create Private Subnets

### AZ 1

```bash
export PRIVATE_SUBNET_ID_1=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.11.0/24 \
  --availability-zone ap-south-1a \
  --query 'Subnet.SubnetId' \
  --output text)
```

### AZ 2

```bash
export PRIVATE_SUBNET_ID_2=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.12.0/24 \
  --availability-zone ap-south-1b \
  --query 'Subnet.SubnetId' \
  --output text)
```

### AZ 3

```bash
export PRIVATE_SUBNET_ID_3=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.13.0/24 \
  --availability-zone ap-south-1c \
  --query 'Subnet.SubnetId' \
  --output text)
```

---

# 8. Create Public Route Table

Create:

```bash
export PUBLIC_RT_ID=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' \
  --output text)
```

Add Internet route:

```bash
aws ec2 create-route \
  --route-table-id $PUBLIC_RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID
```

Associate all public subnets:

```bash
aws ec2 associate-route-table \
  --route-table-id $PUBLIC_RT_ID \
  --subnet-id $PUBLIC_SUBNET_ID_1

aws ec2 associate-route-table \
  --route-table-id $PUBLIC_RT_ID \
  --subnet-id $PUBLIC_SUBNET_ID_2

aws ec2 associate-route-table \
  --route-table-id $PUBLIC_RT_ID \
  --subnet-id $PUBLIC_SUBNET_ID_3
```

---

# 9. Create NAT Gateways

For production-style AZ resilience, use one NAT Gateway per AZ.

First allocate Elastic IPs:

```bash
export EIP_1=$(aws ec2 allocate-address \
  --domain vpc \
  --query 'AllocationId' \
  --output text)

export EIP_2=$(aws ec2 allocate-address \
  --domain vpc \
  --query 'AllocationId' \
  --output text)

export EIP_3=$(aws ec2 allocate-address \
  --domain vpc \
  --query 'AllocationId' \
  --output text)
```

Create NAT Gateway 1:

```bash
export NAT_GW_1=$(aws ec2 create-nat-gateway \
  --subnet-id $PUBLIC_SUBNET_ID_1 \
  --allocation-id $EIP_1 \
  --query 'NatGateway.NatGatewayId' \
  --output text)
```

NAT Gateway 2:

```bash
export NAT_GW_2=$(aws ec2 create-nat-gateway \
  --subnet-id $PUBLIC_SUBNET_ID_2 \
  --allocation-id $EIP_2 \
  --query 'NatGateway.NatGatewayId' \
  --output text)
```

NAT Gateway 3:

```bash
export NAT_GW_3=$(aws ec2 create-nat-gateway \
  --subnet-id $PUBLIC_SUBNET_ID_3 \
  --allocation-id $EIP_3 \
  --query 'NatGateway.NatGatewayId' \
  --output text)
```

Wait until available:

```bash
aws ec2 wait nat-gateway-available \
  --nat-gateway-ids $NAT_GW_1 $NAT_GW_2 $NAT_GW_3
```

---

# 10. Create Private Route Tables

Create three:

```bash
export PRIVATE_RT_1=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' \
  --output text)

export PRIVATE_RT_2=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' \
  --output text)

export PRIVATE_RT_3=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' \
  --output text)
```

Routes:

```bash
aws ec2 create-route \
  --route-table-id $PRIVATE_RT_1 \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NAT_GW_1

aws ec2 create-route \
  --route-table-id $PRIVATE_RT_2 \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NAT_GW_2

aws ec2 create-route \
  --route-table-id $PRIVATE_RT_3 \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NAT_GW_3
```

Associate:

```bash
aws ec2 associate-route-table \
  --route-table-id $PRIVATE_RT_1 \
  --subnet-id $PRIVATE_SUBNET_ID_1

aws ec2 associate-route-table \
  --route-table-id $PRIVATE_RT_2 \
  --subnet-id $PRIVATE_SUBNET_ID_2

aws ec2 associate-route-table \
  --route-table-id $PRIVATE_RT_3 \
  --subnet-id $PRIVATE_SUBNET_ID_3
```

---

# 11. Create EKS Cluster IAM Role

Create trust policy:

```bash
cat > eks-cluster-trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```

Create role:

```bash
aws iam create-role \
  --role-name prod-eks-cluster-role \
  --assume-role-policy-document file://eks-cluster-trust.json
```

Attach:

```bash
aws iam attach-role-policy \
  --role-name prod-eks-cluster-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
```

Get ARN:

```bash
export CLUSTER_ROLE_ARN=$(aws iam get-role \
  --role-name prod-eks-cluster-role \
  --query 'Role.Arn' \
  --output text)
```

---

# 12. Create EKS Cluster

Create the cluster:

```bash
aws eks create-cluster \
  --name $CLUSTER_NAME \
  --version 1.36 \
  --role-arn $CLUSTER_ROLE_ARN \
  --resources-vpc-config \
    subnetIds=$PRIVATE_SUBNET_ID_1,$PRIVATE_SUBNET_ID_2,$PRIVATE_SUBNET_ID_3 \
    ,endpointPublicAccess=true \
    ,endpointPrivateAccess=true
```

Wait:

```bash
aws eks wait cluster-active \
  --name $CLUSTER_NAME
```

Check:

```bash
aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --query 'cluster.status'
```

Expected:

```text
ACTIVE
```

---

# 13. Configure kubectl

Run:

```bash
aws eks update-kubeconfig \
  --region $AWS_REGION \
  --name $CLUSTER_NAME
```

Verify:

```bash
kubectl get nodes
```

At this stage there may be no worker nodes yet.

Check cluster:

```bash
kubectl cluster-info
```

---

# 14. Create Worker Node IAM Role

Create trust policy:

```bash
cat > node-trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```

Create:

```bash
aws iam create-role \
  --role-name prod-eks-node-role \
  --assume-role-policy-document file://node-trust.json
```

Attach worker-node policy:

```bash
aws iam attach-role-policy \
  --role-name prod-eks-node-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
```

Attach ECR pull permission:

```bash
aws iam attach-role-policy \
  --role-name prod-eks-node-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPullOnly
```

Get ARN:

```bash
export NODE_ROLE_ARN=$(aws iam get-role \
  --role-name prod-eks-node-role \
  --query 'Role.Arn' \
  --output text)
```

Verify:

```bash
echo $NODE_ROLE_ARN
```

---

# 15. Create Managed Node Group

Before creating the node group, make sure subnet variables exist.

Check:

```bash
echo $PRIVATE_SUBNET_ID_1
echo $PRIVATE_SUBNET_ID_2
echo $PRIVATE_SUBNET_ID_3
echo $NODE_ROLE_ARN
```

They must not be empty.

Create node group:

```bash
aws eks create-nodegroup \
  --cluster-name $CLUSTER_NAME \
  --nodegroup-name prod-general-ng \
  --node-role $NODE_ROLE_ARN \
  --subnets \
    $PRIVATE_SUBNET_ID_1 \
    $PRIVATE_SUBNET_ID_2 \
    $PRIVATE_SUBNET_ID_3 \
  --instance-types m7i-flex.large \
  --ami-type AL2023_x86_64_STANDARD \
  --disk-size 80 \
  --capacity-type ON_DEMAND \
  --scaling-config minSize=3,maxSize=6,desiredSize=3 \
  --region $AWS_REGION
```

Check:

```bash
aws eks describe-nodegroup \
  --cluster-name $CLUSTER_NAME \
  --nodegroup-name prod-general-ng \
  --query 'nodegroup.status'
```

Wait:

```bash
aws eks wait nodegroup-active \
  --cluster-name $CLUSTER_NAME \
  --nodegroup-name prod-general-ng
```

---

# 16. Verify Worker Nodes

Run:

```bash
kubectl get nodes -o wide
```

Expected:

```text
NAME                                         STATUS   ROLES    AGE   VERSION
ip-10-0-11-xxx.ap-south-1.compute.internal   Ready    <none>   ...   v1.36.x
ip-10-0-12-xxx.ap-south-1.compute.internal   Ready    <none>   ...   v1.36.x
ip-10-0-13-xxx.ap-south-1.compute.internal   Ready    <none>   ...   v1.36.x
```

---

# 17. EKS System Components

Check:

```bash
kubectl get pods -n kube-system -o wide
```

Typical components include:

```text
aws-node
kube-proxy
coredns
```

Check:

```bash
kubectl get daemonsets -n kube-system
```

---

# 18. Issue Faced #1 — Environment Variables Were Empty

### Problem

The command:

```bash
aws eks create-nodegroup \
  --subnets $PRIVATE_SUBNET_1 $PRIVATE_SUBNET_2 $PRIVATE_SUBNET_3
```

returned:

```text
InvalidParameterException: Subnets are required
```

### Cause

The variables were empty:

```bash
echo $PRIVATE_SUBNET_1
```

returned nothing.

The shell environment had been lost after reconnecting to the server.

### Fix

Recover the subnet IDs from AWS:

```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone]' \
  --output table
```

Then set:

```bash
export PRIVATE_SUBNET_ID_1=subnet-xxxxxxxx
export PRIVATE_SUBNET_ID_2=subnet-yyyyyyyy
export PRIVATE_SUBNET_ID_3=subnet-zzzzzzzz
```

Always verify:

```bash
echo $PRIVATE_SUBNET_ID_1
echo $PRIVATE_SUBNET_ID_2
echo $PRIVATE_SUBNET_ID_3
```

### Prevention

Store variables in:

```text
~/eks-env.sh
```

and run:

```bash
source ~/eks-env.sh
```

after reconnecting.

---

# 19. Issue Faced #2 — Node Group Creation Failed

### Error

The first node group attempt failed with:

```text
AsgInstanceLaunchFailures
Could not launch On-Demand Instances.

InvalidParameterCombination:
The specified instance type is not eligible for Free Tier.
```

### Cause

The selected EC2 instance type was not eligible for the account's applicable Free Tier.

### Important

Free Tier eligibility depends on the AWS account, instance type, region, and current AWS Free Tier rules.

Do not assume that every `*.large` instance is Free Tier eligible.

### Troubleshooting

Check the node group:

```bash
aws eks describe-nodegroup \
  --cluster-name $CLUSTER_NAME \
  --nodegroup-name prod-general-ng \
  --query 'nodegroup.health.issues'
```

Check EC2 instances:

```bash
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[InstanceId,InstanceType,State.Name]' \
  --output table
```

### Lesson

For a production-style architecture:

```text
3 nodes
```

is reasonable as a starting point, but it is not necessarily cheap.

Always verify AWS pricing and Free Tier eligibility before creating large instances.

---

# 20. Issue Faced #3 — Nodes Became NotReady

After the node group successfully launched, the nodes showed:

```text
STATUS
NotReady
```

Example:

```text
ip-10-0-11-163.ap-south-1.compute.internal   NotReady
ip-10-0-12-128.ap-south-1.compute.internal   NotReady
ip-10-0-13-186.ap-south-1.compute.internal   NotReady
```

Checking the node:

```bash
kubectl describe node <NODE_NAME>
```

showed:

```text
Ready False

Reason:
KubeletNotReady

Message:
container runtime network not ready:
NetworkReady=false
reason:NetworkPluginNotReady
message:Network plugin returns error:
cni plugin not initialized
```

### Meaning

The EC2 instances were running, and kubelet was running, but Kubernetes networking had not initialized.

---

# 21. Issue Faced #4 — AWS VPC CNI CrashLoopBackOff

Check:

```bash
kubectl get pods -n kube-system \
  -l k8s-app=aws-node \
  -o wide
```

The result was:

```text
aws-node-xxxxx   1/2   CrashLoopBackOff
aws-node-yyyyy   1/2   CrashLoopBackOff
aws-node-zzzzz   1/2   CrashLoopBackOff
```

Check logs:

```bash
kubectl logs \
  -n kube-system \
  aws-node-xxxxx \
  -c aws-node \
  --previous \
  --tail=200
```

The container reached:

```text
Starting IPAM daemon...
Checking for IPAM connectivity...
```

but could not initialize.

---

# 22. Root Cause — Missing IAM Permission

The most important diagnostic command was:

```bash
kubectl describe pod \
  -n kube-system \
  aws-node-xxxxx
```

The Events section showed:

```text
MissingIAMPermissions

Unauthorized operation:
failed to call ec2:DescribeNetworkInterfaces
due to missing permissions
```

### Root Cause

The AWS VPC CNI needed EC2 permissions to manage and discover network interfaces.

The IAM role being used by the CNI did not have the required permission.

This caused:

```text
aws-node
    ↓
IPAM initialization
    ↓
AWS EC2 API call
    ↓
ec2:DescribeNetworkInterfaces
    ↓
AccessDenied
    ↓
aws-node crashes
    ↓
CNI unavailable
    ↓
kubelet reports NetworkPluginNotReady
    ↓
Node = NotReady
```

---

# 23. Fix — VPC CNI IAM Permission

First check the node role:

```bash
aws iam list-attached-role-policies \
  --role-name prod-eks-node-role
```

For a simple lab configuration, attach:

```bash
aws iam attach-role-policy \
  --role-name prod-eks-node-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
```

Verify:

```bash
aws iam list-attached-role-policies \
  --role-name prod-eks-node-role
```

Then restart the CNI:

```bash
kubectl rollout restart daemonset aws-node \
  -n kube-system
```

Monitor:

```bash
kubectl get pods \
  -n kube-system \
  -l k8s-app=aws-node \
  -o wide
```

Expected:

```text
READY   STATUS
2/2     Running
2/2     Running
2/2     Running
```

Then:

```bash
kubectl get nodes
```

Expected:

```text
Ready
Ready
Ready
```

---

# 24. Production IAM Improvement

The above solution is useful for understanding and quickly fixing a lab.

However, for a production environment, avoid putting every Kubernetes workload permission on the EC2 node role.

A better design is:

```text
EC2 Node Role
    │
    ├── Worker node permissions
    └── ECR permissions

aws-node Service Account
    │
    └── Dedicated IAM Role
          │
          └── VPC CNI permissions
```

Use:

```text
EKS Pod Identity
```

or an appropriately configured IAM role for service accounts.

This follows the principle of least privilege more closely.

---

# 25. Verify VPC CNI

After fixing IAM:

```bash
kubectl get daemonset aws-node \
  -n kube-system
```

Expected:

```text
DESIRED   CURRENT   READY
3         3         3
```

Check:

```bash
kubectl get pods \
  -n kube-system \
  -l k8s-app=aws-node
```

Expected:

```text
2/2 Running
2/2 Running
2/2 Running
```

---

# 26. Verify Kubernetes Nodes

```bash
kubectl get nodes -o wide
```

Expected:

```text
STATUS
Ready
Ready
Ready
```

Check conditions:

```bash
kubectl describe node <NODE_NAME> | grep -A10 Conditions
```

Expected:

```text
MemoryPressure   False
DiskPressure     False
PIDPressure      False
Ready             True
```

---

# 27. Verify CoreDNS

```bash
kubectl get pods \
  -n kube-system \
  -l k8s-app=kube-dns
```

Expected:

```text
Running
Running
```

---

# 28. Verify kube-proxy

```bash
kubectl get pods \
  -n kube-system \
  -l k8s-app=kube-proxy \
  -o wide
```

All nodes should have a kube-proxy pod.

---

# 29. Basic Kubernetes Test

Create a test deployment:

```bash
kubectl create deployment nginx \
  --image=nginx
```

Check:

```bash
kubectl get pods -o wide
```

Expose:

```bash
kubectl expose deployment nginx \
  --port=80 \
  --type=ClusterIP
```

Check:

```bash
kubectl get svc
```

Test:

```bash
kubectl run test-pod \
  --image=busybox:1.36 \
  --rm -it \
  --restart=Never \
  -- wget -qO- nginx
```

Expected:

```text
Welcome to nginx!
```

Delete test resources:

```bash
kubectl delete deployment nginx
kubectl delete service nginx
```

---

# 30. Current EKS Foundation

At this stage the architecture is:

```text
                    Internet
                       │
                       ▼
                Internet Gateway
                       │
             ┌─────────┴─────────┐
             │                   │
       Public Subnets       Public Subnets
             │                   │
        NAT Gateway          NAT Gateway
             │                   │
             ▼                   ▼
       Private Subnet       Private Subnet
             │                   │
          EKS Node             EKS Node
             │
             └──────────────┐
                            │
                       EKS Cluster
                            │
                       EKS Node 3
```

---

# 31. Recommended Next Components

Once the basic cluster is healthy, build the platform in this order.

## Networking

```text
VPC CNI
AWS Load Balancer Controller
```

## Storage

```text
EBS CSI Driver
```

## DNS / TLS

```text
Route53
ExternalDNS
ACM
```

## GitOps

```text
ArgoCD
```

## CI/CD

```text
GitLab/GitHub
       ↓
Jenkins
       ↓
Docker Build
       ↓
Amazon ECR
       ↓
GitOps Repository
       ↓
ArgoCD
       ↓
EKS
```

## Application

```text
Node.js Application
       ↓
Deployment
       ↓
Service
       ↓
AWS Load Balancer
       ↓
Route53 DNS
       ↓
HTTPS / ACM
```

## Monitoring

```text
Prometheus
Grafana
CloudWatch
OpenTelemetry
```

## Autoscaling

```text
HPA
Karpenter
Cluster Autoscaling
```

## Security

```text
IAM
EKS Pod Identity
Secrets Manager
KMS
Network Policies
Pod Security
RBAC
```

## Reliability

```text
PDB
Topology Spread Constraints
Multi-AZ
Backups
Velero
Disaster Recovery
```

---

# 32. Troubleshooting Commands

## Cluster

```bash
aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --query 'cluster.status'
```

## Node Group

```bash
aws eks describe-nodegroup \
  --cluster-name $CLUSTER_NAME \
  --nodegroup-name prod-general-ng \
  --query 'nodegroup.{status:status,health:health}'
```

## Nodes

```bash
kubectl get nodes -o wide
```

## Node Details

```bash
kubectl describe node <NODE_NAME>
```

## System Pods

```bash
kubectl get pods \
  -n kube-system \
  -o wide
```

## CNI

```bash
kubectl get pods \
  -n kube-system \
  -l k8s-app=aws-node
```

## CNI Logs

```bash
kubectl logs \
  -n kube-system \
  <AWS_NODE_POD> \
  -c aws-node \
  --previous \
  --tail=200
```

## CNI Events

```bash
kubectl describe pod \
  -n kube-system \
  <AWS_NODE_POD>
```

## IAM Policies

```bash
aws iam list-attached-role-policies \
  --role-name prod-eks-node-role
```

## EC2 Nodes

```bash
aws ec2 describe-instances \
  --filters \
  "Name=tag:eks:nodegroup-name,Values=prod-general-ng" \
  --query \
  'Reservations[*].Instances[*].[InstanceId,PrivateIpAddress,SubnetId,InstanceType,State.Name]' \
  --output table
```

---

# 33. Common Failure Checklist

When a node is `NotReady`, check in this order:

```text
1. Is EC2 instance running?
        ↓
2. Is node registered with EKS?
        ↓
3. kubectl describe node
        ↓
4. Check Ready condition
        ↓
5. Check aws-node
        ↓
6. Check aws-node logs
        ↓
7. Check aws-node events
        ↓
8. Check IAM permissions
        ↓
9. Check subnet routing/NAT
        ↓
10. Check EKS add-ons
```

---

# 34. Important Lessons Learned

## Lesson 1 — Shell variables are temporary

This:

```bash
export CLUSTER_NAME=prod-eks
```

only applies to the current shell.

Use an environment file for persistence.

---

## Lesson 2 — Never run commands with empty variables

Before:

```bash
aws eks create-nodegroup
```

verify:

```bash
echo $CLUSTER_NAME
echo $NODE_ROLE_ARN
echo $PRIVATE_SUBNET_ID_1
echo $PRIVATE_SUBNET_ID_2
echo $PRIVATE_SUBNET_ID_3
```

If any are empty, stop and fix them.

---

## Lesson 3 — EC2 Running doesn't mean Kubernetes Ready

An EC2 instance can be:

```text
EC2 = Running
```

while:

```text
Kubernetes = NotReady
```

because Kubernetes networking, kubelet, container runtime, or CNI may still be unhealthy.

---

## Lesson 4 — Always check Events

For Kubernetes troubleshooting:

```bash
kubectl describe pod
```

and:

```bash
kubectl describe node
```

are extremely useful.

In this project, the actual root cause was clearly exposed in the Events section:

```text
MissingIAMPermissions
```

---

## Lesson 5 — CNI is critical

Without the AWS VPC CNI:

```text
Pod networking
      ↓
Not initialized
      ↓
Node networking unavailable
      ↓
Node NotReady
```

---

## Lesson 6 — IAM is part of Kubernetes networking on EKS

EKS networking is not only Kubernetes configuration.

The AWS VPC CNI interacts with AWS APIs and therefore needs appropriate IAM permissions.

---

## Lesson 7 — Don't randomly rebuild infrastructure

When the nodes were `NotReady`, the correct approach was:

```text
Diagnose
   ↓
Find error
   ↓
Identify IAM problem
   ↓
Fix IAM
   ↓
Restart CNI
   ↓
Verify nodes
```

Instead of:

```text
Delete cluster
Recreate cluster
Hope it works
```

This is an important DevOps troubleshooting habit.

---

# 35. Final Validation Checklist

Before moving to the next stage:

```bash
aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --query 'cluster.status'
```

Should be:

```text
ACTIVE
```

Check node group:

```bash
aws eks describe-nodegroup \
  --cluster-name $CLUSTER_NAME \
  --nodegroup-name prod-general-ng \
  --query 'nodegroup.status'
```

Should be:

```text
ACTIVE
```

Check nodes:

```bash
kubectl get nodes
```

All:

```text
Ready
```

Check system pods:

```bash
kubectl get pods -n kube-system
```

Check CNI:

```bash
kubectl get pods \
  -n kube-system \
  -l k8s-app=aws-node
```

All should be:

```text
Running
```

---

# 36. Target Production Architecture

The final platform will eventually look like:

```text
                    GitLab / GitHub
                          │
                          ▼
                       Jenkins
                          │
                    Build + Test
                          │
                          ▼
                    Docker Image
                          │
                          ▼
                    Amazon ECR
                          │
                          │
                 Update GitOps Repo
                          │
                          ▼
                       ArgoCD
                          │
                          ▼
                    ┌─────────────┐
                    │     EKS     │
                    │             │
                    │ Namespace   │
                    │             │
                    │ Application │
                    │ Deployment  │
                    │ Service     │
                    └──────┬──────┘
                           │
                           ▼
                 AWS Load Balancer
                           │
                           ▼
                       Route53
                           │
                           ▼
                        HTTPS
                           │
                           ▼
                      End Users
```

Monitoring:

```text
EKS
 │
 ├── Prometheus
 ├── Grafana
 ├── CloudWatch
 └── OpenTelemetry
```

Security:

```text
IAM
 │
 ├── EKS Pod Identity
 ├── KMS
 ├── Secrets Manager
 ├── RBAC
 ├── Network Policies
 └── Pod Security
```

Reliability:

```text
Multi-AZ
 │
 ├── 3 Private Subnets
 ├── 3 Worker Nodes
 ├── HPA
 ├── Karpenter
 ├── PDB
 ├── Topology Spread
 └── Backup / DR
```

---

# 🎯 Project Goal

This EKS project is intentionally built manually so that the engineer understands:

```text
AWS Networking
       ↓
IAM
       ↓
EKS Control Plane
       ↓
Worker Nodes
       ↓
Kubernetes Networking
       ↓
Storage
       ↓
Load Balancing
       ↓
GitOps
       ↓
CI/CD
       ↓
Monitoring
       ↓
Security
       ↓
Autoscaling
       ↓
Production Operations
```

This provides a strong foundation for a **real-world AWS DevOps/SRE portfolio project**.
