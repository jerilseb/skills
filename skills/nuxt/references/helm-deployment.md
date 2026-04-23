# Helm Deployment

Deploy a Nuxt application to Kubernetes using Helm. The chart provisions a Namespace, Deployment, Service, and Ingress with TLS.

## Chart Structure

```
helm/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── namespace.yaml
    ├── api.yaml
    └── ingress.yaml
```

## Chart.yaml

```yaml
apiVersion: v2
name: <app-name>
description: A Helm chart for <app-name>
type: application
version: 0.1.0
appVersion: "1.0.0"
```

## values.yaml

```yaml
environment: production

image:
  repository: <registry>/<image-name>
  tag: ""  # Set at deploy time, e.g. via --set image.tag=abc123

replicas: 1

ingress:
  host: app.example.com

cert:
  issuer: letsencrypt-prod  # or letsencrypt-staging
```

## templates/namespace.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: {{ .Release.Name }}
```

## templates/api.yaml

Contains the Deployment and Service.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <app-name>-deployment
  namespace: {{ .Release.Name }}
spec:
  selector:
    matchLabels:
      app: <app-name>
  replicas: {{ .Values.replicas }}
  template:
    metadata:
      labels:
        app: <app-name>
    spec:
      containers:
        - name: <app-name>
          image: {{ required "image.repository is required" .Values.image.repository }}:{{ required "image.tag is required" .Values.image.tag }}
          imagePullPolicy: Always
          command: ["node", ".output/server/index.mjs"]
          ports:
            - containerPort: 9999
          envFrom:
            - secretRef:
                name: <app-name>-secrets
---
apiVersion: v1
kind: Service
metadata:
  name: <app-name>-service
  namespace: {{ .Release.Name }}
spec:
  type: ClusterIP
  ports:
    - port: 9999
      targetPort: 9999
  selector:
    app: <app-name>
```

### Key conventions

- `command: ["node", ".output/server/index.mjs"]` — runs the Nitro production server built by `nuxt build`.
- `envFrom.secretRef` — loads environment variables (e.g. `DATABASE_URL`, `NUXT_JWT_SECRET`, `NUXT_GOOGLE_CLIENT_ID`, `NUXT_GOOGLE_CLIENT_SECRET`) from a pre-created Kubernetes Secret named `<app-name>-secrets`.
- The container port matches the `PORT` env in the Dockerfile (default `9999`).

## templates/ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <app-name>-ingress
  namespace: {{ .Release.Name }}
  annotations:
    cert-manager.io/cluster-issuer: {{ required "cert.issuer is required" .Values.cert.issuer }}
spec:
  ingressClassName: nginx
  rules:
    - host: {{ required "ingress.host is required" .Values.ingress.host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: <app-name>-service
                port:
                  number: 9999
  tls:
    - hosts:
        - {{ .Values.ingress.host }}
      secretName: <app-name>-ingress-tls
```

### Key conventions

- `cert-manager.io/cluster-issuer` — configures automatic TLS via cert-manager.
- The Ingress forwards traffic to the Service on the same port the container exposes.

## Prerequisites

Before installing the chart, create the secrets:

```bash
kubectl create secret generic <app-name>-secrets \
  --from-literal=DATABASE_URL="postgres://..." \
  --from-literal=NUXT_JWT_SECRET="..." \
  --from-literal=NUXT_GOOGLE_CLIENT_ID="..." \
  --from-literal=NUXT_GOOGLE_CLIENT_SECRET="..." \
  -n <namespace>
```

## Deploy

```bash
helm upgrade --install <release-name> ./helm \
  --set image.tag=<git-sha-or-semver> \
  --set ingress.host=app.example.com
```

## Optional: Spot Instances

If your cluster supports spot/preemptible nodes, add tolerations to the Pod spec:

```yaml
spec:
  tolerations:
    - key: "lifecycle"
      operator: "Equal"
      value: "spot"
```
