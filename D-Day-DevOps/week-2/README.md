# Week 2: Production-Grade EKS with Terraform

## Overview

This week we move from local Kind clusters to production-grade AWS EKS infrastructure. You'll learn to provision and manage real cloud infrastructure using Terraform, package applications with Helm, and implement production patterns.

## Learning Objectives

By the end of this week, you will:
- Provision production-grade EKS clusters with Terraform
- Manage multiple environments using Terraform workspaces
- Write and package Helm charts
- Implement IRSA (IAM Roles for Service Accounts)
- Deploy 3-tier applications with RDS databases
- Handle database migrations with Kubernetes Jobs
- Configure AWS Load Balancer Controller and ExternalDNS
- Implement RBAC and security hardening

## Prerequisites

- AWS Account with appropriate permissions
- AWS CLI configured
- Terraform installed (v1.5+)
- Helm installed (v3.12+)
- kubectl installed
- Week 1 knowledge

---

## Day 1-2: Terraform for EKS

### Topics
1. Terraform modules architecture
2. VPC design for EKS
3. EKS cluster provisioning
4. Node groups configuration
5. EKS add-ons management

### Files
- [01-terraform-eks-fundamentals.md](theory/01-terraform-eks-fundamentals.md)

---

## Day 3-4: Helm Charts Deep Dive

### Topics
1. Helm chart structure
2. Templates and values
3. Dependencies and umbrella charts
4. Helm hooks for migrations
5. Chart packaging and repositories

### Files
- [02-helm-charts-deep-dive.md](theory/02-helm-charts-deep-dive.md)

---

## Day 5-6: IRSA, Load Balancing, and DNS

### Topics
1. IRSA architecture and implementation
2. AWS Load Balancer Controller
3. ExternalDNS for Route53
4. SSL/TLS with ACM
5. Ingress configuration

### Files
- [03-irsa-loadbalancer-dns.md](theory/03-irsa-loadbalancer-dns.md)

---

## Day 7-8: RBAC, Security, and 3-Tier App

### Topics
1. Kubernetes RBAC deep dive
2. EKS access management
3. Database migrations with Jobs
4. Production deployment patterns
5. Capstone project

### Files
- [04-rbac-security-3tier-app.md](theory/04-rbac-security-3tier-app.md)

---

## Capstone Project

Deploy a production-grade 3-tier e-commerce application on EKS:
- Frontend (React)
- Backend (Node.js)
- Database (RDS PostgreSQL)

With:
- Terraform-provisioned infrastructure
- Helm-packaged application
- IRSA for AWS service access
- Custom domain with SSL
- Database migrations via Jobs
- Full RBAC implementation

---

## Resources

### Official Documentation
- [EKS User Guide](https://docs.aws.amazon.com/eks/latest/userguide/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Helm Documentation](https://helm.sh/docs/)

### Terraform Modules
- [terraform-aws-modules/vpc](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest)
- [terraform-aws-modules/eks](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest)

### Tools
- [eksctl](https://eksctl.io/) - Alternative EKS management tool
- [AWS IAM Authenticator](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html)
