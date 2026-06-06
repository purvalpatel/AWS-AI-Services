infrastructure as a code service for AWS. <br>

Instead of creating manual resources, cloudFormation creates it for you.

## Difference between Terraform and CloudFormation:

| Terraform                         | CloudFormation                    |
| --------------------------------- | --------------------------------- |
| Infrastructure as Code (IaC) tool | Infrastructure as Code (IaC) tool |
| Multi-cloud                       | AWS only                          |
| Uses HCL language                 | Uses JSON/YAML                    |
| Created by HashiCorp              | Created by AWS                    |
| CLI-driven                        | CLI + AWS Console                 |
| Stores state in tfstate           | State managed by CloudFormation   |

Think them as a competetors. <br>

### Without CloudFormation:
```
VPC
  |
Subnets 
  |
Internet Gateway
  |
Route Tables
  |
Security Group
  |
EC2
  |
Load balancer
  |
EKS
```

### with CloudFormation:
```
MyVPC:
  Type:AWS:EC2:VPC
MyCluster:
  Type:AWS:EKS:Cluster
```
## How it works ?

When you create an EKS cluster using:
```
eksctl create cluster
```

eksctl actually generates CloudFormation templates behind the scenes. <br>

Architecture:
```
eksctl
   ↓
CloudFormation Stack
   ↓
VPC
Subnets
Security Groups
IAM Roles
EKS Cluster
Node Groups
```

## Sample Setup:
Create `ec2-demo.yaml`
```YAML
AWSTemplateFormatVersion: '2010-09-09'
Description: Simple EC2 Demo

Resources:
  MyEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t2.micro
      ImageId: ami-0f58b397bc5c1f2e8   # Amazon Linux 2023 (ap-south-1)
      Tags:
        - Key: Name
          Value: CloudFormation-Demo
```

Create Stack:
```
aws cloudformation create-stack \
  --stack-name demo-ec2-stack \
  --template-body file://ec2-demo.yaml \
  --region ap-south-1
```
Watch Progress:
```
aws cloudformation describe-stacks \
  --stack-name demo-ec2-stack \
  --region ap-south-1
```

Delete Stack ( Optional ):
```
aws cloudformation delete-stack \
  --stack-name demo-ec2-stack \
  --region ap-south-1
```
