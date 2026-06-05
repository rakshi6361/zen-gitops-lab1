# Lab: Deploy auth-service with ArgoCD — Plain YAML → Helm

**Goal:** Deploy a single microservice (`auth-service`) to Kubernetes using ArgoCD.  
You will do this in two phases so you understand *why* Helm exists.

- **Phase 1** — Write raw Kubernetes YAML and point ArgoCD at it. Watch it sync.  
- **Phase 2** — Convert the same manifest to a Helm chart with a values file. Update the ArgoCD app. Watch it re-sync.

---

## Prerequisites

- A fresh git repo (your `zen-gitops`) pushed to GitHub
- ArgoCD installed on your cluster and accessible (`argocd` CLI logged in)
- `kubectl` pointed at your cluster
- The ArgoCD `pharma` AppProject already applied (see `argocd/projects/pharma-project.yaml`)

---

## Repo layout you will build

```
zen-gitops/
├── argocd/
│   ├── projects/
│   │   └── pharma-project.yaml       ← already done
│   └── apps/
│       └── auth-service-app.yaml     ← you will create this
├── k8s/                              ← Phase 1: plain K8s manifests
│   └── auth-service/
│       ├── namespace.yaml
│       ├── configmap.yaml
│       ├── deployment.yaml
│       └── service.yaml
├── helm-charts/                      ← Phase 2: Helm chart
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── _helpers.tpl
│       ├── configmap.yaml
│       ├── deployment.yaml
│       ├── service.yaml
│       └── serviceaccount.yaml
└── envs/
    └── prod/
        └── values-auth-service.yaml  ← Phase 2: per-service overrides
```

---

## Phase 1 — Raw Kubernetes YAML

### Step 1.1 — Create the namespace manifest

Create `k8s/auth-service/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
```

### Step 1.2 — Create the ConfigMap

`auth-service` is a Spring Boot app. Its config (DB host, port, log level) lives in a ConfigMap so you can change it without rebuilding the image.

Create `k8s/auth-service/configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: auth-service
  namespace: prod
data:
  SPRING_PROFILES_ACTIVE: "prod"
  SERVER_PORT: "8081"
  LOG_LEVEL: "INFO"
  TOKEN_EXPIRATION_MS: "86400000"
  MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE: "health,info,metrics"
```

> **Why ConfigMap and not env vars hardcoded in the Deployment?**  
> ConfigMaps let you change config independently of the Deployment spec. You can update just the ConfigMap, and a rolling restart picks up the new values — no image rebuild needed.

### Step 1.3 — Create the Deployment

Create `k8s/auth-service/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: prod
  labels:
    app: auth-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
    spec:
      containers:
        - name: auth-service
          image: 516209541629.dkr.ecr.us-east-1.amazonaws.com/auth-service:sha-ef1ccc7
          imagePullPolicy: Always
          ports:
            - containerPort: 8081
              protocol: TCP
          envFrom:
            - configMapRef:
                name: auth-service
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 8081
            initialDelaySeconds: 60
            periodSeconds: 15
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8081
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
```

**Key concepts in this Deployment:**

| Field | What it does |
|---|---|
| `selector.matchLabels` | Links the Deployment to its Pods — must match `template.metadata.labels` |
| `envFrom.configMapRef` | Injects all ConfigMap keys as environment variables |
| `resources.requests` | What the scheduler reserves on a node for this pod |
| `resources.limits` | Hard ceiling — container is OOM-killed if it exceeds memory limit |
| `livenessProbe` | K8s restarts the container if this fails repeatedly |
| `readinessProbe` | K8s stops sending traffic to the pod until this passes |

### Step 1.4 — Create the Service

Create `k8s/auth-service/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: auth-service
  namespace: prod
  labels:
    app: auth-service
spec:
  type: ClusterIP
  selector:
    app: auth-service
  ports:
    - name: http
      port: 8081
      targetPort: 8081
      protocol: TCP
```

> **ClusterIP** means this service is only reachable from inside the cluster. Other microservices call `http://auth-service:8081`. External traffic will later go through an API Gateway service.

### Step 1.5 — Create the ArgoCD Application

Now tell ArgoCD where to find your manifests.

Create `argocd/apps/auth-service-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: auth-service-prod
  namespace: argocd
  labels:
    env: prod
    app: auth-service
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: pharma

  source:
    repoURL: https://github.com/<YOUR-ORG>/zen-gitops.git   # replace with your repo URL
    targetRevision: HEAD
    path: k8s/auth-service                                   # pointing at raw YAML folder

  destination:
    server: https://kubernetes.default.svc
    namespace: prod

  syncPolicy:
    automated:
      prune: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
```

> **What `automated` means:**  
> ArgoCD polls your git repo every ~3 minutes (default). When it detects a commit that changes the files under `path:`, it automatically applies them to the cluster — you do not run `kubectl apply` manually.  
> Without `automated`, you would have to click "Sync" in the UI or run `argocd app sync`.

### Step 1.6 — Apply the ArgoCD Application and watch it sync

```bash
# Apply the ArgoCD Application resource
kubectl apply -f argocd/apps/auth-service-app.yaml

# Watch the sync status
argocd app get auth-service-prod

# Stream live sync events
argocd app logs auth-service-prod

# Or watch pods appear in the prod namespace
kubectl get pods -n prod -w
```

You should see ArgoCD:
1. Clone your repo
2. Detect the 4 YAML files under `k8s/auth-service/`
3. Apply them — Namespace → ConfigMap → Deployment → Service
4. Report `Synced` / `Healthy` once the Pod passes its readiness probe

**Verify the deployment:**

```bash
# Check pod is running
kubectl get pods -n prod

# Check the service is created
kubectl get svc -n prod

# Check configmap values were injected
kubectl exec -n prod deploy/auth-service -- env | grep SPRING
```

### Step 1.7 — Trigger a sync by changing something in git

Edit `k8s/auth-service/configmap.yaml` and change `LOG_LEVEL` from `INFO` to `DEBUG`. Commit and push.

```bash
git add k8s/auth-service/configmap.yaml
git commit -m "chore(auth-service): set log level to DEBUG for testing"
git push
```

Within ~3 minutes, ArgoCD will:
1. Detect the new commit
2. Diff the cluster state against the new desired state
3. Apply only the changed ConfigMap

```bash
# Watch ArgoCD pick up the change
argocd app history auth-service-prod

# Confirm the new value is live on the cluster
kubectl get configmap auth-service -n prod -o yaml
```

> This is the core GitOps loop: **git is the source of truth**. You never run `kubectl apply` yourself in production — you commit to git, and ArgoCD makes it so.

---

## Phase 2 — Convert to Helm

### Why Helm?

Your raw YAML works for one service. But you have 8 microservices. Every one needs a Deployment, Service, ConfigMap, HPA, ServiceAccount — the same shape, different values. Copy-pasting 8 deployments creates drift and maintenance pain.

Helm lets you write the shape once (templates) and supply per-service values. ArgoCD natively renders Helm charts before applying them.

### Step 2.1 — Create Chart.yaml

Create `helm-charts/Chart.yaml`:

```yaml
apiVersion: v2
name: pharma-service
description: Shared Helm chart for all pharma microservices
type: application
version: 1.0.0
appVersion: "1.0.0"
```

### Step 2.2 — Create the default values.yaml

`helm-charts/values.yaml` defines defaults. Every service that does not override a value gets this.

Create `helm-charts/values.yaml`:

```yaml
replicaCount: 1

image:
  repository: ""
  tag: "latest"
  pullPolicy: IfNotPresent

fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

service:
  type: ClusterIP
  port: 8080
  targetPort: 8080

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 70

livenessProbe:
  path: /actuator/health
  port: 8080
  initialDelaySeconds: 60
  periodSeconds: 15
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  path: /actuator/health/readiness
  port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

configmap: {}

envFrom: []

volumes: []
volumeMounts: []

terminationGracePeriodSeconds: 30
```

### Step 2.3 — Create the template helpers

Helm templates reuse name generation logic via `_helpers.tpl`.

Create `helm-charts/templates/_helpers.tpl`:

```
{{- define "pharma-service.name" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{- define "pharma-service.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{- define "pharma-service.labels" -}}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{ include "pharma-service.selectorLabels" . }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{- define "pharma-service.selectorLabels" -}}
app.kubernetes.io/name: {{ include "pharma-service.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{- define "pharma-service.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "pharma-service.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

### Step 2.4 — Create the Deployment template

Create `helm-charts/templates/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "pharma-service.fullname" . }}
  labels:
    {{- include "pharma-service.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "pharma-service.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "pharma-service.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "pharma-service.serviceAccountName" . }}
      terminationGracePeriodSeconds: {{ .Values.terminationGracePeriodSeconds }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          {{- if or .Values.configmap .Values.envFrom }}
          envFrom:
            {{- if .Values.configmap }}
            - configMapRef:
                name: {{ include "pharma-service.fullname" . }}
            {{- end }}
            {{- with .Values.envFrom }}
            {{- toYaml . | nindent 12 }}
            {{- end }}
          {{- end }}
          livenessProbe:
            httpGet:
              path: {{ .Values.livenessProbe.path }}
              port: {{ .Values.livenessProbe.port }}
            initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.livenessProbe.periodSeconds }}
            timeoutSeconds: {{ .Values.livenessProbe.timeoutSeconds }}
            failureThreshold: {{ .Values.livenessProbe.failureThreshold }}
          readinessProbe:
            httpGet:
              path: {{ .Values.readinessProbe.path }}
              port: {{ .Values.readinessProbe.port }}
            initialDelaySeconds: {{ .Values.readinessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.readinessProbe.periodSeconds }}
            timeoutSeconds: {{ .Values.readinessProbe.timeoutSeconds }}
            failureThreshold: {{ .Values.readinessProbe.failureThreshold }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          {{- with .Values.volumeMounts }}
          volumeMounts:
            {{- toYaml . | nindent 12 }}
          {{- end }}
      {{- with .Values.volumes }}
      volumes:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### Step 2.5 — Create the Service template

Create `helm-charts/templates/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "pharma-service.fullname" . }}
  labels:
    {{- include "pharma-service.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
      protocol: TCP
      name: http
  selector:
    {{- include "pharma-service.selectorLabels" . | nindent 4 }}
```

### Step 2.6 — Create the ConfigMap template

Create `helm-charts/templates/configmap.yaml`:

```yaml
{{- if .Values.configmap }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "pharma-service.fullname" . }}
  labels:
    {{- include "pharma-service.labels" . | nindent 4 }}
data:
  {{- range $key, $value := .Values.configmap }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
{{- end }}
```

> The `{{- if .Values.configmap }}` guard means no ConfigMap is created when the values file has `configmap: {}` (the default). Only services with actual config keys get a ConfigMap.

### Step 2.7 — Create the ServiceAccount template

Create `helm-charts/templates/serviceaccount.yaml`:

```yaml
{{- if .Values.serviceAccount.create -}}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "pharma-service.serviceAccountName" . }}
  labels:
    {{- include "pharma-service.labels" . | nindent 4 }}
  {{- with .Values.serviceAccount.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
{{- end }}
```

### Step 2.8 — Create auth-service values override

The shared chart has defaults. `auth-service` overrides only what it needs.

Create `envs/prod/values-auth-service.yaml`:

```yaml
fullnameOverride: auth-service

image:
  repository: 516209541629.dkr.ecr.us-east-1.amazonaws.com/auth-service
  tag: sha-ef1ccc7
  pullPolicy: Always

service:
  type: ClusterIP
  port: 8081
  targetPort: 8081

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

livenessProbe:
  path: /actuator/health
  port: 8081
  initialDelaySeconds: 60
  periodSeconds: 15
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  path: /actuator/health/readiness
  port: 8081
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

configmap:
  SPRING_PROFILES_ACTIVE: prod
  SERVER_PORT: "8081"
  LOG_LEVEL: "INFO"
  TOKEN_EXPIRATION_MS: "86400000"
  MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE: "health,info,metrics"

serviceAccount:
  create: true
  name: auth-service
```

> **Why a separate values file per service?**  
> The chart template is shared. Each service has its own `values-<name>.yaml`. When you add `catalog-service` next week, you write zero new template files — only a new values file. ArgoCD renders the same chart with different values.

### Step 2.9 — Preview what Helm will generate (optional but recommended)

Before you commit, verify Helm renders what you expect locally:

```bash
helm template auth-service-prod helm-charts/ \
  -f envs/prod/values-auth-service.yaml \
  --namespace prod
```

You should see a Deployment, Service, ConfigMap, and ServiceAccount with `auth-service` names and the correct image tag and port `8081`.

### Step 2.10 — Update the ArgoCD Application to use Helm

Edit `argocd/apps/auth-service-app.yaml` — change the `source` block from the raw YAML path to the Helm chart path:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: auth-service-prod
  namespace: argocd
  labels:
    env: prod
    app: auth-service
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: pharma

  source:
    repoURL: https://github.com/<YOUR-ORG>/zen-gitops.git
    targetRevision: HEAD
    path: helm-charts                                    # now pointing at the chart
    helm:
      valueFiles:
        - ../envs/prod/values-auth-service.yaml          # service-specific overrides

  destination:
    server: https://kubernetes.default.svc
    namespace: prod

  syncPolicy:
    automated:
      prune: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

**What changed:**
- `path: k8s/auth-service` → `path: helm-charts`
- Added `helm.valueFiles` pointing at the per-service values file
- The `../` prefix in `valueFiles` is relative to `path:` (helm-charts), going up one level to reach `envs/prod/`

### Step 2.11 — Commit everything and watch ArgoCD re-sync

```bash
git add helm-charts/ envs/prod/values-auth-service.yaml argocd/apps/auth-service-app.yaml
git commit -m "feat(auth-service): migrate to shared Helm chart"
git push
```

Watch ArgoCD pick up the change:

```bash
# Watch the app status
argocd app get auth-service-prod -w

# Once synced, verify resources
argocd app resources auth-service-prod
```

ArgoCD will:
1. Detect the updated Application pointing at `helm-charts/`
2. Run `helm template` internally to render the manifests
3. Diff the rendered manifests against the live cluster
4. Apply the delta — in this case it replaces the raw Deployment/Service/ConfigMap with the Helm-rendered equivalents

**Verify the Helm labels are now on your resources:**

```bash
kubectl get deployment auth-service -n prod -o yaml | grep helm.sh/chart
# Should show: helm.sh/chart: pharma-service-1.0.0
```

---

## What you just built

```
Git commit
    │
    ▼
ArgoCD watches repo (every ~3 min or on webhook)
    │
    ├── Phase 1: reads k8s/auth-service/*.yaml → kubectl apply
    │
    └── Phase 2: reads helm-charts/ + envs/prod/values-auth-service.yaml
                 → helm template → kubectl apply
```

The cluster never drifts from git. If someone runs a manual `kubectl` change, ArgoCD detects the drift and (with `automated`) corrects it back to git state.

---

## Quick reference — useful commands

```bash
# Check sync status
argocd app get auth-service-prod

# Force sync immediately (don't wait for poll)
argocd app sync auth-service-prod

# See what ArgoCD would change without applying
argocd app diff auth-service-prod

# See all managed resources
argocd app resources auth-service-prod

# View pod logs
kubectl logs -n prod deploy/auth-service -f

# Describe the pod (for startup/probe errors)
kubectl describe pod -n prod -l app.kubernetes.io/name=auth-service
```

---

## Troubleshooting

| Symptom | Check |
|---|---|
| ArgoCD shows `OutOfSync` but not applying | `syncPolicy.automated` may be missing — add it or click Sync manually |
| Pod stuck in `CrashLoopBackOff` | `kubectl logs -n prod deploy/auth-service` — likely readiness probe failing too fast |
| `ImagePullBackOff` | Cluster cannot pull from ECR — check IAM role / imagePullSecret |
| Helm template errors in ArgoCD | Run `helm template` locally first to catch YAML errors before pushing |
| `OutOfSync` on `replicas` field | Add `ignoreDifferences` for `/spec/replicas` if HPA manages replica count |
