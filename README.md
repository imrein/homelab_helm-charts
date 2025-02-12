# Homelab helm charts repo

<p align="center">
<img src="https://helm.sh/img/helm.svg" width=100>
</p>

<div align="center">
A collection of helm charts used in my homelab.
</div>

## Overview

Status: **WIP**

This repo contains the helm charts for my `k3s` setup in my homelab. It utilizes a general library chart to create a skeleton for various applications, core components, and infrastructure services. 

The structure includes:
- **Apps**: Application-specific Helm charts (e.g., AdGuardHome, MinIO, Nginx, ...).
- **Core Components**: Charts for essential services (e.g., ArgoCD, Traefik, Vault, ...).
- **Library**: A Helm library with reusable templates for consistent chart development.

> [!NOTE]
> I created all of the profiles from the ground up to learn more about puppet. These profiles are not perfect and might still have flaws and bad-practices.

### Apps

- [x] Adguardhome
- [x] Ddns
- [x] Demo-app
- [x] Homepage
- [x] Minio
- [x] Ntfy
- [x] Postgresql
- [x] Speedtest
- [x] Stirlingpdf
- [x] Twitchminer
- [x] Uptime-kuma
- [x] Blog (Nginx)

## Example usage

My homelab uses ArgoCD to deploy everything, so I created another simple helm chart seperate from this repository to deploy everything (`homelab_k3s`).
As an example this is how my "core" components are deployed:

### ArgoCD application

The main application which acts as an umbrella chart to deploy the applications/components.

```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-application
  namespace: argocd
spec:
  destination:
    namespace: default
    server: https://kubernetes.default.svc
  project: default
  source:
    path: config
    repoURL: git@192.168.129.170:homelab/homelab_k3s.git
    targetRevision: HEAD
    helm:
      valueFiles:
        - values.yaml   # This is changed to the right environment
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Deployment template

Template is created to adjust to the `dev` or `prod` environment. Otherwise it's probably a "core" component and will be deployed in the `default` namespace unless specified otherwise.

```yaml
{{- range .Values.apps }}
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  {{- if eq $.Values.global.env "dev" }}
  name: dev-{{ .name }}
  {{- else if eq $.Values.global.env "prod" }}
  name: prod-{{ .name }}
  {{- else }}
  name: {{ .name }}
  {{- end }}
  namespace: argocd
spec:
  destination:
  {{- if .namespace }}
    namespace: {{ .namespace | default "default" }}
  {{- else }}
    namespace: {{ $.Values.global.env | default "default" }}
  {{- end }}
    server: https://kubernetes.default.svc
  project: default
  sources:
    {{- if eq .type "core" }}
    - path: core-component/{{ .name }}
    {{- else }}
    - path: apps/{{ .name }}
    {{- end }}
      repoURL: git@192.168.129.170:homelab/homelab_helm-charts.git
      targetRevision: {{ $.Values.global.revision | default "HEAD" }}
      helm:
        ignoreMissingValueFiles: true
        valueFiles:
          - values.yaml
        {{- if eq $.Values.global.env "dev" }}
          - values-dev.yaml
        {{- else if eq $.Values.global.env "prod" }}
          - values-prod.yaml
        {{- end }}
    {{- if .extraResources }}
    - path: core-component/{{ .name }}
      repoURL: git@192.168.129.170:homelab/homelab_helm-charts.git
      targetRevision: HEAD
      directory:
        recurse: true
    {{- end }}
  syncPolicy:
    automated: {}
    syncOptions:
      - CreateNamespace=true
{{- end }}
```

### Values

Define default values and apps to deploy.

```yaml
global:
  env: default
  revision: HEAD
apps:
  # Core stuff
  - name: prometheus
    namespace: monitoring
    type: core
  - name: promtail
    namespace: logging
    type: core
  - name: traefik
    namespace: traefik
    type: core
    extraResources: true
  - name: vault-secrets-operator
    type: core
    extraResources: true
  - name: extra-reverse-proxy
    type: core
  - name: longhorn
    namespace: longhorn-system
    type: core
    extraResources: true
  - name: cert-manager
    namespace: cert-manager
    type: core
    extraResources: true
  - name: argocd
    namespace: argocd
    type: core
    extraResources: true
```
