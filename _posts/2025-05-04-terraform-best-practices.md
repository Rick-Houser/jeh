---
title: "Terraform Best Practices for AWS Infrastructure"
description: "Best practices for provisioning AWS infrastructure with Terraform, focusing on modular design, remote backends, and state locking."
date: 2025-05-04 08:00:00 -0700
categories: [Automation, Infrastructure as Code]
tags: [terraform, aws, eks, modular design, remote backend, state locking, s3, dynamodb]
---

This guide outlines best practices for managing AWS infrastructure using Terraform, focusing on modular design, remote backend setup, and state locking. It provisions an EKS cluster and supporting resources for an e-commerce application.

## Prerequisites
- Terraform installed on an EC2 instance or local machine.
- AWS account with IAM permissions for EKS, S3, and DynamoDB.
- AWS CLI configured with credentials. https://aws.amazon.com/cli/

## Step 1: Organizing Terraform Code
Structure the project for scalability:

```
ecommerce-infra/
├── environments/
│   ├── prod/
│   │   └── main.tf
│   └── dev/
│       └── main.tf
├── modules/
│   ├── eks/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── vpc/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── backend.tf
```

- **Modules**: Reusable code for EKS and VPC.
- **Environments**: Separate configurations for prod and dev.
- **Separation**: Avoid monorepos; each microservice’s infrastructure can have its own repository.

**Screenshot Suggestion**: Directory tree of the Terraform project structure.

## Step 2: Creating Modular Terraform
Define a VPC module:
```hcl
# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags = {
    Name = "${var.environment}-vpc"
  }
}

resource "aws_subnet" "public" {
  count             = length(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]
  tags = {
    Name = "${var.environment}-public-${count.index}"
  }
}

variable "vpc_cidr" {}
variable "public_subnet_cidrs" { type = list(string) }
variable "availability_zones" { type = list(string) }
variable "environment" {}
```

Define an EKS module:
```hcl
# modules/eks/main.tf
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "19.15.3"

  cluster_name    = "${var.environment}-cluster"
  cluster_version = "1.32"
  vpc_id          = var.vpc_id
  subnet_ids      = var.subnet_ids
}
```

Use in an environment:
```hcl
# environments/prod/main.tf
module "vpc" {
  source = "../../modules/vpc"

  vpc_cidr           = "10.0.0.0/16"
  public_subnet_cidrs = ["10.0.1.0/24", "10.0.2.0/24"]
  availability_zones = ["us-west-2a", "us-west-2b"]
  environment        = "prod"
}

module "eks" {
  source = "../../modules/eks"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.public_subnet_ids
  environment = "prod"
}
```

- **Modularity**: Reuses VPC and EKS configurations across environments.
- **Variables**: Parameterizes inputs for flexibility.
- **Outputs**: Exposes IDs for cross-module references.

**Screenshot Suggestion**: Code snippet of the EKS module in a code editor.

## Step 3: Setting Up a Remote Backend
Configure an S3 backend with DynamoDB locking:
```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "ecommerce-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "terraform-locks"
  }
}
```

Create the S3 bucket and DynamoDB table:
```hcl
resource "aws_s3_bucket" "state" {
  bucket = "ecommerce-terraform-state"
  versioning {
    enabled = true
  }
  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }
}

resource "aws_dynamodb_table" "locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

- **Remote Backend**: Stores state in S3 for team access.
- **State Locking**: DynamoDB prevents concurrent modifications.
- **Security**: Enables encryption and versioning.

Apply:
```bash
terraform init
terraform apply
```

**Screenshot Suggestion**: AWS console showing the S3 bucket and DynamoDB table configurations.

## Best Practices
- **Encryption**: Use server-side encryption for S3 state files.
- **Access Control**: Restrict bucket access with IAM policies.
- **Versioning**: Enable S3 versioning for state recovery.
- **Modular Design**: Break infrastructure into reusable modules.

## Takeaways
Modular Terraform, combined with a secure remote backend and state locking, ensures scalable and collaborative infrastructure management. The next post will cover setting up CI/CD pipelines for deploying microservices.