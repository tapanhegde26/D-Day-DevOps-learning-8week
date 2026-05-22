# Week 2, Day 1-2: Terraform for EKS Fundamentals

## Why Terraform for EKS?

Infrastructure as Code (IaC) provides:
- **Reproducibility**: Same infrastructure every time
- **Version Control**: Track changes in Git
- **Collaboration**: Team can review infrastructure changes
- **Automation**: CI/CD for infrastructure
- **Documentation**: Code is the documentation

## EKS Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              AWS REGION                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                              VPC                                        │ │
│  │                                                                         │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │ │
│  │  │                    Availability Zone A                           │  │ │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │  │ │
│  │  │  │   Public    │  │   Private   │  │      Private            │ │  │ │
│  │  │  │   Subnet    │  │   Subnet    │  │      Subnet (DB)        │ │  │ │
│  │  │  │             │  │             │  │                         │ │  │ │
│  │  │  │  NAT GW     │  │  EKS Nodes  │  │      RDS Instance       │ │  │ │
│  │  │  │  ALB        │  │             │  │                         │ │  │ │
│  │  │  └─────────────┘  └─────────────┘  └─────────────────────────┘ │  │ │
│  │  └─────────────────────────────────────────────────────────────────┘  │ │
│  │                                                                         │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │ │
│  │  │                    Availability Zone B                           │  │ │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │  │ │
│  │  │  │   Public    │  │   Private   │  │      Private            │ │  │ │
│  │  │  │   Subnet    │  │   Subnet    │  │      Subnet (DB)        │ │  │ │
│  │  │  │             │  │             │  │                         │ │  │ │
│  │  │  │  NAT GW     │  │  EKS Nodes  │  │      RDS Standby        │ │  │ │
│  │  │  │  ALB        │  │             │  │                         │ │  │ │
│  │  │  └─────────────┘  └─────────────┘  └─────────────────────────┘ │  │ │
│  │  └─────────────────────────────────────────────────────────────────┘  │ │
│  │                                                                         │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐ │ │
│  │  │                    EKS Control Plane (AWS Managed)                │ │ │
│  │  │    API Server │ etcd │ Controller Manager │ Scheduler            │ │ │
│  │  └──────────────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Terraform Project Structure

```
terraform/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   └── ...
│   └── prod/
│       └── ...
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── eks/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── rds/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── README.md
```

---

## VPC Module

### VPC Configuration

```hcl
# modules/vpc/main.tf

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

data "aws_availability_zones" "available" {
  state = "available"
}

locals {
  azs = slice(data.aws_availability_zones.available.names, 0, 3)
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = var.vpc_name
  cidr = var.vpc_cidr

  azs             = local.azs
  private_subnets = var.private_subnets
  public_subnets  = var.public_subnets
  intra_subnets   = var.database_subnets

  # NAT Gateway for private subnet internet access
  enable_nat_gateway     = true
  single_nat_gateway     = var.environment == "dev" ? true : false
  one_nat_gateway_per_az = var.environment == "prod" ? true : false

  # DNS settings
  enable_dns_hostnames = true
  enable_dns_support   = true

  # Tags required for EKS
  public_subnet_tags = {
    "kubernetes.io/role/elb"                      = 1
    "kubernetes.io/cluster/${var.cluster_name}"   = "shared"
  }

  private_subnet_tags = {
    "kubernetes.io/role/internal-elb"             = 1
    "kubernetes.io/cluster/${var.cluster_name}"   = "shared"
  }

  tags = {
    Environment = var.environment
    Terraform   = "true"
    Project     = var.project_name
  }
}

# VPC Endpoints for private EKS access
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = module.vpc.vpc_id
  service_name = "com.amazonaws.${var.region}.s3"
  
  route_table_ids = module.vpc.private_route_table_ids

  tags = {
    Name        = "${var.vpc_name}-s3-endpoint"
    Environment = var.environment
  }
}

resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = module.vpc.vpc_id
  service_name        = "com.amazonaws.${var.region}.ecr.api"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = module.vpc.private_subnets
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = {
    Name        = "${var.vpc_name}-ecr-api-endpoint"
    Environment = var.environment
  }
}

resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id              = module.vpc.vpc_id
  service_name        = "com.amazonaws.${var.region}.ecr.dkr"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = module.vpc.private_subnets
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = {
    Name        = "${var.vpc_name}-ecr-dkr-endpoint"
    Environment = var.environment
  }
}

resource "aws_security_group" "vpc_endpoints" {
  name        = "${var.vpc_name}-vpc-endpoints-sg"
  description = "Security group for VPC endpoints"
  vpc_id      = module.vpc.vpc_id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.vpc_name}-vpc-endpoints-sg"
    Environment = var.environment
  }
}
```

```hcl
# modules/vpc/variables.tf

variable "vpc_name" {
  description = "Name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "private_subnets" {
  description = "Private subnet CIDR blocks"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}

variable "public_subnets" {
  description = "Public subnet CIDR blocks"
  type        = list(string)
  default     = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
}

variable "database_subnets" {
  description = "Database subnet CIDR blocks"
  type        = list(string)
  default     = ["10.0.201.0/24", "10.0.202.0/24", "10.0.203.0/24"]
}

variable "cluster_name" {
  description = "Name of the EKS cluster"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "project_name" {
  description = "Project name"
  type        = string
}

variable "region" {
  description = "AWS region"
  type        = string
}
```

```hcl
# modules/vpc/outputs.tf

output "vpc_id" {
  description = "VPC ID"
  value       = module.vpc.vpc_id
}

output "private_subnets" {
  description = "Private subnet IDs"
  value       = module.vpc.private_subnets
}

output "public_subnets" {
  description = "Public subnet IDs"
  value       = module.vpc.public_subnets
}

output "database_subnets" {
  description = "Database subnet IDs"
  value       = module.vpc.intra_subnets
}

output "database_subnet_group_name" {
  description = "Database subnet group name"
  value       = module.vpc.database_subnet_group_name
}

output "vpc_cidr_block" {
  description = "VPC CIDR block"
  value       = module.vpc.vpc_cidr_block
}
```

---

## EKS Module

```hcl
# modules/eks/main.tf

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.23"
    }
  }
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"

  cluster_name    = var.cluster_name
  cluster_version = var.cluster_version

  vpc_id     = var.vpc_id
  subnet_ids = var.private_subnets

  # Cluster access
  cluster_endpoint_public_access  = true
  cluster_endpoint_private_access = true

  # Add-ons
  cluster_addons = {
    coredns = {
      most_recent = true
      configuration_values = jsonencode({
        computeType = "Fargate"
        replicaCount = 2
      })
    }
    kube-proxy = {
      most_recent = true
    }
    vpc-cni = {
      most_recent              = true
      before_compute           = true
      service_account_role_arn = module.vpc_cni_irsa.iam_role_arn
      configuration_values = jsonencode({
        env = {
          ENABLE_PREFIX_DELEGATION = "true"
          WARM_PREFIX_TARGET       = "1"
        }
      })
    }
    aws-ebs-csi-driver = {
      most_recent              = true
      service_account_role_arn = module.ebs_csi_irsa.iam_role_arn
    }
  }

  # Managed Node Groups
  eks_managed_node_groups = {
    # General purpose nodes
    general = {
      name            = "${var.cluster_name}-general"
      instance_types  = var.general_instance_types
      capacity_type   = "ON_DEMAND"
      
      min_size     = var.general_min_size
      max_size     = var.general_max_size
      desired_size = var.general_desired_size

      labels = {
        role = "general"
      }

      tags = {
        "k8s.io/cluster-autoscaler/enabled"             = "true"
        "k8s.io/cluster-autoscaler/${var.cluster_name}" = "owned"
      }
    }

    # Spot instances for cost optimization
    spot = {
      name            = "${var.cluster_name}-spot"
      instance_types  = var.spot_instance_types
      capacity_type   = "SPOT"
      
      min_size     = var.spot_min_size
      max_size     = var.spot_max_size
      desired_size = var.spot_desired_size

      labels = {
        role = "spot"
      }

      taints = [{
        key    = "spot"
        value  = "true"
        effect = "NO_SCHEDULE"
      }]

      tags = {
        "k8s.io/cluster-autoscaler/enabled"             = "true"
        "k8s.io/cluster-autoscaler/${var.cluster_name}" = "owned"
      }
    }
  }

  # aws-auth configmap
  manage_aws_auth_configmap = true

  aws_auth_roles = var.aws_auth_roles

  aws_auth_users = var.aws_auth_users

  tags = {
    Environment = var.environment
    Terraform   = "true"
    Project     = var.project_name
  }
}

# IRSA for VPC CNI
module "vpc_cni_irsa" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  version = "~> 5.0"

  role_name             = "${var.cluster_name}-vpc-cni"
  attach_vpc_cni_policy = true
  vpc_cni_enable_ipv4   = true

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:aws-node"]
    }
  }

  tags = {
    Environment = var.environment
  }
}

# IRSA for EBS CSI Driver
module "ebs_csi_irsa" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  version = "~> 5.0"

  role_name             = "${var.cluster_name}-ebs-csi"
  attach_ebs_csi_policy = true

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:ebs-csi-controller-sa"]
    }
  }

  tags = {
    Environment = var.environment
  }
}

# IRSA for AWS Load Balancer Controller
module "aws_lb_controller_irsa" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  version = "~> 5.0"

  role_name                              = "${var.cluster_name}-aws-lb-controller"
  attach_load_balancer_controller_policy = true

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:aws-load-balancer-controller"]
    }
  }

  tags = {
    Environment = var.environment
  }
}

# IRSA for External DNS
module "external_dns_irsa" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  version = "~> 5.0"

  role_name                     = "${var.cluster_name}-external-dns"
  attach_external_dns_policy    = true
  external_dns_hosted_zone_arns = var.route53_zone_arns

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:external-dns"]
    }
  }

  tags = {
    Environment = var.environment
  }
}
```

```hcl
# modules/eks/variables.tf

variable "cluster_name" {
  description = "Name of the EKS cluster"
  type        = string
}

variable "cluster_version" {
  description = "Kubernetes version"
  type        = string
  default     = "1.28"
}

variable "vpc_id" {
  description = "VPC ID"
  type        = string
}

variable "private_subnets" {
  description = "Private subnet IDs"
  type        = list(string)
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "project_name" {
  description = "Project name"
  type        = string
}

# Node group configurations
variable "general_instance_types" {
  description = "Instance types for general node group"
  type        = list(string)
  default     = ["t3.medium", "t3.large"]
}

variable "general_min_size" {
  description = "Minimum size for general node group"
  type        = number
  default     = 2
}

variable "general_max_size" {
  description = "Maximum size for general node group"
  type        = number
  default     = 10
}

variable "general_desired_size" {
  description = "Desired size for general node group"
  type        = number
  default     = 3
}

variable "spot_instance_types" {
  description = "Instance types for spot node group"
  type        = list(string)
  default     = ["t3.medium", "t3.large", "t3.xlarge"]
}

variable "spot_min_size" {
  description = "Minimum size for spot node group"
  type        = number
  default     = 0
}

variable "spot_max_size" {
  description = "Maximum size for spot node group"
  type        = number
  default     = 10
}

variable "spot_desired_size" {
  description = "Desired size for spot node group"
  type        = number
  default     = 2
}

# Auth configuration
variable "aws_auth_roles" {
  description = "Additional IAM roles to add to aws-auth configmap"
  type = list(object({
    rolearn  = string
    username = string
    groups   = list(string)
  }))
  default = []
}

variable "aws_auth_users" {
  description = "Additional IAM users to add to aws-auth configmap"
  type = list(object({
    userarn  = string
    username = string
    groups   = list(string)
  }))
  default = []
}

variable "route53_zone_arns" {
  description = "Route53 zone ARNs for External DNS"
  type        = list(string)
  default     = []
}
```

```hcl
# modules/eks/outputs.tf

output "cluster_id" {
  description = "EKS cluster ID"
  value       = module.eks.cluster_id
}

output "cluster_name" {
  description = "EKS cluster name"
  value       = module.eks.cluster_name
}

output "cluster_endpoint" {
  description = "EKS cluster endpoint"
  value       = module.eks.cluster_endpoint
}

output "cluster_certificate_authority_data" {
  description = "Base64 encoded certificate data"
  value       = module.eks.cluster_certificate_authority_data
}

output "oidc_provider_arn" {
  description = "OIDC provider ARN"
  value       = module.eks.oidc_provider_arn
}

output "aws_lb_controller_role_arn" {
  description = "IAM role ARN for AWS Load Balancer Controller"
  value       = module.aws_lb_controller_irsa.iam_role_arn
}

output "external_dns_role_arn" {
  description = "IAM role ARN for External DNS"
  value       = module.external_dns_irsa.iam_role_arn
}

output "cluster_security_group_id" {
  description = "Security group ID attached to the EKS cluster"
  value       = module.eks.cluster_security_group_id
}

output "node_security_group_id" {
  description = "Security group ID attached to the EKS nodes"
  value       = module.eks.node_security_group_id
}
```

---

## Environment Configuration

```hcl
# environments/dev/main.tf

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.23"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.11"
    }
  }

  backend "s3" {
    bucket         = "your-terraform-state-bucket"
    key            = "eks/dev/terraform.tfstate"
    region         = "us-west-2"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

provider "aws" {
  region = var.region

  default_tags {
    tags = {
      Environment = var.environment
      Project     = var.project_name
      ManagedBy   = "Terraform"
    }
  }
}

# Configure kubernetes provider after EKS is created
provider "kubernetes" {
  host                   = module.eks.cluster_endpoint
  cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)

  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    args        = ["eks", "get-token", "--cluster-name", module.eks.cluster_name]
  }
}

provider "helm" {
  kubernetes {
    host                   = module.eks.cluster_endpoint
    cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)

    exec {
      api_version = "client.authentication.k8s.io/v1beta1"
      command     = "aws"
      args        = ["eks", "get-token", "--cluster-name", module.eks.cluster_name]
    }
  }
}

# VPC Module
module "vpc" {
  source = "../../modules/vpc"

  vpc_name     = "${var.project_name}-${var.environment}"
  vpc_cidr     = var.vpc_cidr
  cluster_name = local.cluster_name
  environment  = var.environment
  project_name = var.project_name
  region       = var.region
}

# EKS Module
module "eks" {
  source = "../../modules/eks"

  cluster_name    = local.cluster_name
  cluster_version = var.cluster_version
  vpc_id          = module.vpc.vpc_id
  private_subnets = module.vpc.private_subnets
  environment     = var.environment
  project_name    = var.project_name

  general_instance_types = var.general_instance_types
  general_min_size       = var.general_min_size
  general_max_size       = var.general_max_size
  general_desired_size   = var.general_desired_size

  aws_auth_users = var.aws_auth_users
  aws_auth_roles = var.aws_auth_roles

  route53_zone_arns = var.route53_zone_arns
}

locals {
  cluster_name = "${var.project_name}-${var.environment}"
}
```

```hcl
# environments/dev/variables.tf

variable "region" {
  description = "AWS region"
  type        = string
  default     = "us-west-2"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "project_name" {
  description = "Project name"
  type        = string
  default     = "ecommerce"
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  default     = "10.0.0.0/16"
}

variable "cluster_version" {
  description = "EKS cluster version"
  type        = string
  default     = "1.28"
}

variable "general_instance_types" {
  description = "Instance types for general node group"
  type        = list(string)
  default     = ["t3.medium"]
}

variable "general_min_size" {
  type    = number
  default = 2
}

variable "general_max_size" {
  type    = number
  default = 5
}

variable "general_desired_size" {
  type    = number
  default = 2
}

variable "aws_auth_users" {
  type    = list(any)
  default = []
}

variable "aws_auth_roles" {
  type    = list(any)
  default = []
}

variable "route53_zone_arns" {
  type    = list(string)
  default = []
}
```

```hcl
# environments/dev/terraform.tfvars

region       = "us-west-2"
environment  = "dev"
project_name = "ecommerce"

vpc_cidr        = "10.0.0.0/16"
cluster_version = "1.28"

general_instance_types = ["t3.medium"]
general_min_size       = 2
general_max_size       = 5
general_desired_size   = 2

# Add your IAM user ARN here
aws_auth_users = [
  {
    userarn  = "arn:aws:iam::ACCOUNT_ID:user/your-username"
    username = "your-username"
    groups   = ["system:masters"]
  }
]
```

---

## Terraform Workspaces for Multi-Environment

```bash
# Initialize Terraform
cd environments/dev
terraform init

# Create workspaces
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

# List workspaces
terraform workspace list

# Switch workspace
terraform workspace select dev

# Apply for current workspace
terraform apply
```

### Using Workspaces in Configuration

```hcl
# Using workspace in configuration
locals {
  environment = terraform.workspace
  
  # Environment-specific configurations
  config = {
    dev = {
      instance_types = ["t3.medium"]
      min_nodes      = 2
      max_nodes      = 5
    }
    staging = {
      instance_types = ["t3.large"]
      min_nodes      = 2
      max_nodes      = 8
    }
    prod = {
      instance_types = ["t3.large", "t3.xlarge"]
      min_nodes      = 3
      max_nodes      = 20
    }
  }
}

# Use in module
module "eks" {
  # ...
  general_instance_types = local.config[local.environment].instance_types
  general_min_size       = local.config[local.environment].min_nodes
  general_max_size       = local.config[local.environment].max_nodes
}
```

---

## Deploying the Infrastructure

```bash
# 1. Initialize Terraform
terraform init

# 2. Validate configuration
terraform validate

# 3. Plan changes
terraform plan -out=tfplan

# 4. Apply changes
terraform apply tfplan

# 5. Configure kubectl
aws eks update-kubeconfig --name ecommerce-dev --region us-west-2

# 6. Verify cluster
kubectl get nodes
kubectl get pods -A
```

---

## Key Takeaways

1. **Modular Terraform** makes infrastructure reusable and maintainable
2. **VPC design** is critical for EKS security and networking
3. **EKS add-ons** should be managed via Terraform for consistency
4. **IRSA** provides secure AWS access for pods
5. **Workspaces** enable multi-environment management
6. **State management** with S3 backend is essential for teams

---

## Next: Helm Charts Deep Dive

In the next section, we'll learn to package applications with Helm.
