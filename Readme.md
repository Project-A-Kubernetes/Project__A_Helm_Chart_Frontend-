# Frontend Helm Chart

## Overview

This repository contains a **production-grade Helm chart** for deploying the frontend component of Project A on Kubernetes. The chart is designed with **scalability, security, observability, and GitOps practices** in mind.

It supports full lifecycle management of the frontend application, including deployment, networking, TLS, autoscaling, monitoring, and continuous delivery via ArgoCD.

---

## Key Features

* Kubernetes-native deployment (Deployment, Service, Ingress)
* Automated TLS with cert-manager (ClusterIssuer + Certificate)
* Horizontal Pod Autoscaling (HPA) to dynamically scale workloads based on resource utilization and reduce operational cost
* Canary Release Strategy
* serviceAccount(Role) for short live access to AWS
* Prometheus monitoring via ServiceMonitor
* GitOps-ready with ArgoCD Application
* Environment configuration via ConfigMap
* Production-grade security and resource management

---

## Architecture Components

* Deployment
* Service
* Ingress (TLS termination)
* ConfigMap
* HPA
* ServiceMonitor
* serviceAccount
* Canary Release
* ClusterIssuer & Certificate
* ArgoCD Application

---
## Architecture Diagram
### System Diagram
![Kubernetes Architecture](images/frontend_architecture_via_ingress.png)

---
### Diagram of Argocd watching and syncing cluster
![Kubernetes Architecture](images/images3.png)

---

## Repository Structure

```
frontend-helm-chart/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── hpa.yaml
│   ├── servicemonitor.yaml
│   ├── clusterissuer.yaml
│   ├── certificate.yaml
|   |__ Canary_Release.yaml 
│   ├── argocd-application.yaml
│   └── _helpers.tpl
```

---

## Environment Configuration (Production vs Staging)

### Example: `values-prod.yaml`

```yaml
replicaCount: 3
image:
  repository: <ECR_REPO>
  tag: sha-66025d3

resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10

ingress:
  enabled: true
  hosts:
    - app.example.com

tls:
  enabled: true
```

### Example: `values-staging.yaml`

```yaml
replicaCount: 1
image:
  repository: <ECR_REPO>
  tag: latest

autoscaling:
  enabled: false

ingress:
  hosts:
    - staging.example.com
```

---

## ArgoCD GitOps (Multi-Environment)

### ApplicationSet Example

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: frontend-appset
spec:
  generators:
    - list:
        elements:
          - env: prod
            namespace: prod
          - env: staging
            namespace: staging
  template:
    metadata:
      name: frontend-{{env}}
    spec:
      project: default
      source:
        repoURL: <REPO_URL>
        targetRevision: main
        path: .
        helm:
          valueFiles:
            - values-{{env}}.yaml
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

---
## Auto updated Helm Chart diagram
* With CICD we update the helm repo after images has been built and push to ecr automatically
![Kubernetes Architecture](images/image2.png)

## AWS ECR + IRSA Integration

### Image Pull Secret

```bash
#For local minikube test
kubectl create secret docker-registry ecr-secret \
  --docker-server=<AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region <REGION>) \
  -n prod
```

### IRSA (IAM Role for Service Account)

* Associated IAM role(created with terraform) with Kubernetes ServiceAccount
* Grant least-privilege access (ECR pullOnly)

---

## Ingress Security Hardening

Recommended annotations:

```yaml
nginx.ingress.kubernetes.io/backend-protocol: "HTTP"  #internal routing
nginx.ingress.kubernetes.io/force-ssl-redirect: "true" # enforce SSL/TLS redirection 
nginx.ingress.kubernetes.io/proxy-body-size: "10m"
cert-manager.io/cluster-issuer: letsencrypt-production
nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
nginx.ingress.kubernetes.io/limit-rps: "5" #adjust as required

```
---
### Diagram of my frontend connected to my backend and database
![Kubernetes Architecture](images/image1.png)

---
## Observability & Alerting
- Metrics collected via Prometheus for CPU, memory, request rate, and application health
- ServiceMonitor used for automatic service discovery
- Alerts defined using PrometheusRule based on SLI/SLO thresholds
- Integrated with alerting channels (Slack / PagerDuty)

--- 
### A sidecar was injected to the pod so it can scrape nginx metric for prometheus to use /metrics

---
### Picture of prometheus rule configured and backend service discovey
---
![Kubernetes Observability Architecture](images/image.png)

- This prometheus rule send alert to slack channel when pod is crashing or
- when pod CPU and memory Usage exceed limits defined in the **resources:**
- It Collects metrics and send ALERT when our SLO is **violated** to slack teams or pagerduty if configured
---

### Picture of  Alert firing because of Violated SLI,SLO,BURN_RATE 
![Kubernetes Observability Architecture](images/image6.png)

---
### Picture of Alert when issue was resolved and Alert stop firing 
![Kubernetes Observability Architecture](images/image7.png)

---
> **Note:** Check [`project-a-observability` ](https://github.com/Project-A-Kubernetes/Project_A_Observability.git) for the observability Helm chart Automated with Argocd.
---
> **Important** Check [`project-a-cluster-stacks-tools` ](https://github.com/Project-A-Kubernetes/Project_A_STACKS_TOOLS.git) for the monitoring stacks, networking and security Helm chart automated with Argocd.
---
### Prometheus Alert Example

```yaml
groups:
  - name: frontend-alerts
    rules:
      - alert: HighCPUUsage
        expr: rate(container_cpu_usage_seconds_total[5m]) > 0.7
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: High CPU usage detected
```

### Metrics

* `/metrics` endpoint exposed
* Scraped via ServiceMonitor
* Visualized in Grafana dashboards

---

## Deployment Commands

```bash
helm install frontend . -n prod
helm upgrade frontend . -n prod
helm uninstall frontend -n prod

```
---
### Use Argocd
```
argocd app create -f <Application_Name>
```

---

## Release Strategy

- Image versioning based on Git commit SHA for traceability
- Optional `latest` tag for development environments only
- Separation of Helm chart version (`Chart.yaml`) and application version (`appVersion`)

---

## Scalability & Reliability

* HPA-based scaling
* Pod health Probing
* Rolling updates (zero downtime)
* Canary Release
* Stateless design

---

## Cost Optimization

* Autoscaling reduces idle cost
* Right-sized resource requests/limits
* Compatible with Cluster Autoscaler

---

## Extensibility

Future improvements:

* Service mesh (Istio / Linkerd)
* Cluster Autoscaler 
* Network Policy
* Blue and Green Release

---

## Security Best Practices

- Pods configured with restrictive securityContext (non-root, read-only filesystem where applicable)
- No secrets stored in Helm charts or repositories
- IAM Roles for Service Accounts (IRSA) for fine-grained AWS access
- Enforced HTTPS via ingress with TLS termination
- GitHub App authentication for secure repository access

---

## Summary

This Helm chart delivers a **production-ready frontend platform** with:

* Secure ingress and TLS
* Autoscaling
* Observability
* Reliability
* GitOps-driven deployments

Designed to reflect **real-world DevOps engineering standards** used in production environments.
