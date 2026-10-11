PRODUCTION-ORIENTED AMAZON EKS CLUSTER BUILD RUNBOOK
====================================================

Purpose
-------
This file documents the end-to-end AWS CLI setup for the EKS cluster from the
recent lab: VPC/networking, EKS control plane, managed node group, core add-ons,
OIDC/IRSA, EBS CSI, and AWS Load Balancer Controller (ALB).

It is intentionally limited to EKS infrastructure and cluster add-ons. It does
NOT install Jenkins or Argo CD.

IMPORTANT SAFETY NOTES
---------------------
1. Commands create billable AWS resources: NAT gateways, EC2 nodes, EBS, ALBs,
   and related resources. Review costs before running.
2. Run from a trusted admin machine/EC2 host with AWS CLI v2, kubectl, eksctl,
   Helm, jq, curl and unzip installed.
3. Use an IAM identity authorized to create VPC, IAM, EKS, EC2, and ELB resources.
   Prefer AWS IAM Identity Center or an attached IAM role; avoid long-lived keys.
4. Verify the AWS account before running any create/delete command.
5. These commands are a reproducible baseline, not a certification of production
   readiness. Adapt endpoint access, CIDRs, KMS, logging, tags, budgets, and
   compliance settings to your environment.
6. This runbook uses a VPC with public and private subnets in three AZs:
   VPC 10.0.0.0/16
   Public: 10.0.1.0/24, 10.0.2.0/24, 10.0.3.0/24
   Private: 10.0.11.0/24, 10.0.12.0/24, 10.0.13.0/24
   AZs: ap-south-1a, ap-south-1b, ap-south-1c
   Check that these CIDRs do not overlap with any connected network.
7. Existing lab facts: region ap-south-1, cluster prod-eks, node group
   prod-general-ng, node type m7i-flex.large, three nodes, 80 GiB node disks,
   desired/minimum 3, maximum 6. The prior lab used Kubernetes 1.36; always
   verify the version is currently supported before creating a new cluster.
8. If prod-eks already exists, DO NOT rerun cluster/VPC creation commands blindly.
   First inspect it and skip already-created resources. Resource names must be
   unique in your account/region.

CONTENTS
--------
1. Install/check tools and set variables
2. Create VPC, subnets, internet gateway, NAT gateways and routes
3. Create IAM roles
4. Create EKS cluster
5. Create managed node group
6. Configure kubectl and validate nodes
7. Install/verify core EKS add-ons
8. Configure OIDC and IRSA
9. Configure EBS CSI driver IAM and storage
10. Configure AWS Load Balancer Controller and ALB
11. Enable control-plane logs and final validation
12. Troubleshooting from the lab
13. Clean-up warning and official references


1. INSTALL/CHECK TOOLS AND SET VARIABLES
========================================

Why:
- The AWS CLI provisions AWS resources.
- kubectl talks to the Kubernetes API.
- eksctl simplifies registering the cluster's OIDC provider.
- Helm installs the AWS Load Balancer Controller chart.
- jq helps inspect JSON responses.

Check installed tools:

    aws --version
    kubectl version --client
    eksctl version
    helm version
    jq --version
    curl --version

Check current AWS identity:

    aws sts get-caller-identity

Why:
This shows the account and IAM identity that will own the resources. Stop if
the account is not the intended account.

Set the variables:

    export AWS_REGION="ap-south-1"
    export CLUSTER_NAME="prod-eks"
    export NODEGROUP_NAME="prod-general-ng"
    export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
    export AWS_PARTITION="aws"
    export VPC_CIDR="10.0.0.0/16"

    printf 'Account=%s Region=%s Cluster=%s NodeGroup=%s\n' \
      "$AWS_ACCOUNT_ID" "$AWS_REGION" "$CLUSTER_NAME" "$NODEGROUP_NAME"

Why:
A previous failure occurred because CLUSTER_NAME or subnet variables were
empty. This validation catches empty or incorrect variables before provisioning.

Check whether the cluster already exists:

    aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION"

If this succeeds, inspect the existing cluster rather than creating it again.
If it returns ResourceNotFoundException, continue only if this is a new-cluster
build and you have confirmed the rest of the resources do not already exist.


2. CREATE THE VPC AND NETWORKING
================================

Production network plan
-----------------------
- Three public subnets: for internet-facing ALBs and NAT gateways.
- Three private subnets: for EKS worker nodes.
- One NAT gateway per AZ for resilient private-subnet egress. NAT gateways cost
  money; one NAT gateway is cheaper but introduces an AZ dependency.
- EKS worker nodes should not receive public IP addresses.
- Ensure private subnets have sufficient IP capacity for nodes and pods.

2.1 Create VPC

    aws ec2 create-vpc \
      --cidr-block "$VPC_CIDR" \
      --region "$AWS_REGION" \
      --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=prod-eks-vpc},{Key=Environment,Value=production}]'

    export VPC_ID="$(aws ec2 describe-vpcs --region "$AWS_REGION" \
      --filters "Name=tag:Name,Values=prod-eks-vpc" \
      --query 'Vpcs[0].VpcId' --output text)"

    echo "$VPC_ID"

Enable DNS support and DNS hostnames:

    aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-support '{"Value":true}' --region "$AWS_REGION"
    aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-hostnames '{"Value":true}' --region "$AWS_REGION"

Why:
EKS and internal service discovery rely on VPC networking and DNS. DNS
hostnames/support should be enabled for normal EKS operation.

2.2 Create six subnets

    export PUB_A="$(aws ec2 create-subnet --vpc-id "$VPC_ID" --availability-zone ap-south-1a --cidr-block 10.0.1.0/24 --region "$AWS_REGION" --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=prod-eks-public-a},{Key=kubernetes.io/role/elb,Value=1},{Key=kubernetes.io/cluster/prod-eks,Value=shared}]' --query 'Subnet.SubnetId' --output text)"
    export PUB_B="$(aws ec2 create-subnet --vpc-id "$VPC_ID" --availability-zone ap-south-1b --cidr-block 10.0.2.0/24 --region "$AWS_REGION" --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=prod-eks-public-b},{Key=kubernetes.io/role/elb,Value=1},{Key=kubernetes.io/cluster/prod-eks,Value=shared}]' --query 'Subnet.SubnetId' --output text)"
    export PUB_C="$(aws ec2 create-subnet --vpc-id "$VPC_ID" --availability-zone ap-south-1c --cidr-block 10.0.3.0/24 --region "$AWS_REGION" --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=prod-eks-public-c},{Key=kubernetes.io/role/elb,Value=1},{Key=kubernetes.io/cluster/prod-eks,Value=shared}]' --query 'Subnet.SubnetId' --output text)"

    export PRIV_A="$(aws ec2 create-subnet --vpc-id "$VPC_ID" --availability-zone ap-south-1a --cidr-block 10.0.11.0/24 --region "$AWS_REGION" --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=prod-eks-private-a},{Key=kubernetes.io/role/internal-elb,Value=1},{Key=kubernetes.io/cluster/prod-eks,Value=shared}]' --query 'Subnet.SubnetId' --output text)"
    export PRIV_B="$(aws ec2 create-subnet --vpc-id "$VPC_ID" --availability-zone ap-south-1b --cidr-block 10.0.12.0/24 --region "$AWS_REGION" --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=prod-eks-private-b},{Key=kubernetes.io/role/internal-elb,Value=1},{Key=kubernetes.io/cluster/prod-eks,Value=shared}]' --query 'Subnet.SubnetId' --output text)"
    export PRIV_C="$(aws ec2 create-subnet --vpc-id "$VPC_ID" --availability-zone ap-south-1c --cidr-block 10.0.13.0/24 --region "$AWS_REGION" --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=prod-eks-private-c},{Key=kubernetes.io/role/internal-elb,Value=1},{Key=kubernetes.io/cluster/prod-eks,Value=shared}]' --query 'Subnet.SubnetId' --output text)"

Check IDs and AZs:

    printf 'VPC=%s\nPublic=%s %s %s\nPrivate=%s %s %s\n' \
      "$VPC_ID" "$PUB_A" "$PUB_B" "$PUB_C" "$PRIV_A" "$PRIV_B" "$PRIV_C"

    aws ec2 describe-subnets --region "$AWS_REGION" \
      --subnet-ids "$PUB_A" "$PUB_B" "$PUB_C" "$PRIV_A" "$PRIV_B" "$PRIV_C" \
      --query 'Subnets[].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock,PublicIP:MapPublicIpOnLaunch}' \
      --output table

Why:
Subnet IDs are required by EKS. The prior lab hit “Subnets are required” because
the subnet variables were empty. The table confirms IDs, CIDRs, and AZ placement.

Do not assign public IPs automatically to private subnets. If a subnet was
created with public IP assignment enabled by mistake, disable it:

    aws ec2 modify-subnet-attribute --subnet-id "$PRIV_A" --no-map-public-ip-on-launch --region "$AWS_REGION"
    aws ec2 modify-subnet-attribute --subnet-id "$PRIV_B" --no-map-public-ip-on-launch --region "$AWS_REGION"
    aws ec2 modify-subnet-attribute --subnet-id "$PRIV_C" --no-map-public-ip-on-launch --region "$AWS_REGION"

2.3 Create and attach Internet Gateway

    export IGW_ID="$(aws ec2 create-internet-gateway --region "$AWS_REGION" \
      --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=prod-eks-igw}]' \
      --query 'InternetGateway.InternetGatewayId' --output text)"

    aws ec2 attach-internet-gateway --vpc-id "$VPC_ID" --internet-gateway-id "$IGW_ID" --region "$AWS_REGION"

Why:
The internet gateway provides internet routing for public subnets, including
internet-facing ALB subnets and NAT gateway public IP connectivity.

2.4 Create public route table and default route

    export PUBLIC_RT="$(aws ec2 create-route-table --vpc-id "$VPC_ID" --region "$AWS_REGION" \
      --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=prod-eks-public-rt}]' \
      --query 'RouteTable.RouteTableId' --output text)"

    aws ec2 create-route --route-table-id "$PUBLIC_RT" --destination-cidr-block 0.0.0.0/0 --gateway-id "$IGW_ID" --region "$AWS_REGION"

    aws ec2 associate-route-table --route-table-id "$PUBLIC_RT" --subnet-id "$PUB_A" --region "$AWS_REGION"
    aws ec2 associate-route-table --route-table-id "$PUBLIC_RT" --subnet-id "$PUB_B" --region "$AWS_REGION"
    aws ec2 associate-route-table --route-table-id "$PUBLIC_RT" --subnet-id "$PUB_C" --region "$AWS_REGION"

Why:
The 0.0.0.0/0 route sends public subnet internet traffic through the IGW.
The public subnet tags allow the AWS Load Balancer Controller to discover
subnets for internet-facing ALBs.

2.5 Allocate Elastic IPs and create one NAT gateway per AZ

    export EIP_A="$(aws ec2 allocate-address --domain vpc --region "$AWS_REGION" --query AllocationId --output text)"
    export EIP_B="$(aws ec2 allocate-address --domain vpc --region "$AWS_REGION" --query AllocationId --output text)"
    export EIP_C="$(aws ec2 allocate-address --domain vpc --region "$AWS_REGION" --query AllocationId --output text)"

    export NAT_A="$(aws ec2 create-nat-gateway --subnet-id "$PUB_A" --allocation-id "$EIP_A" --region "$AWS_REGION" --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=prod-eks-nat-a}]' --query NatGateway.NatGatewayId --output text)"
    export NAT_B="$(aws ec2 create-nat-gateway --subnet-id "$PUB_B" --allocation-id "$EIP_B" --region "$AWS_REGION" --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=prod-eks-nat-b}]' --query NatGateway.NatGatewayId --output text)"
    export NAT_C="$(aws ec2 create-nat-gateway --subnet-id "$PUB_C" --allocation-id "$EIP_C" --region "$AWS_REGION" --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=prod-eks-nat-c}]' --query NatGateway.NatGatewayId --output text)"

Wait until NAT gateways are available:

    aws ec2 wait nat-gateway-available --nat-gateway-ids "$NAT_A" "$NAT_B" "$NAT_C" --region "$AWS_REGION"

Why:
Private nodes need outbound access for image pulls and external endpoints unless
the required traffic is served through VPC endpoints. One NAT per AZ avoids
making all private subnets depend on one AZ's NAT gateway. NAT gateways are
billable resources.

2.6 Create private route tables and route each AZ through its own NAT

    export PRIVATE_RT_A="$(aws ec2 create-route-table --vpc-id "$VPC_ID" --region "$AWS_REGION" --query RouteTable.RouteTableId --output text)"
    export PRIVATE_RT_B="$(aws ec2 create-route-table --vpc-id "$VPC_ID" --region "$AWS_REGION" --query RouteTable.RouteTableId --output text)"
    export PRIVATE_RT_C="$(aws ec2 create-route-table --vpc-id "$VPC_ID" --region "$AWS_REGION" --query RouteTable.RouteTableId --output text)"

    aws ec2 create-route --route-table-id "$PRIVATE_RT_A" --destination-cidr-block 0.0.0.0/0 --nat-gateway-id "$NAT_A" --region "$AWS_REGION"
    aws ec2 create-route --route-table-id "$PRIVATE_RT_B" --destination-cidr-block 0.0.0.0/0 --nat-gateway-id "$NAT_B" --region "$AWS_REGION"
    aws ec2 create-route --route-table-id "$PRIVATE_RT_C" --destination-cidr-block 0.0.0.0/0 --nat-gateway-id "$NAT_C" --region "$AWS_REGION"

    aws ec2 associate-route-table --route-table-id "$PRIVATE_RT_A" --subnet-id "$PRIV_A" --region "$AWS_REGION"
    aws ec2 associate-route-table --route-table-id "$PRIVATE_RT_B" --subnet-id "$PRIV_B" --region "$AWS_REGION"
    aws ec2 associate-route-table --route-table-id "$PRIVATE_RT_C" --subnet-id "$PRIV_C" --region "$AWS_REGION"

Verify routes:

    aws ec2 describe-route-tables --route-table-ids "$PUBLIC_RT" "$PRIVATE_RT_A" "$PRIVATE_RT_B" "$PRIVATE_RT_C" --region "$AWS_REGION" --output table

Why:
Each private subnet sends internet-bound egress through its same-AZ NAT gateway.
This keeps worker nodes private while allowing outbound access.

2.7 Validate subnet tags and routes

    aws ec2 describe-subnets --region "$AWS_REGION" \
      --subnet-ids "$PUB_A" "$PUB_B" "$PUB_C" "$PRIV_A" "$PRIV_B" "$PRIV_C" \
      --output table

Confirm:
- Public route table has 0.0.0.0/0 -> Internet Gateway.
- Private route tables have 0.0.0.0/0 -> same-AZ NAT gateway.
- Public subnets have kubernetes.io/role/elb=1.
- Private subnets have kubernetes.io/role/internal-elb=1.
- Cluster tags match prod-eks.
- Subnet CIDRs do not overlap other networks.


3. CREATE IAM ROLES
===================

IAM role types are different:
- EKS cluster role: assumed by eks.amazonaws.com to manage EKS control-plane
  integrations with AWS.
- Node role: assumed by EC2 worker instances.
- IRSA roles: assumed by a specific Kubernetes service account using OIDC.
Never reuse one role for every component.

3.1 Create EKS cluster role

Create trust policy:

    cat > eks-cluster-trust.json <<'EOF'
    {
      "Version": "2012-10-17",
      "Statement": [{
        "Effect": "Allow",
        "Principal": {"Service": "eks.amazonaws.com"},
        "Action": "sts:AssumeRole"
      }]
    }
    EOF

Create role and attach the cluster policy:

    aws iam create-role --role-name prod-eks-cluster-role \
      --assume-role-policy-document file://eks-cluster-trust.json

    aws iam attach-role-policy --role-name prod-eks-cluster-role \
      --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

Why:
The control plane needs its own role. It must not use the EC2 worker node role.

3.2 Create worker node role

    cat > eks-node-trust.json <<'EOF'
    {
      "Version": "2012-10-17",
      "Statement": [{
        "Effect": "Allow",
        "Principal": {"Service": "ec2.amazonaws.com"},
        "Action": "sts:AssumeRole"
      }]
    }
    EOF

    aws iam create-role --role-name prod-eks-node-role \
      --assume-role-policy-document file://eks-node-trust.json

Attach the standard node policies:

    aws iam attach-role-policy --role-name prod-eks-node-role \
      --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy

    aws iam attach-role-policy --role-name prod-eks-node-role \
      --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPullOnly

VPC CNI IAM choice:
For the lab, aws-node crashed because the node role lacked required EC2 network
interface permissions. The quick diagnostic/fix used was:

    aws iam attach-role-policy --role-name prod-eks-node-role \
      --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy

This attaches CNI permissions to the node role and is a simple recovery path.
For a stricter production design, give the VPC CNI its own IRSA/Pod Identity role,
then remove AmazonEKS_CNI_Policy from the node role only after confirming the
CNI service account has the dedicated permissions and networking works. Do not
remove permissions first or nodes/pods may lose networking.

Wait briefly for IAM propagation before creating EKS resources.


4. CREATE THE EKS CONTROL PLANE
===============================

4.1 Choose the supported Kubernetes version

    aws eks describe-cluster-versions --region "$AWS_REGION" 2>/dev/null

If that command is not available in your CLI version, consult the current EKS
Kubernetes version support documentation and choose a currently supported
version. The lab was seen using Kubernetes 1.36, but verify current support and
compatibility rather than blindly hardcoding it.

Set a verified version:

    export K8S_VERSION="<SUPPORTED_VERSION>"

4.2 Create cluster

This template creates a cluster with private API endpoint access and public
access disabled. Use this only if the admin host can reach the private endpoint
through the VPC, VPN or Direct Connect. If your admin machine is outside the VPC,
you must arrange private connectivity first or use a restricted public endpoint.

    aws eks create-cluster \
      --name "$CLUSTER_NAME" \
      --region "$AWS_REGION" \
      --kubernetes-version "$K8S_VERSION" \
      --role-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:role/prod-eks-cluster-role" \
      --resources-vpc-config "subnetIds=${PRIV_A},${PRIV_B},${PRIV_C},endpointPrivateAccess=true,endpointPublicAccess=false"

Why:
- EKS creates the managed Kubernetes control plane.
- Private subnets provide the cluster's VPC placement.
- Private endpoint access reduces public exposure.
- The EKS cluster role allows the control plane to call the AWS APIs it needs.

Wait for the cluster:

    aws eks wait cluster-active --name "$CLUSTER_NAME" --region "$AWS_REGION"

Inspect it:

    aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" \
      --query 'cluster.{Status:status,Version:version,Endpoint:endpoint,OIDC:identity.oidc.issuer,VPC:resourcesVpcConfig}' \
      --output yaml

If wait fails, inspect:

    aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --output yaml


5. CREATE THE MANAGED NODE GROUP
================================

5.1 Create node group

The lab configuration used three on-demand m7i-flex.large nodes, 80 GiB disks,
desired 3, minimum 3 and maximum 6. Instance availability and pricing depend on
region/AZ and can change. The earlier m6i.large attempt failed in the lab because
of a Free Tier eligibility message; m7i-flex.large was subsequently used.

    aws eks create-nodegroup \
      --cluster-name "$CLUSTER_NAME" \
      --nodegroup-name "$NODEGROUP_NAME" \
      --region "$AWS_REGION" \
      --subnets "$PRIV_A" "$PRIV_B" "$PRIV_C" \
      --node-role "arn:aws:iam::${AWS_ACCOUNT_ID}:role/prod-eks-node-role" \
      --instance-types m7i-flex.large \
      --ami-type AL2023_x86_64_STANDARD \
      --capacity-type ON_DEMAND \
      --disk-size 80 \
      --scaling-config minSize=3,maxSize=6,desiredSize=3

Why:
- Managed node groups automate EC2 node lifecycle and EKS node registration.
- Private subnets keep worker nodes off public IPs.
- Three nodes across three AZs improve fault tolerance.
- 80 GiB disk provides space for container images/logs; monitor actual disk use.
- Scaling config sets minimum, maximum and initial desired node count.

Wait:

    aws eks wait nodegroup-active --cluster-name "$CLUSTER_NAME" \
      --nodegroup-name "$NODEGROUP_NAME" --region "$AWS_REGION"

Inspect:

    aws eks describe-nodegroup --cluster-name "$CLUSTER_NAME" \
      --nodegroup-name "$NODEGROUP_NAME" --region "$AWS_REGION" \
      --query 'nodegroup.{Status:status,Health:health,Scaling:scalingConfig,Subnets:subnets,InstanceTypes:instanceTypes,NodeRole:nodeRole}' \
      --output yaml


6. CONFIGURE KUBECTL AND VERIFY NODES
=====================================

    aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"

Why:
This writes/updates a kubeconfig context so kubectl can authenticate to the EKS
API using AWS IAM authentication.

Check:

    kubectl config current-context
    kubectl cluster-info
    kubectl get nodes -o wide
    kubectl get pods -n kube-system -o wide
    kubectl get events -A --sort-by=.lastTimestamp | tail -n 80

Expected:
- Three worker nodes should eventually show Ready.
- Core system pods should be Running/Completed as appropriate.
- A node group being ACTIVE does not guarantee every node and CNI pod is healthy.

If nodes are NotReady, do not continue to application deployment. Troubleshoot
VPC CNI, node IAM, routes, security groups, subnet IP availability and node
health first.


7. INSTALL/VERIFY CORE EKS ADD-ONS
==================================

The lab initially returned an empty add-on list:

    aws eks list-addons --cluster-name "$CLUSTER_NAME" --region "$AWS_REGION"

That means no EKS-managed add-ons were listed at that point; it does not mean
every Kubernetes system component is necessarily absent. Inspect pods too.

List compatible add-on versions:

    aws eks describe-addon-versions --region "$AWS_REGION" \
      --kubernetes-version "$K8S_VERSION" \
      --query 'addons[].addonName' --output table

Review the current compatible version for each add-on before installation.
EKS add-on versions change over time.

7.1 VPC CNI

VPC CNI gives pods VPC networking and manages pod IP allocation.

    aws eks describe-addon --cluster-name "$CLUSTER_NAME" --addon-name vpc-cni \
      --region "$AWS_REGION" 2>/dev/null

If absent, install a compatible version (set ADDON_VERSION to one returned by
describe-addon-versions):

    export VPC_CNI_VERSION="<COMPATIBLE_ADDON_VERSION>"
    aws eks create-addon --cluster-name "$CLUSTER_NAME" --addon-name vpc-cni \
      --addon-version "$VPC_CNI_VERSION" --region "$AWS_REGION"

Do not blindly use --resolve-conflicts OVERWRITE on an existing cluster because
it can overwrite customized settings. Inspect configuration first.

7.2 CoreDNS

CoreDNS provides cluster DNS for Kubernetes services.

    export COREDNS_VERSION="<COMPATIBLE_ADDON_VERSION>"
    aws eks create-addon --cluster-name "$CLUSTER_NAME" --addon-name coredns \
      --addon-version "$COREDNS_VERSION" --region "$AWS_REGION"

If CoreDNS already exists, use describe-addon rather than create-addon.

7.3 kube-proxy

kube-proxy implements Kubernetes Service networking rules on nodes.

    export KUBEPROXY_VERSION="<COMPATIBLE_ADDON_VERSION>"
    aws eks create-addon --cluster-name "$CLUSTER_NAME" --addon-name kube-proxy \
      --addon-version "$KUBEPROXY_VERSION" --region "$AWS_REGION"

7.4 Verify core add-ons

    aws eks list-addons --cluster-name "$CLUSTER_NAME" --region "$AWS_REGION"

    kubectl get pods -n kube-system -o wide

    for ADDON in vpc-cni coredns kube-proxy; do
      aws eks describe-addon --cluster-name "$CLUSTER_NAME" --addon-name "$ADDON" \
        --region "$AWS_REGION" \
        --query 'addon.{Name:addonName,Status:status,Version:addonVersion,Health:health}' \
        --output yaml
    done

Important:
The create-addon commands require compatible version strings. Get those from
AWS for the selected Kubernetes version. If an add-on is already present, inspect
it and update it instead of creating it a second time.

7.5 VPC CNI IAM: recommended production path

For a dedicated workload identity:
- Associate the IAM OIDC provider (Section 8).
- Create an IAM role trusted only by the kube-system/aws-node service account.
- Attach the current AWS-recommended VPC CNI policy.
- Configure the VPC CNI add-on's service-account role or IRSA annotation using
  the supported EKS add-on configuration.
- Verify aws-node pods and pod networking before removing CNI permissions from
  the EC2 node role.

Because the lab's aws-node failed due to missing EC2 network-interface
permissions, do not remove AmazonEKS_CNI_Policy from the node role until the
dedicated identity is tested successfully.


8. OIDC PROVIDER AND IRSA
=========================

8.1 What OIDC and IRSA mean

EKS issues projected service-account tokens with an OIDC issuer. IAM can trust
the issuer so a Kubernetes service account can assume a specific IAM role.
IRSA avoids granting every pod the broad permissions of the EC2 node role.

OIDC provider setup is needed for IRSA. It is not required for EKS Pod Identity,
which uses a different association mechanism.

8.2 Associate the OIDC provider

    eksctl utils associate-iam-oidc-provider \
      --cluster "$CLUSTER_NAME" \
      --region "$AWS_REGION" \
      --approve

Why:
This registers the EKS cluster's OIDC issuer as an IAM OpenID Connect provider,
so IAM trust policies can validate projected service-account tokens.

Verify issuer:

    export OIDC_ISSUER="$(aws eks describe-cluster --name "$CLUSTER_NAME" \
      --region "$AWS_REGION" --query 'cluster.identity.oidc.issuer' --output text)"
    export OIDC_ID="${OIDC_ISSUER#https://}"

    echo "$OIDC_ISSUER"
    echo "$OIDC_ID"
    aws iam list-open-id-connect-providers

The lab's issuer identifier was recorded as CE19D5ACC32BBE08ECE0454E7E1DDEB2
for that particular cluster. A recreated cluster can have a different issuer.
Always query the live cluster; do not reuse an old issuer ID or thumbprint.

8.3 IRSA trust-policy pattern

An IRSA role trust policy must include:
- Principal Federated = IAM OIDC provider ARN for this cluster.
- Action sts:AssumeRoleWithWebIdentity.
- Condition <OIDC_ID>:aud = sts.amazonaws.com.
- Condition <OIDC_ID>:sub = system:serviceaccount:<namespace>:<service-account>.

Example only (replace placeholders with values obtained from the live cluster):

    {
      "Version": "2012-10-17",
      "Statement": [{
        "Effect": "Allow",
        "Principal": {
          "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/<OIDC_ID>"
        },
        "Action": "sts:AssumeRoleWithWebIdentity",
        "Condition": {
          "StringEquals": {
            "<OIDC_ID>:aud": "sts.amazonaws.com",
            "<OIDC_ID>:sub": "system:serviceaccount:kube-system/<SERVICE_ACCOUNT>"
          }
        }
      }]
    }

The trust policy says WHICH Kubernetes identity may assume the role. The attached
permissions policy says WHAT AWS APIs that role can call. These are separate
controls and both must be correct.

8.4 Troubleshoot InvalidIdentityToken

The lab encountered InvalidIdentityToken while configuring the EBS CSI role.
Check:

    aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" \
      --query 'cluster.identity.oidc.issuer' --output text
    aws iam list-open-id-connect-providers
    aws iam get-role --role-name AmazonEKS_EBS_CSI_DriverRole
    kubectl get serviceaccounts -n kube-system
    kubectl get pods -n kube-system -o wide

Common causes:
- No IAM OIDC provider registered for this cluster.
- Trust policy references an OIDC provider from a different/recreated cluster.
- `aud` is not sts.amazonaws.com.
- `sub` does not exactly match namespace and service-account name.
- Role ARN annotation/add-on setting is wrong.
- IAM changes have not propagated yet.
- The workload is using a different service account than expected.


9. EBS CSI DRIVER AND STORAGE IAM
=================================

Why EBS CSI:
The EBS CSI driver lets Kubernetes dynamically provision, attach and mount EBS
volumes for PVCs, including stateful workloads. It needs AWS IAM permissions.

9.1 Inspect service account and add-on status

    kubectl get serviceaccounts -n kube-system
    kubectl get pods -n kube-system | grep ebs
    aws eks describe-addon --cluster-name "$CLUSTER_NAME" \
      --addon-name aws-ebs-csi-driver --region "$AWS_REGION" --output yaml

9.2 Create EBS CSI IAM role using eksctl (IRSA path)

First ensure Section 8 OIDC setup is complete. Check the current official EBS
CSI IAM policy guidance and use the policy recommended for your configuration.
The AWS-managed policy commonly used is AmazonEBSCSIDriverPolicy.

    eksctl create iamserviceaccount \
      --cluster "$CLUSTER_NAME" \
      --region "$AWS_REGION" \
      --namespace kube-system \
      --name ebs-csi-controller-sa \
      --role-name AmazonEKS_EBS_CSI_DriverRole \
      --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
      --approve \
      --override-existing-serviceaccounts

Caution:
If the EKS-managed add-on already owns or configures this service account, use
the EKS add-on's supported service-account role configuration instead of
creating conflicting configuration. Confirm the actual service-account name
and role before applying this command to an existing installation.

9.3 Install EBS CSI add-on if absent

    aws eks create-addon --cluster-name "$CLUSTER_NAME" \
      --addon-name aws-ebs-csi-driver --region "$AWS_REGION"

If the command reports it already exists, describe the add-on and inspect its
status/health instead of creating it again. Configure the role through the
supported add-on API/console if necessary.

Verify:

    aws eks describe-addon --cluster-name "$CLUSTER_NAME" \
      --addon-name aws-ebs-csi-driver --region "$AWS_REGION" \
      --query 'addon.{Status:status,Version:addonVersion,Health:health}' --output yaml

    kubectl get pods -n kube-system | grep ebs
    kubectl get csidrivers

9.4 Create a gp3 StorageClass

For production, consider encrypted gp3 volumes and a retention policy. A KMS key
can be specified if your organization requires a customer-managed key.

    cat > gp3-encrypted-storageclass.yaml <<'EOF'
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: gp3-encrypted
    provisioner: ebs.csi.aws.com
    parameters:
      type: gp3
      encrypted: "true"
    volumeBindingMode: WaitForFirstConsumer
    allowVolumeExpansion: true
    reclaimPolicy: Retain
    EOF

    kubectl apply -f gp3-encrypted-storageclass.yaml
    kubectl get storageclass
    kubectl describe storageclass gp3-encrypted

Why:
- gp3 is a general-purpose SSD volume type.
- encrypted=true encrypts new EBS volumes using the AWS default EBS key unless
  a customer-managed KMS key is specified.
- WaitForFirstConsumer delays binding until Kubernetes knows the pod's AZ.
- Retain prevents Kubernetes PVC deletion from automatically deleting the PV's
  underlying volume under the normal reclaim workflow; it also means cleanup
  requires deliberate volume management.

9.5 Test EBS provisioning

    cat > ebs-smoke-test-pvc.yaml <<'EOF'
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: ebs-smoke-test
      namespace: default
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: gp3-encrypted
      resources:
        requests:
          storage: 1Gi
    EOF

    kubectl apply -f ebs-smoke-test-pvc.yaml

Create a consumer pod so WaitForFirstConsumer can select an AZ:

    cat > ebs-smoke-test-pod.yaml <<'EOF'
    apiVersion: v1
    kind: Pod
    metadata:
      name: ebs-smoke-test
      namespace: default
    spec:
      containers:
        - name: test
          image: public.ecr.aws/docker/library/busybox:1.36
          command: ["sh", "-c", "date > /data/test.txt; sleep 3600"]
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: ebs-smoke-test
    EOF

    kubectl apply -f ebs-smoke-test-pod.yaml
    kubectl get pvc,pod -n default -o wide
    kubectl describe pvc ebs-smoke-test
    kubectl logs ebs-smoke-test
    kubectl exec ebs-smoke-test -- cat /data/test.txt

If the PVC stays Pending, inspect events, scheduler decisions, EBS CSI controller
logs, IAM permissions and node/AZ availability. Pending before a consumer pod
is scheduled can be normal with WaitForFirstConsumer.

After successful testing, remove only these test resources:

    kubectl delete pod ebs-smoke-test
    kubectl delete pvc ebs-smoke-test


10. AWS LOAD BALANCER CONTROLLER AND ALB
========================================

What it does:
The AWS Load Balancer Controller watches Kubernetes Ingress and selected Service
resources and creates/configures AWS ALBs/NLBs. The controller needs IAM rights
to describe networking resources and manage load balancers/target groups.
Do not give it AdministratorAccess.

10.1 Verify subnet tags

    aws ec2 describe-subnets --region "$AWS_REGION" \
      --subnet-ids "$PUB_A" "$PUB_B" "$PUB_C" "$PRIV_A" "$PRIV_B" "$PRIV_C" \
      --output json

Public subnets for internet-facing ALBs should have:
    kubernetes.io/role/elb = 1

Private subnets for internal ALBs should have:
    kubernetes.io/role/internal-elb = 1

The controller must be able to discover the intended subnets. Wrong/missing tags
are a common reason an Ingress does not produce an ALB.

10.2 Download the official IAM policy

Use the AWS Load Balancer Controller installation guide for the exact version
you intend to install and download the matching official IAM policy JSON. Do
not use an old copied policy: missing actions can produce AccessDenied errors
for operations such as DescribeTags or DescribeLoadBalancerAttributes.

The official installation page is:
https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html

The upstream policy source is maintained by the AWS Load Balancer Controller
project. For a version-pinned installation, use the policy file from the same
release tag as the chart/controller version.

After saving the reviewed official policy as iam_policy.json:

    aws iam create-policy \
      --policy-name AWSLoadBalancerControllerIAMPolicy-prod-eks \
      --policy-document file://iam_policy.json

If the policy already exists, get its ARN and confirm its version/document rather
than creating a duplicate:

    aws iam list-policies --scope Local \
      --query "Policies[?PolicyName=='AWSLoadBalancerControllerIAMPolicy-prod-eks'].Arn" \
      --output text

Set the policy ARN (use the exact ARN returned by AWS):

    export LBC_POLICY_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy-prod-eks"

10.3 Create the controller service account and IRSA role

After OIDC association, run:

    eksctl create iamserviceaccount \
      --cluster "$CLUSTER_NAME" \
      --region "$AWS_REGION" \
      --namespace kube-system \
      --name aws-load-balancer-controller \
      --role-name AmazonEKSLoadBalancerControllerRole \
      --attach-policy-arn "$LBC_POLICY_ARN" \
      --approve \
      --override-existing-serviceaccounts

Why:
This creates an IAM role with a trust policy restricted to the
kube-system/aws-load-balancer-controller Kubernetes service account and attaches
the controller's required permissions policy.

If the service account already exists, inspect its annotations and IAM trust
policy before overwriting it.

Verify:

    kubectl get serviceaccount aws-load-balancer-controller -n kube-system -o yaml
    aws iam get-role --role-name AmazonEKSLoadBalancerControllerRole

10.4 Install Helm chart

Add the official EKS charts repository:

    helm repo add eks https://aws.github.io/eks-charts
    helm repo update

Discover available chart versions:

    helm search repo eks/aws-load-balancer-controller --versions | head -n 20

For production, choose and pin a reviewed chart version compatible with your EKS
version. Set the selected version explicitly:

    export LBC_CHART_VERSION="<REVIEWED_CHART_VERSION>"

Find the VPC ID:

    export VPC_ID="$(aws eks describe-cluster --name "$CLUSTER_NAME" \
      --region "$AWS_REGION" --query 'cluster.resourcesVpcConfig.vpcId' --output text)"
    echo "$VPC_ID"

Install the controller:

    helm upgrade --install aws-load-balancer-controller \
      eks/aws-load-balancer-controller \
      --namespace kube-system \
      --version "$LBC_CHART_VERSION" \
      --set clusterName="$CLUSTER_NAME" \
      --set region="$AWS_REGION" \
      --set vpcId="$VPC_ID" \
      --set serviceAccount.create=false \
      --set serviceAccount.name=aws-load-balancer-controller

Why:
- clusterName identifies the EKS cluster to reconcile.
- region and vpcId tell the controller where to manage load balancers.
- serviceAccount.create=false uses the service account and IRSA role created above.
- A pinned chart version makes the install repeatable.

Verify:

    kubectl rollout status deployment/aws-load-balancer-controller -n kube-system
    kubectl get deployment,pods -n kube-system | grep aws-load-balancer-controller
    kubectl logs -n kube-system deployment/aws-load-balancer-controller --tail=200

10.5 ALB smoke-test Ingress

Only run this if you already have an application Service named app-service in
namespace apps-test that listens on port 80. Replace the namespace, service name,
host, and certificate ARN for your actual application. A certificate ARN is
required for HTTPS configuration; this simple smoke test uses HTTP and should
not be considered a production TLS setup.

Example for an internet-facing test ALB:

    cat > alb-smoke-test-ingress.yaml <<'EOF'
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: alb-smoke-test
      namespace: apps-test
      annotations:
        alb.ingress.kubernetes.io/scheme: internet-facing
        alb.ingress.kubernetes.io/target-type: ip
        alb.ingress.kubernetes.io/healthcheck-path: /
    spec:
      ingressClassName: alb
      rules:
        - http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: app-service
                    port:
                      number: 80
    EOF

    kubectl apply -f alb-smoke-test-ingress.yaml
    kubectl get ingress -n apps-test -w
    kubectl describe ingress alb-smoke-test -n apps-test

For a production ALB, configure HTTPS with an ACM certificate, redirect HTTP to
HTTPS, use an appropriate internal/internet-facing scheme, health checks, WAF
where required, and restricted access. Avoid exposing admin endpoints publicly.

10.6 Troubleshoot ALB errors

    kubectl describe ingress -A
    kubectl logs -n kube-system deployment/aws-load-balancer-controller --tail=300
    kubectl get serviceaccount aws-load-balancer-controller -n kube-system -o yaml
    aws iam get-role --role-name AmazonEKSLoadBalancerControllerRole

If logs show AccessDenied for DescribeTags, DescribeTargetGroupAttributes,
DescribeLoadBalancerAttributes or similar actions:
- compare the attached policy to the official policy for the installed release;
- attach a correct policy version through an approved change;
- check that the controller pod is using the expected service account;
- check IRSA role ARN and trust conditions;
- restart only after the role/policy is corrected.

If the error is InvalidIdentityToken:
- verify the IAM OIDC provider exists for this exact cluster;
- verify the role trust policy's OIDC provider ARN, aud and sub;
- verify the service account name/namespace and role annotation.


11. ENABLE CONTROL-PLANE LOGGING AND FINAL VALIDATION
====================================================

Enable EKS API, audit, authenticator, controller-manager and scheduler logs.
CloudWatch Logs charges and retention settings apply.

    aws eks update-cluster-config \
      --name "$CLUSTER_NAME" \
      --region "$AWS_REGION" \
      --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'

Wait/check the update:

    aws eks list-updates --name "$CLUSTER_NAME" --region "$AWS_REGION"
    aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" \
      --query 'cluster.logging' --output yaml

Final validation:

    aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" \
      --query 'cluster.{Status:status,Version:version,Logging:logging,EndpointAccess:resourcesVpcConfig,OIDC:identity.oidc.issuer}' \
      --output yaml

    aws eks list-nodegroups --cluster-name "$CLUSTER_NAME" --region "$AWS_REGION"
    aws eks list-addons --cluster-name "$CLUSTER_NAME" --region "$AWS_REGION"
    kubectl get nodes -o wide
    kubectl get pods -A
    kubectl get storageclass
    kubectl get csidrivers
    kubectl get ingress -A
    kubectl get events -A --sort-by=.lastTimestamp | tail -n 100

Go-live requirements:
[ ] Cluster is ACTIVE and uses a currently supported Kubernetes version.
[ ] Nodes are Ready across at least two AZs (three AZs preferred).
[ ] Private workers have working network egress and adequate pod IP capacity.
[ ] VPC CNI, CoreDNS and kube-proxy are healthy.
[ ] OIDC provider is registered if using IRSA.
[ ] Each workload/add-on has a least-privilege IAM identity.
[ ] EBS CSI is healthy and a test PVC mounts successfully.
[ ] AWS Load Balancer Controller is healthy and a test ALB has healthy targets.
[ ] EKS control-plane logs are enabled and retained appropriately.
[ ] Alerting, backups, restore tests, upgrade plan, budgets and access reviews exist.


12. TROUBLESHOOTING FROM THE LAB
===============================

Problem: "expected one argument"
Cause: a shell variable such as CLUSTER_NAME was empty.
Fix:
    export CLUSTER_NAME="prod-eks"
    printf '%s\n' "$CLUSTER_NAME"

Problem: "Subnets are required"
Cause: subnet ID variables were blank or no subnet IDs were passed.
Fix:
    printf 'PUB_A=%s PUB_B=%s PUB_C=%s\n' "$PUB_A" "$PUB_B" "$PUB_C"
    printf 'PRIV_A=%s PRIV_B=%s PRIV_C=%s\n' "$PRIV_A" "$PRIV_B" "$PRIV_C"
    aws ec2 describe-subnets --subnet-ids "$PRIV_A" "$PRIV_B" "$PRIV_C" --region "$AWS_REGION"

Problem: instance type rejected with a Free Tier eligibility message
Cause: selected instance type did not satisfy that account's eligibility check.
Fix: use a supported instance type that is available in the required AZs and
verify its cost. The lab later used m7i-flex.large. Never choose instance type
based only on a Free Tier label for a production workload.

Problem: nodes NotReady / aws-node CrashLoopBackOff
Inspect:
    kubectl get nodes -o wide
    kubectl get pods -n kube-system -o wide
    kubectl logs -n kube-system daemonset/aws-node -c aws-node --tail=200
    kubectl describe daemonset aws-node -n kube-system
    aws eks describe-nodegroup --cluster-name "$CLUSTER_NAME" --nodegroup-name "$NODEGROUP_NAME" --region "$AWS_REGION" --output yaml

In the lab, aws-node was missing EC2 network-interface permissions. Attaching
AmazonEKS_CNI_Policy to prod-eks-node-role was used as the immediate fix. A
production-hardening step is to move CNI permissions to a dedicated IRSA/Pod
Identity role, test it, and only then remove them from the node role.

Problem: EBS CSI add-on stuck Creating / InvalidIdentityToken
Inspect:
    aws eks describe-addon --cluster-name "$CLUSTER_NAME" --addon-name aws-ebs-csi-driver --region "$AWS_REGION" --output yaml
    kubectl get pods -n kube-system | grep ebs
    kubectl logs -n kube-system deployment/ebs-csi-controller --all-containers --tail=200
    aws iam list-open-id-connect-providers
    aws iam get-role --role-name AmazonEKS_EBS_CSI_DriverRole

Check OIDC provider, trust policy, exact service-account subject, audience, role
permissions and STS connectivity. The issuer URL in the cluster description
does not by itself prove the IAM OIDC provider exists.

Problem: ALB controller returns AccessDenied
Inspect:
    kubectl logs -n kube-system deployment/aws-load-balancer-controller --tail=300
    aws iam get-role --role-name AmazonEKSLoadBalancerControllerRole
    kubectl get serviceaccount aws-load-balancer-controller -n kube-system -o yaml

The lab previously saw missing permissions for ELB Describe* operations. Compare
the policy to the official controller policy matching the installed release.
Do not solve this by attaching AdministratorAccess.

Problem: PVC Pending
Inspect:
    kubectl describe pvc <PVC_NAME> -n <NAMESPACE>
    kubectl get storageclass
    kubectl get events -A --sort-by=.lastTimestamp | tail -n 100
    kubectl logs -n kube-system deployment/ebs-csi-controller --all-containers --tail=200

WaitForFirstConsumer may intentionally delay binding until a consuming pod is
scheduled. If it remains Pending, inspect scheduler events, IAM, CSI health,
AZ topology, node capacity and subnet availability.


13. CLEAN-UP WARNING
====================

Do not run broad deletion commands in an account with existing workloads.
Deleting EKS does not automatically remove every related AWS resource in every
scenario, and NAT gateways, Elastic IPs, ALBs, EBS volumes, IAM roles/policies,
and CloudWatch log groups may remain billable.

Before deleting anything:
- inventory resources by tags and IDs;
- back up data and cluster configuration;
- confirm no workloads depend on the VPC or volumes;
- delete in dependency order;
- verify that retained EBS volumes and Elastic IPs are intentionally retained.

This runbook intentionally does not include an automated destroy script because
it is easy to delete unrelated or production resources by mistake.


14. OFFICIAL REFERENCES
=======================

Amazon EKS User Guide:
https://docs.aws.amazon.com/eks/latest/userguide/

EKS cluster IAM role:
https://docs.aws.amazon.com/eks/latest/userguide/cluster-iam-role.html

EKS node IAM role:
https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html

EKS managed add-ons:
https://docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html

IRSA / service accounts:
https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html

EKS Pod Identity:
https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html

EBS CSI driver:
https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html

AWS Load Balancer Controller:
https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html

AWS Load Balancer Controller Helm install:
https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html

EKS security best practices:
https://docs.aws.amazon.com/eks/latest/best-practices/security.html

END OF RUNBOOK
==============
