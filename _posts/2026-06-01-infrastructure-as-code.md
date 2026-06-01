---
layout: post
title: "Infrastructure as Code - Hạ tầng dưới dạng mã"
date: 2026-06-01
categories: [containers]
tags: [terraform, cloudformation, cdk, iac, infrastructure]
---

## Mục lục
1. [Góc nhìn tổng quan - Bản vẽ kỹ thuật số](#goc-nhin-tong-quan)
2. [IaC là gì và tại sao cần?](#iac-la-gi)
3. [AWS CloudFormation - Native AWS](#cloudformation)
4. [Terraform - Multi-cloud champion](#terraform)
5. [AWS CDK - Infrastructure bằng code thật](#cdk)
6. [So sánh: CloudFormation vs Terraform vs CDK](#so-sanh)
7. [State Management - Quản lý trạng thái](#state-management)
8. [Modules và Reusability](#modules)
9. [Khi nào dùng công cụ nào?](#khi-nao)
10. [Tổng kết và best practices](#tong-ket)

---

## 1. Góc nhìn tổng quan - Bản vẽ kỹ thuật số {#goc-nhin-tong-quan}

### Ví dụ đời thường

IaC giống **bản vẽ kiến trúc** cho tòa nhà:

- **Không có IaC** = xây nhà bằng miệng: "anh thợ ơi, xây cho em tường đây, cửa đó" → mỗi lần xây khác nhau, không ai nhớ cấu trúc
- **Có IaC** = có bản vẽ CAD chi tiết: ai cũng xây được giống nhau, biết chính xác cấu trúc, thay đổi phải sửa bản vẽ trước

Cụ thể:
- **CloudFormation** = bản vẽ chỉ dùng cho nhà thầu AWS (YAML/JSON mô tả resources)
- **Terraform** = bản vẽ universal (HCL language, hỗ trợ AWS + Azure + GCP + nhiều hơn)
- **CDK** = architect dùng code (TypeScript/Python) tạo bản vẽ, CDK chuyển sang CloudFormation

### Tại sao cần IaC?

```
❌ Quản lý thủ công (ClickOps):
- Click tạo VPC, EC2, RDS trong Console
- Không ai biết chính xác config
- Không reproducible (tạo lại khác)
- Không version control (không biết ai thay đổi gì)
- Dev/Staging/Prod khác nhau

✅ Infrastructure as Code:
- Code mô tả mọi resource
- Git history = audit trail
- PR review cho infrastructure changes
- Reproducible: clone environment trong phút
- Dev = Staging = Prod (same code, different params)
- Destroy/recreate anytime
```

---

## 2. IaC là gì và tại sao cần? {#iac-la-gi}

### Declarative vs Imperative

```
Declarative (WHAT - Tôi muốn gì):
  "Tôi muốn 3 EC2 instances, 1 RDS, 1 VPC"
  → Tool tự tìm cách đạt trạng thái mong muốn
  → CloudFormation, Terraform, CDK

Imperative (HOW - Làm thế nào):
  "Bước 1: Tạo VPC. Bước 2: Tạo subnet. Bước 3: Launch EC2"
  → Bạn chỉ định từng bước
  → Scripts (bash, AWS CLI), Ansible
```

### IaC Benefits

```
1. VERSION CONTROL: Infrastructure thay đổi = Git commits
2. REPRODUCIBILITY: Tạo N environments giống hệt nhau
3. DRIFT DETECTION: Phát hiện khi ai đó sửa tay
4. SELF-DOCUMENTING: Code IS the documentation
5. COST ESTIMATION: Preview cost trước khi apply
6. BLAST RADIUS: Plan → Review → Apply (safe changes)
7. COLLABORATION: PR reviews cho infra changes
```

---

## 3. AWS CloudFormation {#cloudformation}

### CloudFormation overview

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Web application infrastructure'

Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, staging, prod]
  InstanceType:
    Type: String
    Default: t3.micro

Resources:
  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      ImageId: ami-0abc123456789
      SecurityGroupIds:
        - !Ref WebSG
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-web-server'

  WebSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Web server security group
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

Outputs:
  WebServerIP:
    Value: !GetAtt WebServer.PublicIp
    Export:
      Name: !Sub '${Environment}-web-ip'
```

### CloudFormation lifecycle

```bash
# Create stack
aws cloudformation create-stack \
  --stack-name my-app \
  --template-body file://template.yaml \
  --parameters ParameterKey=Environment,ParameterValue=prod

# Update stack
aws cloudformation update-stack \
  --stack-name my-app \
  --template-body file://template.yaml

# Delete stack (destroys ALL resources)
aws cloudformation delete-stack --stack-name my-app

# Change set (preview changes before applying)
aws cloudformation create-change-set \
  --stack-name my-app \
  --change-set-name my-changes \
  --template-body file://template.yaml
```

---

## 4. Terraform {#terraform}

### Terraform overview

```hcl
# main.tf
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
  }
}

provider "aws" {
  region = var.region
}

# Variables
variable "environment" {
  type    = string
  default = "prod"
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}

# Resources
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  
  vpc_security_group_ids = [aws_security_group.web.id]
  
  tags = {
    Name        = "${var.environment}-web"
    Environment = var.environment
  }
}

resource "aws_security_group" "web" {
  name_prefix = "${var.environment}-web-"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Output
output "web_ip" {
  value = aws_instance.web.public_ip
}
```

### Terraform workflow

```bash
# Initialize (download providers)
terraform init

# Plan (preview changes)
terraform plan -out=tfplan
# Shows: + create, ~ modify, - destroy

# Apply (execute changes)
terraform apply tfplan

# Destroy (delete everything)
terraform destroy
```

---

## 5. AWS CDK {#cdk}

### CDK overview (TypeScript)

```typescript
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';

export class WebStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // VPC
    const vpc = new ec2.Vpc(this, 'WebVpc', {
      maxAzs: 2,
      natGateways: 1,
    });

    // Security Group
    const sg = new ec2.SecurityGroup(this, 'WebSG', {
      vpc,
      allowAllOutbound: true,
    });
    sg.addIngressRule(ec2.Peer.anyIpv4(), ec2.Port.tcp(80));

    // EC2 Instance
    const instance = new ec2.Instance(this, 'WebServer', {
      vpc,
      instanceType: ec2.InstanceType.of(
        ec2.InstanceClass.T3, ec2.InstanceSize.MICRO
      ),
      machineImage: ec2.MachineImage.latestAmazonLinux2023(),
      securityGroup: sg,
    });

    new cdk.CfnOutput(this, 'WebIP', {
      value: instance.instancePublicIp,
    });
  }
}
```

---

## 6. So sánh {#so-sanh}

```
┌────────────────┬──────────────────┬─────────────────┬──────────────────┐
│ Aspect         │ CloudFormation   │ Terraform       │ CDK              │
├────────────────┼──────────────────┼─────────────────┼──────────────────┤
│ Language       │ YAML/JSON        │ HCL             │ TS/Python/Java   │
│ Cloud          │ AWS only         │ Multi-cloud     │ AWS (→ CFN)      │
│ State          │ AWS managed      │ Self-managed    │ AWS managed      │
│ Learning curve │ Medium           │ Medium          │ Low (for devs)   │
│ Ecosystem      │ AWS              │ Huge (providers)│ Constructs       │
│ Drift detect   │ Yes              │ terraform plan  │ Yes (via CFN)    │
│ Rollback       │ Automatic        │ Manual          │ Automatic        │
│ Cost           │ Free             │ Free/Enterprise │ Free             │
│ Testing        │ TaskCat          │ Terratest       │ CDK assertions   │
└────────────────┴──────────────────┴─────────────────┴──────────────────┘
```

---

## 7. State Management {#state-management}

### Terraform State

```
Terraform state = source of truth:
- Maps resource IDs to config
- Tracks metadata (dependencies, attributes)
- MUST be shared in team environments
- MUST be locked during operations

Remote state backends:
- S3 + DynamoDB (AWS) ← Most common
- Azure Blob Storage
- GCS (Google Cloud Storage)
- Terraform Cloud/Enterprise
- Consul
```

```hcl
# Remote state configuration
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "prod/vpc/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"  # Locking
  }
}
```

---

## 8. Modules {#modules}

### Terraform Modules

```hcl
# modules/vpc/main.tf
variable "cidr" { type = string }
variable "environment" { type = string }

resource "aws_vpc" "main" {
  cidr_block = var.cidr
  tags = { Name = "${var.environment}-vpc" }
}

output "vpc_id" { value = aws_vpc.main.id }

# Usage:
module "vpc" {
  source      = "./modules/vpc"
  cidr        = "10.0.0.0/16"
  environment = "prod"
}

# Public registry modules:
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
  
  cidr = "10.0.0.0/16"
  azs  = ["us-east-1a", "us-east-1b"]
}
```

---

## 9-10. Khi nào dùng gì + Tổng kết {#khi-nao}

```
AWS-only, team nhỏ, muốn đơn giản → CloudFormation
Multi-cloud, team DevOps, flexibility → Terraform
Developer-focused, AWS, type-safe → CDK
Already using Terraform, want types → CDKTF
```

### Tài liệu tham khảo

| Tài liệu | Mô tả |
|-----------|--------|
| Terraform documentation | hashicorp.com/terraform |
| AWS CloudFormation docs | docs.aws.amazon.com |
| AWS CDK documentation | docs.aws.amazon.com/cdk |
| Terraform: Up & Running (O'Reilly) | Sách thực hành |

---

*Bài viết tiếp theo: [SQL Deep Dive](/2026/08/20/sql-deep-dive/)*
