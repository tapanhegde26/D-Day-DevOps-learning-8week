# DevOps Mastery: 8-Week Beginner to Advanced Course

A comprehensive, self-paced DevOps learning curriculum covering Kubernetes, AWS EKS, Terraform, and AIOps.

## Course Overview

This course transforms you from a Kubernetes beginner to an advanced DevOps practitioner capable of designing, deploying, and managing production-grade infrastructure.

## Prerequisites

Before starting, ensure you have:
- Basic AWS knowledge (EC2, S3, IAM, VPC)
- Terraform fundamentals (resources, providers, state)
- Docker basics (building images, running containers)

## Course Structure

| Week | Topic | Key Skills |
|------|-------|------------|
| 1 | [Kubernetes Fundamentals](week-1/) | Kind, Pods, Deployments, Services, Prometheus, ArgoCD |
| 2 | [Production EKS with Terraform](week-2/) | Terraform modules, Helm charts, IRSA, RDS |
| 3 | [Microservices & Service Mesh](week-3/) | Istio, mTLS, Gateway API, Network Policies |
| 4 | [Observability & SRE](week-4/) | Prometheus, Loki, Vault, SLOs, Error Budgets |
| 5 | [Scaling & Storage](week-5/) | Karpenter, KEDA, StatefulSets, Velero |
| 6 | [DevSecOps](week-6/) | Trivy, Cosign, Kyverno, Falco, CIS Benchmarks |
| 7 | [Operators & Chaos](week-7/) | CRDs, Operator SDK, LitmusChaos, Incident Response |
| 8 | [AIOps](week-8/) | Ollama, RAG, Qdrant, AI-powered monitoring |

## Directory Structure

```
d-day-devops/
├── 8-week-plan              # Original course outline
├── README.md                # This file
├── week-1/
│   ├── theory/              # Detailed learning materials
│   ├── labs/                # Hands-on exercises
│   └── project/             # Capstone project
├── week-2/
│   ├── theory/
│   ├── labs/
│   ├── project/
│   └── terraform/           # Terraform modules
├── week-3/
│   ├── theory/
│   ├── labs/
│   └── project/
├── week-4/
│   ├── theory/
│   ├── labs/
│   └── project/
├── week-5/
│   ├── theory/
│   ├── labs/
│   └── project/
├── week-6/
│   ├── theory/
│   ├── labs/
│   └── project/
├── week-7/
│   ├── theory/
│   ├── labs/
│   └── project/
└── week-8/
    ├── theory/
    ├── labs/
    └── project/
```

## How to Use This Course

### Self-Paced Learning

1. **Read the theory** — Each week has comprehensive markdown files explaining concepts
2. **Complete the labs** — Hands-on exercises to practice what you learned
3. **Build the project** — Each week culminates in a capstone project
4. **Review and iterate** — Go back to topics you find challenging

### Recommended Schedule

- **4 days/week, 2 hours/day** = 8 weeks
- **2 days/week, 2 hours/day** = 16 weeks
- **Intensive: 8 hours/day** = 2 weeks

### Tools Required

**Local Development:**
- Docker Desktop
- kubectl
- Kind
- Helm
- k9s or Lens (optional but recommended)

**Cloud (Week 2+):**
- AWS Account (Free Tier works for most labs)
- AWS CLI configured
- Terraform

**CI/CD:**
- GitHub Account
- GitHub Actions (free tier)

## Week-by-Week Summary

### Week 1: Kubernetes Fundamentals
Build a solid foundation with core Kubernetes concepts on a local Kind cluster.

**Topics:**
- Kubernetes architecture (Control Plane, Worker Nodes)
- Pods, Deployments, Services, ReplicaSets
- ConfigMaps, Secrets, Resource Management
- Health Probes, HPA, Deployment Strategies
- Prometheus + Grafana monitoring
- ArgoCD GitOps

**Project:** 2-tier e-commerce app with full monitoring and GitOps

### Week 2: Production EKS with Terraform
Move to production-grade AWS infrastructure.

**Topics:**
- Terraform modules for VPC and EKS
- Helm chart development
- IRSA (IAM Roles for Service Accounts)
- AWS Load Balancer Controller
- ExternalDNS and SSL/TLS
- Database migrations with Jobs

**Project:** 3-tier app on EKS with RDS, custom domain, and RBAC

### Week 3: Microservices & Service Mesh
Master complex microservices patterns.

**Topics:**
- Istio service mesh
- mTLS for zero-trust security
- Traffic management (canary, blue-green)
- Gateway API
- Network Policies
- ArgoCD App-of-Apps

**Project:** Microservices e-commerce with Istio and canary deployments

### Week 4: Observability & SRE
Build full observability and implement SRE practices.

**Topics:**
- Prometheus Operator and ServiceMonitors
- Loki for log aggregation
- Distributed tracing with OpenTelemetry
- HashiCorp Vault for secrets
- SLOs, SLIs, Error Budgets
- Kubecost for cost visibility

**Project:** Full observability stack with SLO dashboards

### Week 5: Scaling & Storage
Master advanced scaling and stateful workloads.

**Topics:**
- Karpenter for node autoscaling
- KEDA for event-driven scaling
- StatefulSets for databases
- Persistent Volumes and Storage Classes
- Velero for backup and DR

**Project:** Microservices with Karpenter, KEDA, and StatefulSets

### Week 6: DevSecOps
Implement end-to-end security.

**Topics:**
- Container image scanning (Trivy, Grype)
- SBOM generation and image signing
- Kyverno policy enforcement
- Pod Security Standards
- Falco runtime security
- CIS Benchmark compliance

**Project:** Full DevSecOps pipeline with security gates

### Week 7: Operators & Chaos Engineering
Extend Kubernetes and validate resilience.

**Topics:**
- Custom Resource Definitions
- Building Operators with Operator SDK
- LitmusChaos experiments
- Incident response lifecycle
- War room simulations
- RCA and postmortem writing

**Project:** Custom Operator and chaos experiments

### Week 8: AIOps
Deploy AI workloads and build intelligent infrastructure.

**Topics:**
- Ollama on Kubernetes
- RAG pipeline with Qdrant
- AI-powered health checking
- Automated troubleshooting
- Chat-based infrastructure management

**Project:** Complete AIOps system with AI chat interface

## Certifications

After completing this course, you'll be well-prepared for:
- **CKA** (Certified Kubernetes Administrator)
- **CKAD** (Certified Kubernetes Application Developer)
- **CKS** (Certified Kubernetes Security Specialist)
- **AWS Solutions Architect**
- **HashiCorp Terraform Associate**

## Additional Resources

### Official Documentation
- [Kubernetes Docs](https://kubernetes.io/docs/)
- [AWS EKS User Guide](https://docs.aws.amazon.com/eks/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/)
- [Helm Documentation](https://helm.sh/docs/)
- [Istio Documentation](https://istio.io/latest/docs/)

### Books
- "Kubernetes Up & Running" by Kelsey Hightower
- "Production Kubernetes" by Josh Rosso
- "Site Reliability Engineering" by Google

### Practice Platforms
- [Killercoda](https://killercoda.com/) — Free Kubernetes scenarios
- [KodeKloud](https://kodekloud.com/) — Hands-on labs
- [A Cloud Guru](https://acloudguru.com/) — Cloud certifications

## Contributing

Found an error or want to improve the content? Feel free to:
1. Open an issue
2. Submit a pull request
3. Suggest new topics

## License

This educational content is provided for personal learning purposes.

---

**Happy Learning! 🚀**

Remember: The best way to learn DevOps is by doing. Don't just read — build, break, fix, and repeat.
