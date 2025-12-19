# GitOps with ArgoCD

Declarative, Git-centric continuous delivery for Kubernetes.

## ArgoCD Application Patterns

```yaml
# App of Apps Pattern - Bootstrap entire cluster
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cluster-bootstrap
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/cluster-config
    targetRevision: HEAD
    path: apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
---
# Individual App managed by App of Apps
# apps/api.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: production
  source:
    repoURL: https://github.com/org/api
    targetRevision: HEAD
    path: k8s/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: api
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

## ApplicationSet for Multi-Cluster

```yaml
# Deploy to multiple clusters/environments
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: api-multi-cluster
  namespace: argocd
spec:
  generators:
    - matrix:
        generators:
          - git:
              repoURL: https://github.com/org/environments
              revision: HEAD
              directories:
                - path: environments/*
          - clusters:
              selector:
                matchLabels:
                  env: "{{path.basename}}"
  template:
    metadata:
      name: "api-{{path.basename}}-{{name}}"
    spec:
      project: "{{path.basename}}"
      source:
        repoURL: https://github.com/org/api
        targetRevision: HEAD
        path: "k8s/overlays/{{path.basename}}"
        helm:
          valueFiles:
            - "values-{{path.basename}}.yaml"
          parameters:
            - name: cluster.name
              value: "{{name}}"
            - name: cluster.server
              value: "{{server}}"
      destination:
        server: "{{server}}"
        namespace: api
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
---
# Pull Request Preview Environments
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: pr-previews
spec:
  generators:
    - pullRequest:
        github:
          owner: org
          repo: api
          tokenRef:
            secretName: github-token
            key: token
        requeueAfterSeconds: 60
  template:
    metadata:
      name: "api-pr-{{number}}"
      annotations:
        preview-url: "https://pr-{{number}}.preview.example.com"
    spec:
      project: previews
      source:
        repoURL: https://github.com/org/api
        targetRevision: "{{head_sha}}"
        path: k8s/overlays/preview
        helm:
          parameters:
            - name: ingress.host
              value: "pr-{{number}}.preview.example.com"
      destination:
        server: https://kubernetes.default.svc
        namespace: "preview-{{number}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

## Progressive Delivery with Argo Rollouts

```yaml
# Analysis Templates for Canary Validation
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 1m
      count: 5
      successCondition: result[0] >= 0.99
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{service="{{args.service-name}}", status=~"2.."}[5m]))
            /
            sum(rate(http_requests_total{service="{{args.service-name}}"}[5m]))
    - name: latency-p99
      interval: 1m
      count: 5
      successCondition: result[0] < 500
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_ms_bucket{service="{{args.service-name}}"}[5m])) by (le)
            )
    - name: error-rate
      interval: 1m
      count: 5
      successCondition: result[0] < 0.01
      failureLimit: 2
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{service="{{args.service-name}}", status=~"5.."}[5m]))
            /
            sum(rate(http_requests_total{service="{{args.service-name}}"}[5m]))
---
# Experiment for A/B Testing
apiVersion: argoproj.io/v1alpha1
kind: Experiment
metadata:
  name: checkout-experiment
spec:
  duration: 1h
  progressDeadlineSeconds: 300
  templates:
    - name: baseline
      replicas: 1
      selector:
        matchLabels:
          app: checkout
          version: baseline
      template:
        metadata:
          labels:
            app: checkout
            version: baseline
        spec:
          containers:
            - name: checkout
              image: checkout:v1
    - name: experiment
      replicas: 1
      selector:
        matchLabels:
          app: checkout
          version: experiment
      template:
        metadata:
          labels:
            app: checkout
            version: experiment
        spec:
          containers:
            - name: checkout
              image: checkout:v2-new-ui
  analyses:
    - name: conversion-rate
      templateName: conversion-comparison
      args:
        - name: baseline-service
          value: checkout-baseline
        - name: experiment-service
          value: checkout-experiment
```

## Sync Waves & Hooks

```yaml
# Control deployment order with sync waves
apiVersion: v1
kind: Namespace
metadata:
  name: api
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
---
apiVersion: v1
kind: Secret
metadata:
  name: api-secrets
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  annotations:
    argocd.argoproj.io/sync-wave: "0"
---
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/sync-wave: "1"
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: api:latest
          command: ["npm", "run", "migrate"]
      restartPolicy: Never
---
apiVersion: batch/v1
kind: Job
metadata:
  name: smoke-test
  annotations:
    argocd.argoproj.io/sync-wave: "2"
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
        - name: smoke-test
          image: api:latest
          command: ["npm", "run", "test:smoke"]
      restartPolicy: Never
```

## Image Updater Automation

```yaml
# Automatically update images from registry
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api
  annotations:
    argocd-image-updater.argoproj.io/image-list: api=ghcr.io/org/api
    argocd-image-updater.argoproj.io/api.update-strategy: semver
    argocd-image-updater.argoproj.io/api.allow-tags: regexp:^v[0-9]+\.[0-9]+\.[0-9]+$
    argocd-image-updater.argoproj.io/api.ignore-tags: "*-rc*,*-alpha*,*-beta*"
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
spec:
  source:
    repoURL: https://github.com/org/api
    path: k8s/overlays/production
```

---

*Learned: 2024*
*Tags: GitOps, ArgoCD, Kubernetes, CI/CD, Progressive Delivery*
