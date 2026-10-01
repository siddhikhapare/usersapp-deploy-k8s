# EKS App Deployment Instructions (NLB with TLS)

The following instructions demonstrate how to provision an Amazon EKS cluster on AWS and deploy a cloud-native application into it, fronted by an Nginx Ingress Controller exposed through a Network Load Balancer (NLB) with TLS termination, a custom Route 53 domain, and an ACM certificate.
The application consists of a **Frontend**, a **Backend (API)**, and a **PostgreSQL** database running as a StatefulSet with a headless service.

## Architecture Overview

- **VPC**: `192.168.0.0/16` with 2 public + 2 private subnets across 2 AZs
- **IAM**: separate roles for the EKS control plane, the worker node group, and a bastion EC2 instance used to manage the cluster
- **Route 53**: hosted zones for the root domain and a subdomain
- **ACM**: DNS-validated certificate for the subdomain, terminated at the NLB
- **EKS Cluster** (`myapp-cluster`) with **OIDC/IRSA** enabled
- **Managed Node Group** in the private subnets
- **Bastion EC2 instance** (in a public subnet) used to run `kubectl` / `eksctl` / `aws` instead of a local machine
- **Add-ons**: VPC CNI, CoreDNS, kube-proxy, **Amazon EBS CSI Driver**
- **AWS Load Balancer Controller** (via Helm + IRSA)
- **Nginx Ingress Controller** exposed via an NLB with TLS termination
- **PostgreSQL StatefulSet + headless Service**, **Backend Deployment/Service**, **Frontend Deployment/Service**, wired together with an **Ingress**

---

## Client Tools

Tested with the following client tool versions:

- `aws-cli` 
- `eksctl` 
- `kubectl` 
- `helm` 
  
---

# STEP 1: Create the VPC

## STEP 1.1: Create the VPC

- **CIDR block**: `192.168.0.0/16` → 65,536 IP addresses, split across public/private subnets.

## STEP 1.2: Create 4 subnets across 2 AZs

| Subnet          | AZ           | CIDR              | Purpose                                            |
|------------------|-------------|-------------------|-----------------------------------------------------|
| public-subnet-1  | ap-south-1a | 192.168.1.0/24    | Internet-facing resources (NLB, Nginx ingress)      |
| public-subnet-2  | ap-south-1b | 192.168.2.0/24    | Internet-facing resources (NLB, Nginx ingress)      |
| private-subnet-1 | ap-south-1a | 192.168.3.0/24    | Worker nodes, app pods, database                    |
| private-subnet-2 | ap-south-1b | 192.168.4.0/24    | Worker nodes, app pods, database                    |

**Why public + private subnets?**
- **Security** – databases and backend services sit in private subnets, minimizing internet exposure.
- **Scalability/HA** – spreading subnets across 2 AZs increases availability and resilience.

## STEP 1.3: Create an Internet Gateway (IGW) and attach it to the VPC

## STEP 1.4: Create a NAT Gateway

- Place the NAT Gateway in a public subnet (e.g. `public-subnet-1`).
- Allocate and associate an **Elastic IP** to the NAT Gateway.
- This lets instances in the private subnets reach the internet (e.g. for pulling images from ECR, OS updates) without being reachable from the internet.

## STEP 1.5: Set up route tables

- **Public route table** (`myapp-public-route`) — associated with the 2 public subnets:
  - `0.0.0.0/0` → Internet Gateway
  - `192.168.0.0/16` → local
- **Private route table** (`myapp-private-route`) — associated with the 2 private subnets:
  - `0.0.0.0/0` → NAT Gateway
  - `192.168.0.0/16` → local

## STEP 1.6: Tag the subnets for AWS Load Balancer Controller subnet auto-discovery

The AWS Load Balancer Controller and EKS use these tags to automatically discover the right subnets for internal vs. external load balancers ([subnet auto-discovery docs](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.11/deploy/subnet_discovery/#subnet-auto-discovery)):

**Public subnets** (external/internet-facing load balancers):

```
kubernetes.io/role/elb                     = 1
kubernetes.io/cluster/myapp-cluster        = owned
Name                                       = public-subnet-1 / public-subnet-2
```

**Private subnets** (internal load balancers):

```
kubernetes.io/role/internal-elb            = 1
kubernetes.io/cluster/myapp-cluster        = owned
Name                                       = private-subnet-1 / private-subnet-2
```

> `kubernetes.io/role/elb=1` marks a subnet as usable for an **external (internet-facing)** load balancer. `kubernetes.io/role/internal-elb=1` marks a subnet as usable for an **internal** load balancer. `kubernetes.io/cluster/${cluster-name}=owned` tells AWS which subnets belong to this EKS cluster so the Load Balancer Controller can associate the correct resources.

## STEP 1.7: Verify NACLs and default Security Group

- Default NACL: allow all traffic inbound/outbound (rule 100 → Allow, `0.0.0.0/0`).
- Default VPC security group: inbound self-referencing rule, outbound all traffic to `0.0.0.0/0`.

---

# STEP 2: Create IAM for EKS, Worker Node, and EC2

## STEP 2.1: IAM role for the EKS cluster (control plane)

1. IAM → Roles → Create role.
2. Trusted entity type: **AWS service**.
3. Service or use case: **EKS** → **EKS – Cluster**.
4. Attach policy: `AmazonEKSClusterPolicy`.
5. Role name: **`myAmazonEKSClusterRole`**.

## STEP 2.2: IAM role for the Node Group (worker nodes)

1. IAM → Roles → Create role.
2. Trusted entity type: **AWS service** → **EC2**.
3. Attach policies:
   - `AmazonEC2ContainerRegistryReadOnly`
   - `AmazonEKS_CNI_Policy`
   - `AmazonEKSWorkerNodePolicy`
   - (EBS CSI driver policy is **not** attached here — it is granted separately via IRSA in Step 11.)
4. Role name: **`myEKSNodeRole`**.

## STEP 2.3: IAM role for the EC2 bastion (to access EKS/Nodes from AWS instead of a local machine)

1. IAM → Roles → Create role → trusted entity **EC2**.
2. Create an **inline policy** named `eksaccesspolicy`:

```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Action": [
            "eks:DescribeCluster",
            "eks:ListClusters",
            "eks:DescribeNodegroup",
            "eks:ListNodegroups",
            "eks:ListUpdates",
            "eks:AccessKubernetesApi"
        ],
        "Resource": "*"
    }]
}
```

3. Role name: **`EKSaccess`**.
4. Additional inline policies added to `EKSaccess` later (see Step 3) to let the bastion also manage Route 53, ACM, Load Balancer resources, and IAM roles for `eksctl`.

---

# STEP 3: Attach IAM Policy Required for the IAM User Running These Commands

The IAM **user** (`myuser`) running `aws`, `eksctl`, and `kubectl` commands needs enough privileges to create VPCs, IAM roles, EKS clusters, EC2 instances, load balancers, Route 53 records, and ACM certificates.

## STEP 3.1: Attach AWS managed policies to the IAM user

```
AmazonEC2FullAccess
AmazonRoute53FullAccess
AmazonRoute53DomainsFullAccess
IAMReadOnlyAccess
AWSCertificateManagerFullAccess
```

> If these aren't sufficient, `AdministratorAccess` can be used directly for a demo/lab environment.

## STEP 3.2: Custom inline policy for the IAM user (`IAM-eks`) — Route 53 + IAM role management

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets",
        "route53:ListTagsForResource",
        "route53:UpdateHostedZoneComment",
        "route53:GetHostedZone",
        "route53:DeleteHostedZone",
        "route53:GetHostedZoneCount",
        "route53:ListHostedZonesByName",
        "route53:CreateHostedZone",
        "route53:ChangeResourceRecordSets",
        "route53:CreateHealthCheck",
        "route53:DeleteHealthCheck",
        "route53:UpdateHealthCheck"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "iam:PutRolePolicy",
        "iam:GetRole",
        "iam:ListAttachedRolePolicies",
        "iam:ListRoles"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster",
        "eks:ListClusters"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "cloudformation:CreateStack",
        "cloudformation:UpdateStack",
        "cloudformation:DescribeStacks",
        "cloudformation:ListStacks"
      ],
      "Resource": "*"
    }
  ]
}
```

## STEP 3.3: Additional inline policies on the `EKSaccess` role (bastion) — used when running `eksctl`/`aws` on the EC2 instance itself

**`EKSloadbalancerPolicy`** — lets the bastion inspect/describe LB resources:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "elasticloadbalancing:CreateLoadBalancer",
        "elasticloadbalancing:DescribeLoadBalancers",
        "elasticloadbalancing:DescribeTargetGroups",
        "elasticloadbalancing:DescribeListeners",
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeVpcs",
        "ec2:DescribeInstances"
      ],
      "Resource": "*"
    }
  ]
}
```

**`EKSLoadBalancerControllerAccess`** — lets the bastion create the IAM role/policy/CloudFormation stacks needed to install the AWS Load Balancer Controller:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iam:AttachRolePolicy",
      "Resource": "arn:aws:iam::<AWS_ACCOUNT_ID>:role/EKSaccess"
    },
    { "Effect": "Allow", "Action": "iam:GetOpenIDConnectProvider", "Resource": "*" },
    { "Effect": "Allow", "Action": "iam:CreatePolicy", "Resource": "*" },
    {
      "Effect": "Allow",
      "Action": "cloudformation:ListStacks",
      "Resource": "arn:aws:cloudformation:<AWS_REGION>:<AWS_ACCOUNT_ID>:stack/*/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "cloudformation:CreateStack",
        "cloudformation:DescribeStacks",
        "cloudformation:UpdateStack",
        "cloudformation:DeleteStack",
        "cloudformation:DescribeStackResources"
      ],
      "Resource": "*"
    }
  ]
}
```

**`cert-manager-Route53-policy`** — lets the bastion manage DNS records directly (for ACM validation / ingress DNS records) via `eksctl`/`aws` CLI:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:GetChange",
        "route53:ChangeResourceRecordSets",
        "route53:ListResourceRecordSets",
        "route53:ListHostedZones",
        "route53:ListHostedZonesByName"
      ],
      "Resource": "*"
    }
  ]
}
```

---

# STEP 4: Create Route 53 Hosted Zone for Root and Sub Domain

## STEP 4.1: Create a hosted zone for the root domain

- Domain name: `ksiddhiapphub.org`
- Type: **Public hosted zone**

## STEP 4.2: Point the registrar at the root zone's name servers

The domain was registered outside Route 53 (Squarespace), so the `.org` registry currently sends queries to Squarespace's name servers. Route 53 has to be made authoritative:

1. In the registrar's **Domain Nameservers** settings, choose custom nameservers.
2. Enter the 4 NS values from the **root** zone (Step 4.1), one per entry.
3. Save.

Do this immediately after creating the root zone. Propagation can take from a few minutes to a few hours, and ACM validation (Step 5) cannot succeed until it completes.

> **Only the root zone's NS values go to the registrar.** The subdomain zone's NS values (Step 4.3) never go to the registrar.

Steps 4.2 and 4.4 must be working before ACM can validate. If they aren't, the certificate stays in **Pending validation**.

## STEP 4.3: Create a hosted zone for the subdomain

- Domain name: `demo.ksiddhiapphub.org`
- Type: **Public hosted zone**

## STEP 4.4: Delegate the subdomain from the root zone

In the **root** hosted zone (`ksiddhiapphub.org`), create an **NS record** named `demo` whose values are the 4 NS records from the subdomain's hosted zone (Step 4.3). This makes `demo.ksiddhiapphub.org` resolve through its own hosted zone.

Verify propagation:

```bash
dig NS ksiddhiapphub.org +short
dig NS demo.ksiddhiapphub.org +short
```

---

# STEP 5: Create an ACM Certificate for the Subdomain

## STEP 5.1: Request a public certificate

1. ACM → Request certificate → **Request a public certificate**.
2. Fully qualified domain name: `demo.ksiddhiapphub.org`.
3. Validation method: **DNS validation** (recommended).
4. Key algorithm: **RSA 2048**.

## STEP 5.2: Create the DNS validation record

ACM shows a **Create DNS records in Amazon Route 53** — select the domain and click **Create records**. This drops a CNAME validation record directly into the `demo.ksiddhiapphub.org` hosted zone.
Once **Issued**, note the certificate ARN — it's used later to configure TLS termination on the Nginx Ingress Controller's NLB.

---

# STEP 6: Create the EKS Cluster

## STEP 6.1: Cluster configuration

- **Name**: `myapp-cluster`
- **Cluster IAM role**: `myAmazonEKSClusterRole` (from Step 2.1)
- **Kubernetes version**: `1.31`
- **Cluster access**:
  - Bootstrap cluster administrator access: **Allow**
  - Authentication mode: **EKS API and ConfigMap** (supports both the modern EKS access-entry API and the legacy `aws-auth` ConfigMap)

> By default EKS authenticates IAM principals via the EKS API / IAM Authenticator. The `aws-auth` ConfigMap maps IAM roles/users to Kubernetes users/groups so you can grant cluster access (e.g. `system:masters`) to specific IAM roles.

## STEP 6.2: Networking

- **VPC**: the VPC created in Step 1 (`192.168.0.0/16`).
- **Subnets**: select all 4 subnets (2 public + 2 private) — this lets EKS place elastic network interfaces (ENIs) for the control plane in your VPC.
- **Additional security groups**: the default VPC security group.
- **Cluster IP address family**: IPv4.
- **Cluster endpoint access**: **Public** (accessible from outside the VPC, e.g. from a local machine or an EC2 instance).

## STEP 6.3: Add-ons

Select the following (all "Ready to install"):

- **Amazon VPC CNI** — gives pods native VPC networking with private IPs, so pods can communicate with each other and external resources securely.
- **CoreDNS** — provides DNS resolution for services and pods in the cluster.
- **kube-proxy** — maintains network rules for pod-to-pod/service communication.

> The **Amazon EBS CSI Driver** add-on is installed separately in Step 11, after its IAM role is created, because it needs an IRSA-backed service account.

> **Note**: if CoreDNS shows status **Degraded** right after cluster creation, this is expected — it will move to **Running** once the worker nodes (Step 8) are up.

Review the summary and click **Create**. Cluster creation takes several minutes.

## STEP 6.4: Point kubectl at the cluster (temporarily, from wherever you're running commands)

```bash
aws eks update-kubeconfig --name myapp-cluster --region ap-south-1
kubectl config view
kubectl config get-contexts
kubectl config current-context
```

---

# STEP 7: Create the OIDC Provider

When an EKS cluster is created, AWS automatically generates an **OIDC issuer URL** for it (visible under the cluster's **Access** tab, or via `aws eks describe-cluster`), but the **IAM OIDC identity provider** itself must be created separately so that Kubernetes service accounts can assume IAM roles (IRSA).

## STEP 7.1: Get the OIDC issuer URL

```bash
aws eks describe-cluster --name myapp-cluster --region ap-south-1 \
  --query "cluster.identity.oidc.issuer" --output text
```

Example: `https://oidc.eks.ap-south-1.amazonaws.com/id/D66E0EA871C8A8DC0F81BDD028507C6D`

## STEP 7.2: Create the OIDC identity provider (console)

1. IAM → Identity providers → **Add provider**.
2. Provider type: **OpenID Connect**.
3. Provider URL: the issuer URL from Step 7.1.
4. Audience: `sts.amazonaws.com` (the default audience for EKS).

## STEP 7.2 (alternative): Create it with eksctl

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster myapp-cluster \
  --region ap-south-1 \
  --approve
```

> Without IRSA, AWS IAM has no way to recognize Kubernetes workloads — every add-on that needs AWS API access (EBS CSI driver, AWS Load Balancer Controller, etc.) authenticates through an IAM role tied to a Kubernetes service account via this OIDC provider.

---

# STEP 8: Create the Node Group

## STEP 8.1: Add node group

EKS → Clusters → `myapp-cluster` → Compute → **Add node group**.

- **Name**: `NodeGroup`
- **Node IAM role**: `myEKSNodeRole` (from Step 2.2)

## STEP 8.2: Compute and scaling configuration

- **AMI type**: Amazon Linux 2023 (x86_64) Standard
- **Capacity type**: On-Demand
- **Instance types**: `t2.medium` (or `t3.medium`)
- **Disk size**: 20 GiB

### Scaling configuration

- **Desired size**: 2 nodes
- **Minimum size**: 2 nodes
- **Maximum size**: 2 nodes

> **Desired/min/max** let the cluster scale within a controlled range based on demand while keeping costs predictable, and ensure at least one node is always available for scheduling.

### Update configuration

- **Maximum unavailable**: 1 node — during a rolling node group update, only one node is taken down at a time, minimizing disruption to running workloads.

### Node auto repair

- Enable **node auto repair** so EKS automatically detects and replaces unhealthy nodes.

## STEP 8.3: Networking

- **Subnets**: choose the **2 private subnets** (`private-subnet-1`, `private-subnet-2`).

> Worker nodes go in private subnets so they aren't directly reachable from the internet. They still reach the internet outbound (image pulls, updates) through the NAT Gateway in the public subnet.

## STEP 8.4: Review and create

Click **Create**. After a few minutes:

```bash
kubectl get nodes
kubectl get nodes -o wide
```

---

# STEP 9: Create an EC2 Instance to Access EKS/Nodes from AWS (Bastion Host)

Rather than configuring `kubectl`/`eksctl`/`aws` on a local machine, launch an EC2 instance inside AWS to manage the cluster.

## STEP 9.1: Launch the instance

- **Name**: `myserver`
- **AMI**: Ubuntu Server 24.04 LTS
- **Instance type**: `t2.micro`
- **Key pair**: create a new key pair (e.g. `apsouthkey`, RSA, `.pem`)
- **Network settings**:
  - VPC: the VPC created in Step 1
  - Subnet: a **public** subnet
  - Auto-assign public IP: **Enable**
  - Security group: allow **SSH (22)**, **HTTP (80)**, **HTTPS (443)** from `0.0.0.0/0` (or restrict SSH to your IP)
- **Storage**: 8 GiB gp3
- **Advanced details → IAM instance profile**: attach the **`EKSaccess`** role/instance profile created in Step 2.3.

## STEP 9.2: Connect to the instance

```bash
ssh -i ".pem file" ubuntu@<BASTION_PUBLIC_IP>
```

---

# STEP 10: Installation of AWS CLI, eksctl, and kubectl on EC2

Run the following **on the bastion EC2 instance**.

## STEP 10.1: Install AWS CLI v2

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

## STEP 10.2: Install kubectl

```bash
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.31.3/2024-12-12/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo cp ./kubectl /usr/local/bin
export PATH=/usr/local/bin:$PATH
kubectl version --client
```

## STEP 10.3: Install eksctl

Follow [eksctl.io/installation/#for-unix](https://eksctl.io/installation/#for-unix), then confirm:

```bash
eksctl version
```

## STEP 10.4: Install git and clone the app repo

```bash
sudo apt update
sudo apt install git -y
git clone https://github.com/siddhikhapare/Users-management-app.git
```

## STEP 10.5: Configure kubeconfig on the bastion

```bash
aws configure
# AWS Access Key ID [None]: <your-access-key>
# AWS Secret Access Key [None]: <your-secret-key>
# Default region name [None]: ap-south-1
# Default output format [None]: json

aws eks update-kubeconfig --name myapp-cluster --region ap-south-1
kubectl get ns
kubectl get nodes
```

> **Cross-region gotcha**: if your local `aws configure` used credentials/region for a different region than your cluster (e.g. `us-east-1` credentials but the cluster is in `ap-south-1`), the `aws-auth` ConfigMap combined with the AWS CLI's region flag is what allows secure, cross-region management of the cluster — always pass `--region ap-south-1` explicitly with `aws eks update-kubeconfig`.

If you see `error: You must be logged in to the server (Unauthorized)`, the bastion's IAM role isn't yet mapped into the cluster's `aws-auth` ConfigMap — fix this in the next section.

## STEP 10.6: Map the bastion's IAM role into `aws-auth` (run from a machine/CloudShell that already has admin access)

```bash
aws eks update-kubeconfig --name myapp-cluster --region ap-south-1
kubectl edit configmap aws-auth --namespace kube-system
```

Ensure `mapRoles` contains both the node role and the bastion access role:

```yaml
apiVersion: v1
data:
  mapRoles: |
    - groups:
        - system:bootstrappers
        - system:nodes
      rolearn: arn:aws:iam::<AWS_ACCOUNT_ID>:role/myEKSNodeRole
      username: system:node:{{EC2PrivateDNSName}}
    - rolearn: arn:aws:iam::<AWS_ACCOUNT_ID>:role/EKSaccess
      username: EKSaccess
      groups:
        - system:masters
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
```

- The `myEKSNodeRole` mapping to `system:nodes` lets worker nodes join the cluster and talk to the API server.
- The `EKSaccess` mapping to `system:masters` gives the bastion (and anything assuming that role) full administrative access to the cluster.

Save and exit (`Esc` → `:wq!`). Back on the bastion:

```bash
kubectl get nodes
```

Nodes should now show `Ready`.

---

# STEP 11: Install the EBS CSI Driver

## STEP 11.1: Create an IAM role for the EBS CSI Driver via IRSA

```bash
export AWS_REGION=ap-south-1

eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster myapp-cluster \
  --role-name ebs-csi-driver-role \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve
```

> `--role-only` can be used instead if you plan to add the EBS CSI Driver as a console/eksctl **addon** (which manages the service account itself) rather than letting `eksctl create iamserviceaccount` create the service account.

If needed, manually annotate the service account with the role ARN:

```bash
kubectl annotate serviceaccount ebs-csi-controller-sa \
  --namespace kube-system \
  eks.amazonaws.com/role-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:role/ebs-csi-driver-role
```

## STEP 11.2: Install the EBS CSI Driver add-on

**Using eksctl:**

```bash
eksctl create addon \
  --cluster myapp-cluster \
  --region ap-south-1 \
  --name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::<AWS_ACCOUNT_ID>:role/ebs-csi-driver-role
```

**Or, from the AWS Console:** EKS → Clusters → `myapp-cluster` → Add-ons → **Get more add-ons** → select **Amazon EBS CSI Driver** → Add-on access → **IAM roles for service accounts (IRSA)** → select `ebs-csi-driver-role`.

**Or, with EKS Pod Identity instead of IRSA** (newer alternative — no OIDC trust policy needed): add the **Amazon EKS Pod Identity Agent** add-on first, then add the **Amazon EBS CSI Driver** add-on with **Add-on access → EKS Pod Identity** and select/create the matching role. See:
- https://eksctl.io/usage/pod-identity-associations/
- https://aws.amazon.com/blogs/containers/amazon-eks-pod-identity-a-new-way-for-applications-on-eks-to-obtain-iam-credentials/

## STEP 11.3: Verify the EBS CSI Driver

```bash
kubectl get crds
kubectl get sa -n kube-system | grep ebs
kubectl describe sa ebs-csi-controller-sa -n kube-system
kubectl get pods -n kube-system | grep ebs
kubectl get storageclass
```

You should see a `gp2` (default) StorageClass provisioned by `kubernetes.io/aws-ebs`.

## STEP 11.4 (optional): Add a `gp3` StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

Then reference `storageClassName: gp3` in your StatefulSet's `volumeClaimTemplates`.

## Troubleshooting: re-run the IAM service account

If something goes wrong and you need to redo it:

```bash
eksctl delete iamserviceaccount \
  --cluster=myapp-cluster \
  --namespace=kube-system \
  --name=ebs-csi-controller-sa \
  --region=ap-south-1
```

---

# STEP 12: Install the AWS Load Balancer Controller
References: - [AWS Load Balancer Controller with Helm](https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html)
            - [AWS Load Balancer Controller Installation Guide](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.11/deploy/installation/)

## STEP 12.1: Download and create the IAM policy

```bash
curl -o iam-policy.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy.json
```

## STEP 12.2: Create the IAM role + Kubernetes service account (IRSA)

```bash
eksctl create iamserviceaccount \
  --cluster=myapp-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --region=ap-south-1 \
  --approve
```

This creates a CloudFormation stack (e.g. `eksctl-myapp-cluster-addon-iamserviceaccount-kube-system-aws-load-balancer-controller`) containing the IAM role for the service account.

## STEP 12.3: Install Helm and the controller chart

```bash
sudo snap install helm --classic
helm version

helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
helm search repo eks
```

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=myapp-cluster \
  --set vpcId=<VPC_ID> \
  --set region=ap-south-1 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

### Troubleshooting: "ServiceAccount ... exists and cannot be imported"

If Helm complains that the `aws-load-balancer-controller` ServiceAccount already exists (created by `eksctl` rather than Helm), patch its labels/annotations so Helm can adopt it, then re-run the install:

```bash
kubectl label serviceaccount aws-load-balancer-controller -n kube-system \
  app.kubernetes.io/managed-by=Helm --overwrite

kubectl annotate serviceaccount aws-load-balancer-controller -n kube-system \
  meta.helm.sh/release-name=aws-load-balancer-controller --overwrite

kubectl annotate serviceaccount aws-load-balancer-controller -n kube-system \
  meta.helm.sh/release-namespace=kube-system --overwrite
```
### Verify k8s Service Account -

```bash
# Describe Service Account alb-ingress-controller 
kubectl describe sa aws-load-balancer-controller -n kube-system
```

Then re-run the `helm install` command above.

## STEP 12.4: Verify the installation

```bash
kubectl get pods -n kube-system
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl get crds
```

You should see CRDs such as `targetgroupbindings.elbv2.k8s.aws`, `ingressclassparams.elbv2.k8s.aws`, etc.

### Troubleshooting: re-run the IAM service account

```bash
eksctl delete iamserviceaccount \
  --cluster=myapp-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --region=ap-south-1
```

---

# STEP 13: Install the Nginx Ingress Controller (NLB with TLS)

References: [ingress-nginx AWS deploy docs](https://kubernetes.github.io/ingress-nginx/deploy/#aws)  ·  [AWS LB Controller service annotations](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.11/guide/service/annotations/)

## STEP 13.1: Download the NLB-with-TLS-termination manifest

```bash
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0/deploy/static/provider/aws/nlb-with-tls-termination/deploy.yaml
```

## STEP 13.2: Edit `deploy.yaml`

```bash
sudo vim deploy.yaml
```

**In the `ingress-nginx-controller` ConfigMap**, set:
In the downloaded deploy.yaml, find the ingress-nginx-controller ConfigMap. The upstream file ships with a placeholder (XXX.XXX.XXX/XX) for proxy-real-ip-cidr. Replace it with the CIDR block of the VPC the cluster runs in. For this project that is 192.168.0.0/16 : 

```yaml
apiVersion: v1
kind: ConfigMap
data:
  http-snippet: |
    server {
      listen 2443;
      return 308 https://$host$request_uri;
    }
  proxy-real-ip-cidr: "192.168.0.0/16"
  use-forwarded-headers: "true"
metadata:
  ...
```

**In the `ingress-nginx-controller` Service**, set the annotations to point at your ACM certificate and make it an internet-facing NLB, targeting pods by IP:

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: "60"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:ap-south-1:<AWS_ACCOUNT_ID>:certificate/<cert-id>"
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "https"
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  externalTrafficPolicy: Local
```

## STEP 13.3: Apply the manifest

```bash
kubectl apply -f deploy.yaml
```

This creates the `ingress-nginx` namespace, RBAC, the admission webhook job, the `IngressClass` named `nginx`, and — most importantly — the **`ingress-nginx-controller`** Service of `type: LoadBalancer`, which the AWS Load Balancer Controller / in-tree provider turns into an internet-facing **Network Load Balancer**.

## STEP 13.4: Get the NLB's DNS name

```bash
kubectl get svc -n ingress-nginx
```

```
NAME                                 TYPE           EXTERNAL-IP
ingress-nginx-controller             LoadBalancer   k8s-ingressn-ingressn-00000.elb.ap-south-1.amazonaws.com
ingress-nginx-controller-admission   ClusterIP      <none>
```

> In the AWS console under **EC2 → Load Balancers** you'll see this NLB (`k8s-ingressn-ingressn-...`) with 2 listeners: **TCP:80** (forwarded to the HTTP target group) and **TLS:443** (using the ACM certificate, forwarded to the HTTPS target group).

---

# STEP 14: Deploy the App and Run It

## STEP 14.1: PostgreSQL — Secret

```bash
kubectl create secret generic my-postgresql-credentials \
  --from-literal=password='your-passwd' \
  --from-literal=username='your-user' \
  --dry-run=client -o yaml | kubectl apply -f -
```

## STEP 14.2: PostgreSQL — headless Service

```bash
kubectl apply -f postgres-svc.yaml
kubectl get svc
```

A **headless service** (`clusterIP: None`) gives each pod in the StatefulSet a stable, predictable DNS name, e.g.:

```
postgres-0.postgres-svc.default.svc.cluster.local
postgres-1.postgres-svc.default.svc.cluster.local
```

## STEP 14.3: PostgreSQL — StatefulSet

```bash
kubectl apply -f statefulset.yaml
kubectl get pods -w
```

StatefulSets give PostgreSQL:
- **Stable persistent storage** — each pod is bound to its own PersistentVolume via a PersistentVolumeClaim (auto-created and bound), so data survives pod restarts/rescheduling.
- **Stable network identity** — predictable per-pod DNS names.
- **Orderly deployment/scaling** — pods are created/removed one at a time, in order.
- **Consistent naming** — a restarted pod reattaches to the same PersistentVolume, preserving database integrity.

Verify storage:

```bash
kubectl get pv,pvc
kubectl get storageclass   # gp2 (default) — see Step 11.4 for gp3
```

## STEP 14.4: Initialize the database

```bash
kubectl exec -it postgres-0 -- /bin/sh
psql -U myuser -d users_database
```

```sql
GRANT ALL PRIVILEGES ON DATABASE users_database TO myuser;

CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  email VARCHAR(255) NOT NULL UNIQUE
);

\dt
\d users
\du
SELECT * FROM users;
\q
```

```bash
exit
```

## STEP 14.5: Deploy the Backend and Frontend

`deployment.yaml` (for the frontend; the backend follows the same pattern):

> Environment variables set in the Deployment manifest **override** any values baked into the Docker image at build time. Prefer a `ConfigMap`/`Secret` over hardcoding values directly.

```bash
kubectl apply -f deployment.yaml
kubectl get deploy
kubectl get pods -o wide
```

## STEP 14.6: Expose the Backend and Frontend with Services

`service.yaml`:

```bash
kubectl apply -f service.yaml
kubectl get svc
```

## STEP 14.7: Create the Ingress

```bash
kubectl apply -f nginx-ingress.yaml
kubectl get ingressclass
kubectl get ing
```

```
NAME                CLASS   HOSTS                    ADDRESS                                                       PORTS
pern-app-ingress    nginx   demo.ksiddhiapphub.org   k8s-ingressn-ingressn-....elb.ap-south-1.amazonaws.com   80
```

## STEP 14.8: Verify everything is running

```bash
kubectl get pods -A
kubectl get svc -A
kubectl get pods,pv,pvc
```

Curl the backend/frontend LoadBalancers (or the Ingress host once DNS is set in Step 15) directly to sanity check:

```bash
curl -i http://<frontend-lb-hostname>/
curl -i http://<backend-lb-hostname>/users
```

---

# STEP 15: Add a Route 53 DNS Record Pointing to the Nginx Ingress Controller's NLB

Once you have the Nginx Ingress Controller's Service (`kubectl get svc -n ingress-nginx`) and its NLB hostname:

## STEP 15.1: Create the record (console)

Route 53 → Hosted zones → `demo.ksiddhiapphub.org` → **Create record**:

- **Record name**: leave blank (records at the zone apex) or set the appropriate subdomain
- **Record type**: `A – Routes traffic to an IPv4 address and some AWS resources`
- **Alias**: **On**
- **Route traffic to**: **Alias to Network Load Balancer**
- **Region**: Asia Pacific (Mumbai) / `ap-south-1`
- Select the NLB: `k8s-ingressn-ingressn-...elb.ap-south-1.amazonaws.com`
- **Routing policy**: Simple routing
- **Evaluate target health**: Yes

# Cleanup

```bash
kubectl delete ingress pern-app-ingress
kubectl delete -f service.yaml -f deployment.yaml -f statefulset.yaml -f postgres-svc.yaml

helm uninstall aws-load-balancer-controller -n kube-system
kubectl delete -f deploy.yaml   # nginx-ingress

eksctl delete iamserviceaccount --cluster=myapp-cluster --namespace=kube-system --name=aws-load-balancer-controller --region=ap-south-1
eksctl delete iamserviceaccount --cluster=myapp-cluster --namespace=kube-system --name=ebs-csi-controller-sa --region=ap-south-1

eksctl delete nodegroup --cluster myapp-cluster --region ap-south-1 --name NodeGroup
eksctl delete cluster --name myapp-cluster --region ap-south-1

aws acm delete-certificate --certificate-arn $CERT_ARN --region ap-south-1
aws route53 delete-hosted-zone --id $SUB_ZONE_ID

# Terminate the bastion EC2 instance ("myserver") and release its Elastic IP if any
# Delete the NAT Gateway, release its Elastic IP, delete the VPC/subnets/route tables/IGW
# Remove the IAM roles/policies created in Steps 2-3 if no longer needed
```


## Architectural diagram -

![image](https://github.com/siddhikhapare/images-for-doc/blob/main/k8s-image.png)  

<br><br>

![output](https://github.com/siddhikhapare/images-for-doc/blob/main/users-mgmt-output.PNG)





