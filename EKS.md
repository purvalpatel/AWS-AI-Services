# EKS ( Elastic Kubernetes Service )
- EKs Does not manage worker nodes:
    - Self-managed nodes  - Manually provision
    - Managed Node Group  - Run EKS on optimized images, automates the EC2 nodes.
    - Fargate  -  Serverless architecture.
 

## Steps:
1. Creating an EKS Cluster.
    - Cluster name, K8s version
    - IAM Role for cluster.
      - Provisioning Nodes
      - Storage
      - Secrets
     
2. Creating worker nodes
    - NodeGroup
    - Select instance Type
    - min/max nodes
    - EKS Cluster select.
  
3. Connect to Cluster
    - Kubectl
  
## Methods to create EKS cluster.
1. AWS console
2. eksctl
3. Infrastructure as a code - Terraform/pulumi.


### Install eksctl in your local machine:
```BASH
# for ARM systems, set ARCH to: `arm64`, `armv6` or `armv7`
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

# (Optional) Verify checksum
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check

tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz

sudo install -m 0755 /tmp/eksctl /usr/local/bin && rm /tmp/eksctl
```

### Create IAM user:

> Copy Access key and Secret Key.

### install awscli:
```BASH
aws configure
```
Output:
```Log
AWS Access Key ID [None]: xxxxxxxxxxxxxxxxxx
AWS Secret Access Key [None]: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx/xxxxxxxxxxx
Default region name [None]: ap-south-1
Default output format [None]: json
```

### Create cluster:
```BASH
eksctl create cluster --help        ## for check the options
eksctl create cluster -n cluster1 --nodegroup-name ng1 --region ap-south-1 --node-type t2.micro --nodes 2

### Create Cluster with debug.
eksctl create cluster \
  --name cluster1 \
  --region ap-south-1 \
  --nodegroup-name ng1 \
  --node-type t3.medium \
  --nodes 2 \
  --verbose 5

```

> This will take 10-20 minutes. <br>
> This will automatically created: <br>
> EC2 instances, CloudFormation stack, Security groups, VPC, Subnets etc.


#### get caller identity:
```BASH
aws sts get-caller-identity
```

#### List aws credentials:
```BASH
aws configure list
```

#### Cluster will be automatically added into eksctl:
```BASH
kubectl config view

kubectl get nodes

kubectl get pods -A
```

#### Delete cluster [ Optional ]
```BASH
eksctl delete cluster   --name cluster1   --region ap-south-1
```
> Note: Deleting Stack will take upto 5-10 minutes.

## Troubleshooting:

### Issue 1:
> Some resources are created automatically: Subets, VPC, Security groups everything is created but EC2 instances are not created.

Step 1 - Verify events:
```BASH
aws cloudformation describe-stack-events \
  --stack-name eksctl-cluster1-cluster \
  --region ap-south-1 \
  --max-items 50
```

Step 2 - List node groups:
```BASH
aws eks list-nodegroups \
  --cluster-name cluster1 \
  --region ap-south-1
```
if it returns:
```BASH
{
  "nodegroups": []
}
```
- No node group has been created yet.
- This means: Control plane is still creating.


Step 3 - Check the cloud formation status:
```BASH
aws cloudformation describe-stacks \
  --stack-name eksctl-cluster1-cluster \
  --region ap-south-1 \
  --query 'Stacks[0].StackStatus'
```
OR
```BASH
aws eks describe-cluster \
  --name cluster1 \
  --region ap-south-1 \
  --query 'cluster.status'
```
