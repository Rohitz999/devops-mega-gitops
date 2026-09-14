# 🐙 DevOps Mega GitOps — Kubernetes Manifests & ArgoCD Config

[![ArgoCD](https://img.shields.io/badge/ArgoCD-GitOps-EF7B4D?logo=argo)](https://argocd.mechnomax.co.in)
[![K3s](https://img.shields.io/badge/K3s-Kubernetes-FFC61C?logo=k3s)](https://k3s.io)
[![cert-manager](https://img.shields.io/badge/cert--manager-SSL-326CE5)](https://cert-manager.io)
[![Nginx](https://img.shields.io/badge/Nginx-Ingress-009639?logo=nginx)](https://kubernetes.github.io/ingress-nginx)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

**GitOps repository for the DevOps Mega App. Watched by ArgoCD and auto-synced to a K3s cluster.**

🌐 **Live App:** [https://app.mechnomax.co.in](https://app.mechnomax.co.in)

---

## 📖 Table of Contents

- [Overview](#overview)
- [How It Works](#how-it-works)
- [Repository Structure](#repository-structure)
- [Manifests](#manifests)
- [ArgoCD Application](#argocd-application)
- [GitOps Pipeline](#gitops-pipeline)
- [Setup](#setup)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)
- [Related Repositories](#related-repositories)
- [License](#license)

---

## Overview

This repository holds the **Kubernetes manifests** and **ArgoCD configuration** for the DevOps Mega App.

**Key principle:** *Git is the single source of truth.* Any change to these manifests is automatically synced to the K3s cluster by ArgoCD — no manual `kubectl apply` needed.

**What's in this repo:**

- Kubernetes Deployment, Service, and Ingress manifests
- ArgoCD Application CRD
- GitOps Jenkinsfile (updates image tags + triggers ArgoCD)
- Let's Encrypt SSL via cert-manager

---

## How It Works

```
┌──────────────────────────────────────────────┐
│  App Repo: devops-cicd-app                   │
│  (source code + Dockerfile)                  │
└──────────┬───────────────────────────────────┘
           │ Jenkins builds + pushes image
           ▼
┌──────────────────────────────────────────────┐
│  DockerHub                                   │
│  rohitdockerhub01/devops-mega-app:1.0.0-42   │
└──────────┬───────────────────────────────────┘
           │
           │ Jenkins triggers GitOps pipeline
           ▼
┌──────────────────────────────────────────────┐
│  This Repo: devops-mega-gitops               │
│  ────────────────────────────────────        │
│  Jenkinsfile updates:                        │
│  manifests/deployment.yaml:                  │
│    image: rohitdockerhub01/devops-mega-app   │
│           :1.0.0-42                          │
│  Commits + pushes to GitHub                  │
│  Triggers ArgoCD sync via API                │
└──────────┬───────────────────────────────────┘
           │
           │ ArgoCD watches this repo
           ▼
┌──────────────────────────────────────────────┐
│  ArgoCD                                      │
│  Detects new commit → Syncs manifests to K3s │
└──────────┬───────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────┐
│  K3s Cluster                                 │
│  2 pods running with new image               │
│  Rolling update (zero downtime)              │
└──────────┬───────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────┐
│  https://app.mechnomax.co.in                 │
│  Updated live in ~2 minutes                  │
└──────────────────────────────────────────────┘
```

---

## Repository Structure

```
devops-mega-gitops/
├── Jenkinsfile                    # GitOps pipeline (updates image tag + triggers ArgoCD)
├── README.md                      # This file
├── .gitignore
├── argocd/
│   └── application.yaml           # ArgoCD Application CRD
└── manifests/
    ├── deployment.yaml            # K8s Deployment (2 replicas, probes)
    ├── service.yaml               # K8s Service (ClusterIP)
    └── ingress.yaml               # K8s Ingress (with TLS)
```

---

## Manifests

### `manifests/deployment.yaml`

Defines the app pods.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devops-mega-app
  labels:
    app: devops-mega-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: devops-mega-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: devops-mega-app
    spec:
      containers:
        - name: devops-mega-app
          image: rohitdockerhub01/devops-mega-app:latest   # ← updated by Jenkins
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8080
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 40
            periodSeconds: 20
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

**Key points:**
- **2 replicas** for high availability
- **RollingUpdate** with `maxUnavailable: 0` — zero downtime
- **readinessProbe + livenessProbe** on `/health`
- **`imagePullPolicy: Always`** — always pulls fresh images

### `manifests/service.yaml`

Exposes pods inside the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: devops-mega-app-svc
spec:
  type: ClusterIP
  selector:
    app: devops-mega-app
  ports:
    - port: 80
      targetPort: 8080
```

### `manifests/ingress.yaml`

Exposes the app to the internet with HTTPS.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: devops-mega-app-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.mechnomax.co.in
      secretName: app-mechnomax-tls
  rules:
    - host: app.mechnomax.co.in
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: devops-mega-app-svc
                port:
                  number: 80
```

**SSL is automatic:** cert-manager + Let's Encrypt issues and renews the certificate.

---

## ArgoCD Application

`argocd/application.yaml` tells ArgoCD what to deploy and from where:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: devops-mega-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/Rohitz999/devops-mega-gitops.git
    targetRevision: HEAD
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**What this does:**
- Watches `manifests/` in this repo
- Auto-syncs changes to the `default` namespace in K3s
- Prunes resources removed from Git
- Self-heals if someone manually changes K8s resources

**Apply it:**

```bash
kubectl apply -f argocd/application.yaml
```

---

## GitOps Pipeline

The `Jenkinsfile` in this repo runs when triggered by the main app pipeline.

**Stages:**

| # | Stage | What it does |
|---|-------|--------------|
| 1 | **Clone GitOps Repo** | Clones this repo using `github-creds` |
| 2 | **Update Image Tag** | `sed` replaces image tag in `deployment.yaml` |
| 3 | **Commit and Push** | Commits + pushes to GitHub |
| 4 | **Trigger ArgoCD Sync** | Calls ArgoCD API to sync immediately |

**Parameters:**

| Name | Default | Purpose |
|------|---------|---------|
| `IMAGE_TAG` | `latest` | The new image tag to deploy |

**Trigger token:** `gitops-token` (for remote triggering from the main pipeline)

---

## Setup

### Prerequisites

- **K3s cluster** running
- **ArgoCD** installed and accessible
- **Jenkins** with GitOps job configured (see below)
- **cert-manager** + **Nginx Ingress** installed on K3s
- DNS A record for `app.mechnomax.co.in` pointing to K3s IP

### Deploy ArgoCD Application

```bash
# Apply the Application CRD
kubectl apply -f argocd/application.yaml

# Verify
kubectl get applications -n argocd
# Expected: devops-mega-app   Synced   Healthy

# Check pods
kubectl get pods -l app=devops-mega-app
# Expected: 2 pods Running
```

### Configure Jenkins GitOps Job

**Jenkins → New Item → Pipeline:**

| Field | Value |
|-------|-------|
| Name | `devops-mega-gitops` |
| This project is parameterized | ✅ String Parameter: `IMAGE_TAG` (default: `latest`) |
| Trigger builds remotely | ✅ Token: `gitops-token` |
| Pipeline from SCM | Git |
| Repository URL | `https://github.com/Rohitz999/devops-mega-gitops.git` |
| Credentials | `github-creds` |
| Branch | `*/main` |
| Script Path | `Jenkinsfile` |

**Required Jenkins credentials:**

| ID | Kind | Purpose |
|----|------|---------|
| `github-creds` | Username + Password | Clone/push to GitHub |
| `argocd-token` | Secret text | Trigger ArgoCD API |

---

## Verification

### Check ArgoCD Status

```bash
kubectl get application devops-mega-app -n argocd
# Expected: Synced   Healthy
```

### Check Pods

```bash
kubectl get pods -l app=devops-mega-app
# Expected: 2 pods Running

kubectl get pods -l app=devops-mega-app -o jsonpath='{.items[*].spec.containers[*].image}'
# Expected: rohitdockerhub01/devops-mega-app:1.0.0-<N>
```

### Check Ingress & SSL

```bash
kubectl get ingress
# Expected: devops-mega-app-ingress with ADDRESS

kubectl get certificate
# Expected: app-mechnomax-tls   READY: True
```

### Check Live App

```bash
curl https://app.mechnomax.co.in/health
# Expected: OK

curl https://app.mechnomax.co.in/version
# Expected: v1.0.0
```

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| ArgoCD `OutOfSync` | Auto-sync off | Enable via UI or `kubectl patch` |
| ArgoCD shows old revision | Polling delayed | Click REFRESH; wait 3 min |
| Pods `ImagePullBackOff` | Image missing/private | Verify on DockerHub; set `imagePullPolicy: Always` |
| Pods `CrashLoopBackOff` | App failing to start | `kubectl logs <pod>` to see error |
| Certificate `False` | DNS or port 80 | Check `dig app.mechnomax.co.in`; open port 80 |
| Ingress 404 | Wrong service/class | Verify `ingressClassName: nginx` and service name |
| GitOps push fails | `github-creds` wrong | Verify PAT has `repo` scope |
| "Nothing to push" | Same tag reused | Use unique tags (`1.0.0-<build#>`) |

### Diagnostic Commands

```bash
# ArgoCD app detail
kubectl describe application devops-mega-app -n argocd

# Pod logs
kubectl logs -l app=devops-mega-app --tail=50

# Certificate issue
kubectl describe certificate app-mechnomax-tls
kubectl describe challenge -A

# Force sync
kubectl patch application devops-mega-app -n argocd --type merge \
  -p '{"operation":{"sync":{"revision":"HEAD"}}}'

# Force pod restart
kubectl rollout restart deployment devops-mega-app
kubectl rollout status deployment devops-mega-app
```

---

## Related Repositories

| Repo | Purpose |
|------|---------|
| **[devops-cicd-app](https://github.com/Rohitz999/devops-cicd-app)** | Java app + Dockerfile + main Jenkinsfile |

---

## License

MIT License — see [LICENSE](LICENSE) for details.

---

## Author

**Rohit Vishwakarma**
- GitHub: [@Rohitz999](https://github.com/Rohitz999)
- Email: rohitvishwakarma8082@gmail.com

---

## ⭐ Show Your Support

If this project helped you, give it a star ⭐
