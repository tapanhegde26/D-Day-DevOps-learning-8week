# Week 6: DevSecOps on Kubernetes

## Overview

This week covers end-to-end security for Kubernetes: supply chain security, runtime protection, policy enforcement, and compliance.

---

## Day 1-2: Container Security & Supply Chain

### Shift-Left Security

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DEVSECOPS PIPELINE                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Code ──► Build ──► Test ──► Scan ──► Sign ──► Deploy ──► Monitor          │
│    │        │        │        │        │         │          │               │
│    ▼        ▼        ▼        ▼        ▼         ▼          ▼               │
│  SAST    Dockerfile  Unit   Trivy   Cosign   Admission   Falco             │
│  Lint    Best Prac.  Tests  Grype   Sigstore  Policies   Runtime           │
│  Secrets Multi-stage        SBOM              Kyverno    Detection         │
│                                                                              │
│  ◄─────────── SHIFT LEFT ───────────►  ◄──── SHIFT RIGHT ────►             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Dockerfile Best Practices

```dockerfile
# Bad: Large attack surface
FROM ubuntu:latest
RUN apt-get update && apt-get install -y nodejs npm
COPY . /app
RUN npm install
CMD ["node", "app.js"]

# Good: Multi-stage, minimal, non-root
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
USER nonroot:nonroot
EXPOSE 3000
CMD ["dist/server.js"]
```

### Trivy Scanning

```bash
# Scan container image
trivy image myapp:latest

# Scan with severity filter
trivy image --severity HIGH,CRITICAL myapp:latest

# Scan filesystem
trivy fs --security-checks vuln,secret,config .

# Scan Kubernetes manifests
trivy config ./k8s/

# Output as JSON for CI
trivy image --format json --output results.json myapp:latest
```

```yaml
# GitHub Actions integration
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'myapp:${{ github.sha }}'
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'
```

### SBOM Generation with Syft

```bash
# Generate SBOM
syft myapp:latest -o spdx-json > sbom.json

# Scan SBOM with Grype
grype sbom:sbom.json
```

### Image Signing with Cosign

```bash
# Generate key pair
cosign generate-key-pair

# Sign image
cosign sign --key cosign.key myregistry/myapp:latest

# Verify signature
cosign verify --key cosign.pub myregistry/myapp:latest

# Keyless signing with OIDC (GitHub Actions)
cosign sign myregistry/myapp:latest
```

```yaml
# GitHub Actions keyless signing
- name: Sign image with Cosign
  env:
    COSIGN_EXPERIMENTAL: "true"
  run: |
    cosign sign ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
```

---

## Day 3-4: Policy Enforcement

### Kyverno Policies

```yaml
# Require resource limits
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-limits
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "CPU and memory limits are required"
        pattern:
          spec:
            containers:
              - resources:
                  limits:
                    memory: "?*"
                    cpu: "?*"
---
# Disallow privileged containers
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged
spec:
  validationFailureAction: Enforce
  rules:
    - name: deny-privileged
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Privileged containers are not allowed"
        pattern:
          spec:
            containers:
              - securityContext:
                  privileged: "false"
---
# Require image from trusted registry
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-trusted-registry
spec:
  validationFailureAction: Enforce
  rules:
    - name: trusted-registry
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Images must be from trusted registries"
        pattern:
          spec:
            containers:
              - image: "ghcr.io/myorg/* | myregistry.com/*"
---
# Verify image signatures
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-signature
      match:
        any:
          - resources:
              kinds:
                - Pod
      verifyImages:
        - imageReferences:
            - "ghcr.io/myorg/*"
          attestors:
            - entries:
                - keyless:
                    subject: "https://github.com/myorg/*"
                    issuer: "https://token.actions.githubusercontent.com"
```

### Kyverno Mutations

```yaml
# Add default labels
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-labels
spec:
  rules:
    - name: add-labels
      match:
        any:
          - resources:
              kinds:
                - Pod
      mutate:
        patchStrategicMerge:
          metadata:
            labels:
              managed-by: kyverno
              environment: "{{request.namespace}}"
---
# Inject sidecar
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: inject-logging-sidecar
spec:
  rules:
    - name: inject-sidecar
      match:
        any:
          - resources:
              kinds:
                - Pod
              selector:
                matchLabels:
                  inject-logging: "true"
      mutate:
        patchStrategicMerge:
          spec:
            containers:
              - name: logging-sidecar
                image: fluent/fluent-bit:latest
                volumeMounts:
                  - name: logs
                    mountPath: /var/log/app
```

### Pod Security Standards

```yaml
# Enforce restricted PSS
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

```yaml
# Pod meeting restricted standard
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: myapp:latest
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir: {}
```

---

## Day 5-6: Runtime Security

### Falco for Runtime Detection

```yaml
# Falco Helm values
falco:
  rules_file:
    - /etc/falco/falco_rules.yaml
    - /etc/falco/falco_rules.local.yaml
    - /etc/falco/k8s_audit_rules.yaml
    - /etc/falco/rules.d
  
  json_output: true
  
  http_output:
    enabled: true
    url: "http://falcosidekick:2801"

falcosidekick:
  enabled: true
  config:
    slack:
      webhookurl: "https://hooks.slack.com/services/xxx"
      minimumpriority: warning
```

### Custom Falco Rules

```yaml
# /etc/falco/rules.d/custom-rules.yaml
- rule: Detect Crypto Mining
  desc: Detect crypto mining processes
  condition: >
    spawned_process and 
    (proc.name in (xmrig, minerd, cpuminer) or
     proc.cmdline contains "stratum+tcp" or
     proc.cmdline contains "cryptonight")
  output: >
    Crypto mining detected 
    (user=%user.name command=%proc.cmdline container=%container.name)
  priority: CRITICAL
  tags: [cryptomining, mitre_execution]

- rule: Sensitive File Access
  desc: Detect access to sensitive files
  condition: >
    open_read and 
    fd.name in (/etc/shadow, /etc/passwd, /etc/kubernetes/admin.conf) and
    not proc.name in (cat, grep, systemd)
  output: >
    Sensitive file accessed 
    (user=%user.name file=%fd.name command=%proc.cmdline)
  priority: WARNING

- rule: Shell in Container
  desc: Detect shell spawned in container
  condition: >
    spawned_process and 
    container and 
    proc.name in (bash, sh, zsh) and
    not proc.pname in (docker, containerd)
  output: >
    Shell spawned in container 
    (user=%user.name shell=%proc.name container=%container.name)
  priority: NOTICE
```

### Cilium Network Policies

```yaml
# L7 HTTP policy
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: api-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: backend
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              - method: "GET"
                path: "/api/.*"
              - method: "POST"
                path: "/api/orders"
---
# DNS policy
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: dns-policy
spec:
  endpointSelector:
    matchLabels:
      app: backend
  egress:
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: UDP
          rules:
            dns:
              - matchPattern: "*.production.svc.cluster.local"
              - matchPattern: "*.amazonaws.com"
```

---

## Day 7-8: Compliance & Auditing

### CIS Benchmark with kube-bench

```bash
# Run kube-bench
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml

# View results
kubectl logs job/kube-bench
```

### Secret Scanning with gitleaks

```yaml
# .gitleaks.toml
[allowlist]
description = "Allowlisted files"
paths = [
  '''go\.sum''',
  '''package-lock\.json''',
]

[[rules]]
description = "AWS Access Key"
regex = '''AKIA[0-9A-Z]{16}'''
tags = ["aws", "credentials"]

[[rules]]
description = "Generic API Key"
regex = '''(?i)(api[_-]?key|apikey)['":\s]*['""]?([a-zA-Z0-9]{32,})'''
tags = ["api", "key"]
```

```yaml
# GitHub Actions
- name: Run Gitleaks
  uses: gitleaks/gitleaks-action@v2
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Audit Logging

```yaml
# EKS audit policy
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all requests at Metadata level
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]
  
  # Log pod exec/attach at RequestResponse level
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach"]
  
  # Log authentication failures
  - level: Metadata
    nonResourceURLs:
      - "/api*"
      - "/version"
    omitStages:
      - "RequestReceived"
```

---

## Week 6 Capstone

Full DevSecOps pipeline:
- Trivy + Grype scanning in CI
- SBOM generation with Syft
- Cosign image signing
- Kyverno policy enforcement
- Pod Security Standards
- Falco runtime detection
- CIS benchmark compliance
- Audit logging to Grafana

---

## Key Takeaways

1. **Shift-left** security catches issues early
2. **Multi-stage builds** reduce attack surface
3. **Image signing** ensures supply chain integrity
4. **Kyverno** enforces policies as code
5. **Pod Security Standards** provide baseline security
6. **Falco** detects runtime threats
7. **Audit logging** enables forensics

---

## Next Week: CRDs, Operators, Chaos Engineering

Week 7 covers extending Kubernetes and chaos engineering.
