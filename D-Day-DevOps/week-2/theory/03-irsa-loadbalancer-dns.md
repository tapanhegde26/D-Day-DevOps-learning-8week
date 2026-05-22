# Week 2, Day 5-6: IRSA, AWS Load Balancer Controller & ExternalDNS

## IRSA — IAM Roles for Service Accounts

IRSA allows Kubernetes pods to assume AWS IAM roles securely without storing credentials.

### How IRSA Works

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           IRSA FLOW                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. Pod with ServiceAccount requests AWS credentials                        │
│                                                                              │
│  ┌─────────────┐                                                            │
│  │    Pod      │                                                            │
│  │             │                                                            │
│  │  SA: my-sa  │──────┐                                                     │
│  └─────────────┘      │                                                     │
│                       │ 2. Kubelet injects OIDC token                       │
│                       ▼                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    AWS SDK in Pod                                    │   │
│  │                                                                      │   │
│  │  Reads:                                                              │   │
│  │  - AWS_ROLE_ARN (from env)                                          │   │
│  │  - AWS_WEB_IDENTITY_TOKEN_FILE (OIDC token path)                    │   │
│  └──────────────────────────────┬──────────────────────────────────────┘   │
│                                 │                                           │
│                                 │ 3. AssumeRoleWithWebIdentity              │
│                                 ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         AWS STS                                      │   │
│  │                                                                      │   │
│  │  Validates:                                                          │   │
│  │  - OIDC token signature (via EKS OIDC provider)                     │   │
│  │  - Token claims match IAM role trust policy                         │   │
│  └──────────────────────────────┬──────────────────────────────────────┘   │
│                                 │                                           │
│                                 │ 4. Returns temporary credentials          │
│                                 ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Pod can now access AWS services                   │   │
│  │                    (S3, RDS, Secrets Manager, etc.)                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Creating IRSA with Terraform

```hcl
# IRSA for application that needs S3 access
module "app_irsa" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  version = "~> 5.0"

  role_name = "${var.cluster_name}-app-s3-access"

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["production:backend-sa"]
    }
  }

  role_policy_arns = {
    s3_policy = aws_iam_policy.app_s3_policy.arn
  }

  tags = {
    Environment = var.environment
  }
}

# Custom IAM policy for S3 access
resource "aws_iam_policy" "app_s3_policy" {
  name        = "${var.cluster_name}-app-s3-policy"
  description = "Policy for application S3 access"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:ListBucket"
        ]
        Resource = [
          "arn:aws:s3:::${var.app_bucket_name}",
          "arn:aws:s3:::${var.app_bucket_name}/*"
        ]
      }
    ]
  })
}
```

### Using IRSA in Kubernetes

```yaml
# ServiceAccount with IRSA annotation
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/ecommerce-dev-app-s3-access
---
# Deployment using the ServiceAccount
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: production
spec:
  template:
    spec:
      serviceAccountName: backend-sa
      containers:
        - name: backend
          image: my-app:1.0
          # AWS SDK automatically uses IRSA credentials
```

---

## AWS Load Balancer Controller

The AWS Load Balancer Controller manages AWS Elastic Load Balancers for Kubernetes.

### Features

| Feature | ALB (Application) | NLB (Network) |
|---------|-------------------|---------------|
| Layer | 7 (HTTP/HTTPS) | 4 (TCP/UDP) |
| Use Case | Web apps, APIs | High performance, gRPC |
| Routing | Path, host, header | Port-based |
| SSL Termination | Yes | Yes (TLS) |
| WebSocket | Yes | Yes |

### Installing AWS Load Balancer Controller

```hcl
# Terraform - Helm release for AWS LB Controller
resource "helm_release" "aws_load_balancer_controller" {
  name       = "aws-load-balancer-controller"
  repository = "https://aws.github.io/eks-charts"
  chart      = "aws-load-balancer-controller"
  namespace  = "kube-system"
  version    = "1.6.2"

  set {
    name  = "clusterName"
    value = module.eks.cluster_name
  }

  set {
    name  = "serviceAccount.create"
    value = "true"
  }

  set {
    name  = "serviceAccount.name"
    value = "aws-load-balancer-controller"
  }

  set {
    name  = "serviceAccount.annotations.eks\\.amazonaws\\.com/role-arn"
    value = module.eks.aws_lb_controller_role_arn
  }

  depends_on = [module.eks]
}
```

### ALB Ingress Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: production
  annotations:
    # Use ALB
    kubernetes.io/ingress.class: alb
    # Or use ingressClassName
    # alb.ingress.kubernetes.io/scheme: internet-facing
    
    # Internet-facing or internal
    alb.ingress.kubernetes.io/scheme: internet-facing
    
    # Target type: ip (recommended for EKS) or instance
    alb.ingress.kubernetes.io/target-type: ip
    
    # SSL certificate from ACM
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-west-2:123456789012:certificate/abc123
    
    # Redirect HTTP to HTTPS
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    
    # Health check settings
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "15"
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: "5"
    alb.ingress.kubernetes.io/healthy-threshold-count: "2"
    alb.ingress.kubernetes.io/unhealthy-threshold-count: "2"
    
    # Listener ports
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    
    # Security groups
    alb.ingress.kubernetes.io/security-groups: sg-xxxx
    
    # WAF integration
    alb.ingress.kubernetes.io/wafv2-acl-arn: arn:aws:wafv2:...
    
    # Access logs
    alb.ingress.kubernetes.io/load-balancer-attributes: access_logs.s3.enabled=true,access_logs.s3.bucket=my-logs-bucket
spec:
  ingressClassName: alb
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 80
    - host: www.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

### NLB Service Configuration

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-nlb
  namespace: production
  annotations:
    # Use NLB
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    
    # Internet-facing or internal
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    
    # Cross-zone load balancing
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
    
    # SSL/TLS
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:...
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"
    
    # Proxy protocol v2
    service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"
spec:
  type: LoadBalancer
  selector:
    app: backend
  ports:
    - name: https
      port: 443
      targetPort: 8080
```

---

## ExternalDNS

ExternalDNS automatically manages DNS records in Route53 based on Kubernetes resources.

### Installing ExternalDNS

```hcl
# Terraform - Helm release for ExternalDNS
resource "helm_release" "external_dns" {
  name       = "external-dns"
  repository = "https://kubernetes-sigs.github.io/external-dns"
  chart      = "external-dns"
  namespace  = "kube-system"
  version    = "1.13.1"

  set {
    name  = "provider"
    value = "aws"
  }

  set {
    name  = "aws.region"
    value = var.region
  }

  set {
    name  = "aws.zoneType"
    value = "public"
  }

  set {
    name  = "domainFilters[0]"
    value = var.domain_name
  }

  set {
    name  = "policy"
    value = "sync"  # or "upsert-only"
  }

  set {
    name  = "txtOwnerId"
    value = module.eks.cluster_name
  }

  set {
    name  = "serviceAccount.create"
    value = "true"
  }

  set {
    name  = "serviceAccount.name"
    value = "external-dns"
  }

  set {
    name  = "serviceAccount.annotations.eks\\.amazonaws\\.com/role-arn"
    value = module.eks.external_dns_role_arn
  }

  depends_on = [module.eks]
}
```

### Using ExternalDNS with Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    # ExternalDNS will create this record
    external-dns.alpha.kubernetes.io/hostname: api.example.com
    # TTL for the DNS record
    external-dns.alpha.kubernetes.io/ttl: "300"
spec:
  ingressClassName: alb
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 80
```

### Using ExternalDNS with Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  annotations:
    external-dns.alpha.kubernetes.io/hostname: api.example.com
spec:
  type: LoadBalancer
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
```

---

## SSL/TLS with ACM

### Creating ACM Certificate with Terraform

```hcl
# ACM Certificate
resource "aws_acm_certificate" "main" {
  domain_name       = var.domain_name
  validation_method = "DNS"

  subject_alternative_names = [
    "*.${var.domain_name}"
  ]

  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Environment = var.environment
  }
}

# DNS validation records
resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.main.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  allow_overwrite = true
  name            = each.value.name
  records         = [each.value.record]
  ttl             = 60
  type            = each.value.type
  zone_id         = data.aws_route53_zone.main.zone_id
}

# Certificate validation
resource "aws_acm_certificate_validation" "main" {
  certificate_arn         = aws_acm_certificate.main.arn
  validation_record_fqdns = [for record in aws_route53_record.cert_validation : record.fqdn]
}

# Output certificate ARN for use in Ingress
output "acm_certificate_arn" {
  value = aws_acm_certificate.main.arn
}
```

### Complete Ingress with SSL

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-west-2:123456789012:certificate/xxx
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    external-dns.alpha.kubernetes.io/hostname: api.example.com,www.example.com
spec:
  ingressClassName: alb
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 80
    - host: www.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

---

## AWS Secrets Manager Integration

### External Secrets Operator

```hcl
# Install External Secrets Operator
resource "helm_release" "external_secrets" {
  name       = "external-secrets"
  repository = "https://charts.external-secrets.io"
  chart      = "external-secrets"
  namespace  = "external-secrets"
  version    = "0.9.9"

  create_namespace = true

  set {
    name  = "installCRDs"
    value = "true"
  }
}

# IRSA for External Secrets
module "external_secrets_irsa" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  version = "~> 5.0"

  role_name = "${var.cluster_name}-external-secrets"

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["external-secrets:external-secrets"]
    }
  }

  role_policy_arns = {
    secrets_policy = aws_iam_policy.external_secrets_policy.arn
  }
}

resource "aws_iam_policy" "external_secrets_policy" {
  name = "${var.cluster_name}-external-secrets-policy"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret"
        ]
        Resource = "arn:aws:secretsmanager:${var.region}:${data.aws_caller_identity.current.account_id}:secret:${var.environment}/*"
      }
    ]
  })
}
```

### Using External Secrets

```yaml
# ClusterSecretStore
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-west-2
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
---
# ExternalSecret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
  data:
    - secretKey: username
      remoteRef:
        key: production/database
        property: username
    - secretKey: password
      remoteRef:
        key: production/database
        property: password
```

---

## Key Takeaways

1. **IRSA** provides secure, credential-free AWS access for pods
2. **AWS Load Balancer Controller** manages ALB/NLB automatically
3. **ExternalDNS** automates Route53 record management
4. **ACM** provides free SSL certificates for AWS resources
5. **External Secrets Operator** syncs secrets from AWS Secrets Manager

---

## Next: RBAC, Security, and 3-Tier Application

In the next section, we'll implement RBAC and deploy the complete 3-tier application.
