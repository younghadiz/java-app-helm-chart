# Java Application Helm Chart

A reusable Helm chart for deploying the Java Spring Boot application component of the Java–MySQL Kubernetes Platform.

The chart packages the Kubernetes resources required to run the Java application, connect it securely to MySQL, expose it through a Kubernetes Service, and support production-style availability, health checking, configuration, and scaling.

---

## Table of Contents

* [Overview](#overview)
* [Architecture](#architecture)
* [Features](#features)
* [Prerequisites](#prerequisites)
* [Chart Structure](#chart-structure)
* [Configuration](#configuration)
* [Database Configuration](#database-configuration)
* [Installation](#installation)
* [Upgrade](#upgrade)
* [Validation](#validation)
* [Accessing the Application](#accessing-the-application)
* [Scaling](#scaling)
* [Rollback](#rollback)
* [Uninstallation](#uninstallation)
* [Troubleshooting](#troubleshooting)
* [Security Considerations](#security-considerations)
* [Production Recommendations](#production-recommendations)

---

## Overview

The `java-app-helm-chart` deploys a containerized Spring Boot application to Kubernetes.

The application connects to a MySQL database using environment variables supplied through Kubernetes configuration resources.

The expected database environment variables are:

```text
DB_SERVER
DB_NAME
DB_USER
DB_PWD
```

The application currently uses the standard MySQL port:

```text
3306
```

The chart is designed to work with the MySQL primary Service deployed in the same Kubernetes namespace:

```text
mysql-primary
```

The typical application-to-database connection path is:

```text
User
  |
  v
Ingress Controller
  |
  v
Java Application Service
  |
  v
Java Application Pods
  |
  v
mysql-primary Service
  |
  v
MySQL Primary Pod
```

---

## Architecture

The Helm chart manages the Java application layer of the platform.

```text
Kubernetes Cluster
└── Namespace: java-mysql
    ├── Java Application Deployment
    │   ├── Java Application Pod 1
    │   └── Java Application Pod 2
    │
    ├── Java Application Service
    │   └── Port 8080
    │
    ├── Java Application ConfigMap
    │   ├── DB_SERVER
    │   └── DB_NAME
    │
    ├── Java Application Secret
    │   ├── DB_USER
    │   └── DB_PWD
    │
    ├── PodDisruptionBudget
    │
    └── MySQL Primary Service
        └── Port 3306
```

Ingress resources may be managed by this chart or by a separate platform-level Helm release, depending on the repository configuration.

---

## Features

The chart provides:

* Kubernetes Deployment for the Java Spring Boot application
* Configurable container image repository and tag
* Kubernetes ClusterIP Service
* ConfigMap-based non-sensitive database configuration
* Secret-based database credentials
* Liveness and readiness probes
* CPU and memory requests and limits
* Configurable replica count
* Rolling deployment strategy
* PodDisruptionBudget support
* Pod and container security context support
* Configurable labels and annotations
* Optional ingress configuration
* Helm upgrade and rollback support
* Helmfile compatibility

---

## Prerequisites

Before installing the chart, ensure the following tools are available:

```bash
kubectl version --client
helm version
```

For the local platform environment, also verify Minikube:

```bash
minikube status --profile java-mysql-platform
```

The Kubernetes cluster must be reachable:

```bash
kubectl cluster-info
```

The expected namespace is:

```text
java-mysql
```

Create it if it does not already exist:

```bash
kubectl create namespace java-mysql
```

Verify the namespace:

```bash
kubectl get namespace java-mysql
```

The MySQL deployment should already exist and expose a primary Service:

```bash
kubectl get service mysql-primary \
  --namespace java-mysql
```

Expected Service port:

```text
3306/TCP
```

---

## Chart Structure

A typical chart structure is:

```text
java-app-helm-chart/
├── Chart.yaml
├── values.yaml
├── README.md
├── templates/
│   ├── _helpers.tpl
│   ├── configmap.yaml
│   ├── deployment.yaml
│   ├── ingress.yaml
│   ├── pdb.yaml
│   ├── secret.yaml
│   ├── service.yaml
│   └── tests/
│       └── test-connection.yaml
└── values/
    ├── values-dev.yaml
    ├── values-staging.yaml
    └── values-prod.yaml
```

The exact structure may vary depending on the repository implementation.

### Important files

| File                        | Purpose                                                        |
| --------------------------- | -------------------------------------------------------------- |
| `Chart.yaml`                | Defines chart metadata, name, version, and application version |
| `values.yaml`               | Contains the chart’s default configuration                     |
| `templates/deployment.yaml` | Defines the Java application Deployment                        |
| `templates/service.yaml`    | Exposes the application inside Kubernetes                      |
| `templates/configmap.yaml`  | Stores non-sensitive application configuration                 |
| `templates/secret.yaml`     | References or creates sensitive database credentials           |
| `templates/ingress.yaml`    | Optionally exposes the application through Ingress             |
| `templates/pdb.yaml`        | Protects application availability during voluntary disruptions |
| `templates/_helpers.tpl`    | Contains reusable Helm template helpers                        |

---

## Configuration

Configuration is supplied through `values.yaml` or an environment-specific values file.

Example:

```yaml
replicaCount: 2

image:
  repository: java-mysql-app
  tag: release-candidate
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8080
  targetPort: 8080

database:
  server: mysql-primary
  name: java_app_db
  existingSecret: java-app-db-secret
  userKey: DB_USER
  passwordKey: DB_PWD

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

probes:
  readiness:
    enabled: true
    path: /
    initialDelaySeconds: 10
    periodSeconds: 10
  liveness:
    enabled: true
    path: /
    initialDelaySeconds: 30
    periodSeconds: 20

podDisruptionBudget:
  enabled: true
  minAvailable: 1

ingress:
  enabled: false
  className: nginx
  host: my-java-app.com
  path: /
  pathType: Prefix
```

Inspect the currently supported values:

```bash
helm show values ./java-app-helm-chart
```

Render the chart locally:

```bash
helm template java-app \
  ./java-app-helm-chart \
  --namespace java-mysql
```

---

## Database Configuration

The application reads the following environment variables:

| Variable    | Description                               | Example            |
| ----------- | ----------------------------------------- | ------------------ |
| `DB_SERVER` | MySQL hostname or Kubernetes Service name | `mysql-primary`    |
| `DB_NAME`   | Application database name                 | `java_app_db`      |
| `DB_USER`   | MySQL application username                | `java_app_user`    |
| `DB_PWD`    | MySQL application-user password           | Stored in a Secret |

The application currently uses MySQL port `3306`.

Inside Kubernetes, the expected connection is:

```text
mysql-primary:3306
```

### Create the database Secret

Do not store plain-text passwords directly in Git-tracked values files.

Create the Secret using `kubectl`:

```bash
kubectl create secret generic java-app-db-secret \
  --namespace java-mysql \
  --from-literal=DB_USER='java_app_user' \
  --from-literal=DB_PWD='<DATABASE_PASSWORD>'
```

Verify the Secret exists without displaying its decoded values:

```bash
kubectl get secret java-app-db-secret \
  --namespace java-mysql
```

To update an existing Secret declaratively:

```bash
kubectl create secret generic java-app-db-secret \
  --namespace java-mysql \
  --from-literal=DB_USER='java_app_user' \
  --from-literal=DB_PWD='<DATABASE_PASSWORD>' \
  --dry-run=client \
  --output yaml \
  | kubectl apply -f -
```

Restart the application after updating database credentials:

```bash
kubectl rollout restart deployment/java-app \
  --namespace java-mysql
```

---

## Installation

### Install using Helm

From the repository root:

```bash
helm upgrade --install java-app \
  ./java-app-helm-chart \
  --namespace java-mysql \
  --create-namespace
```

Install with an environment-specific values file:

```bash
helm upgrade --install java-app \
  ./java-app-helm-chart \
  --namespace java-mysql \
  --create-namespace \
  --values ./java-app-helm-chart/values/values-dev.yaml
```

### Install using Helmfile

If the platform uses Helmfile as the deployment source of truth:

```bash
helmfile --selector name=java-app diff
```

Apply the release:

```bash
helmfile --selector name=java-app apply
```

Do not manage Helm-owned Java application resources with regular:

```bash
kubectl apply
```

Using both Helm and direct `kubectl apply` against the same resources can create configuration drift and conflicting ownership.

---

## Upgrade

Update the chart values or container image tag, then run:

```bash
helm upgrade java-app \
  ./java-app-helm-chart \
  --namespace java-mysql
```

With an environment-specific values file:

```bash
helm upgrade java-app \
  ./java-app-helm-chart \
  --namespace java-mysql \
  --values ./java-app-helm-chart/values/values-dev.yaml
```

Using Helmfile:

```bash
helmfile --selector name=java-app diff
```

Review the proposed changes, then apply them:

```bash
helmfile --selector name=java-app apply
```

Monitor the rollout:

```bash
kubectl rollout status deployment/java-app \
  --namespace java-mysql \
  --timeout=180s
```

---

## Validation

### Validate chart syntax

```bash
helm lint ./java-app-helm-chart
```

Expected result:

```text
1 chart(s) linted, 0 chart(s) failed
```

### Render the chart

```bash
helm template java-app \
  ./java-app-helm-chart \
  --namespace java-mysql
```

### Perform a server-side validation

```bash
helm template java-app \
  ./java-app-helm-chart \
  --namespace java-mysql \
  | kubectl apply \
      --dry-run=server \
      --namespace java-mysql \
      --filename -
```

This validates the rendered Kubernetes resources against the Kubernetes API server without applying them.

### Check the Helm release

```bash
helm status java-app \
  --namespace java-mysql
```

### Check application resources

```bash
kubectl get deployment,pods,service \
  --namespace java-mysql \
  --selector app.kubernetes.io/instance=java-app
```

Depending on the chart labels, you can also run:

```bash
kubectl get deployment java-app \
  --namespace java-mysql
```

```bash
kubectl get pods \
  --namespace java-mysql \
  --output wide
```

```bash
kubectl get service java-app \
  --namespace java-mysql
```

### Verify rollout status

```bash
kubectl rollout status deployment/java-app \
  --namespace java-mysql
```

### Inspect application logs

```bash
kubectl logs \
  --namespace java-mysql \
  deployment/java-app \
  --tail=100
```

Follow logs continuously:

```bash
kubectl logs \
  --namespace java-mysql \
  deployment/java-app \
  --follow
```

A successful startup should include messages similar to:

```text
Java app started
Tomcat started on port 8080
Started Application
```

---

## Accessing the Application

### Service port-forward

Forward the Java application Service to local port `8080`:

```bash
kubectl port-forward \
  --namespace java-mysql \
  service/java-app \
  8080:8080
```

Open:

```text
http://127.0.0.1:8080
```

Test with `curl`:

```bash
curl --include http://127.0.0.1:8080/
```

### Use a different local port

If port `8080` is occupied:

```bash
kubectl port-forward \
  --namespace java-mysql \
  service/java-app \
  8082:8080
```

Then access:

```text
http://127.0.0.1:8082
```

### Ingress access

If Ingress is enabled, verify it:

```bash
kubectl get ingress \
  --namespace java-mysql
```

For the local NGINX Ingress Controller, use:

```bash
kubectl port-forward \
  --namespace ingress-nginx \
  service/ingress-nginx-controller \
  18080:80
```

Then test using the configured host:

```bash
curl \
  --header 'Host: my-java-app.com' \
  http://127.0.0.1:18080/
```

To avoid local-port conflicts, request an automatically selected port:

```bash
kubectl port-forward \
  --namespace ingress-nginx \
  service/ingress-nginx-controller \
  :80
```

---

## Scaling

Update the replica count through Helm values:

```yaml
replicaCount: 3
```

Apply the change:

```bash
helm upgrade java-app \
  ./java-app-helm-chart \
  --namespace java-mysql \
  --set replicaCount=3
```

Using Helmfile:

```bash
helmfile --selector name=java-app apply
```

Verify:

```bash
kubectl get deployment java-app \
  --namespace java-mysql
```

Although the Deployment can be scaled directly with `kubectl`, any permanent scaling change should be recorded in Helm values to prevent Helm from reverting it during the next deployment.

---

## Rollback

List release revisions:

```bash
helm history java-app \
  --namespace java-mysql
```

Rollback to a previous revision:

```bash
helm rollback java-app <REVISION_NUMBER> \
  --namespace java-mysql
```

Monitor the rollback:

```bash
kubectl rollout status deployment/java-app \
  --namespace java-mysql
```

Example:

```bash
helm rollback java-app 1 \
  --namespace java-mysql
```

---

## Uninstallation

Remove the Java application release:

```bash
helm uninstall java-app \
  --namespace java-mysql
```

Using Helmfile:

```bash
helmfile --selector name=java-app destroy
```

Verify the resources were removed:

```bash
kubectl get all \
  --namespace java-mysql
```

Removing this chart should not remove the MySQL release unless both components are explicitly managed together.

---

## Troubleshooting

### Application Pod is not starting

Check Pod status:

```bash
kubectl get pods \
  --namespace java-mysql
```

Describe the affected Pod:

```bash
kubectl describe pod <POD_NAME> \
  --namespace java-mysql
```

View logs:

```bash
kubectl logs <POD_NAME> \
  --namespace java-mysql
```

---

### `ImagePullBackOff`

Inspect the Pod:

```bash
kubectl describe pod <POD_NAME> \
  --namespace java-mysql
```

Confirm the configured image:

```bash
kubectl get deployment java-app \
  --namespace java-mysql \
  --output jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

For a local Minikube image, load it into the Minikube profile:

```bash
minikube image load \
  java-mysql-app:release-candidate \
  --profile java-mysql-platform
```

Use the following pull policy for a locally loaded image:

```yaml
image:
  pullPolicy: IfNotPresent
```

Avoid using the `latest` image tag.

---

### Database connection failure

A database connectivity error may appear as:

```text
Communications link failure
```

or:

```text
java.net.ConnectException: Connection refused
```

Verify MySQL Pods:

```bash
kubectl get pods \
  --namespace java-mysql \
  --selector app.kubernetes.io/name=mysql
```

Verify the primary Service:

```bash
kubectl get service mysql-primary \
  --namespace java-mysql
```

Verify Service endpoints:

```bash
kubectl get endpoints mysql-primary \
  --namespace java-mysql
```

The Service should have at least one endpoint on port `3306`.

Test DNS from inside the Java application Pod:

```bash
kubectl exec \
  --namespace java-mysql \
  deployment/java-app \
  -- getent hosts mysql-primary
```

Inspect the application environment-variable names without revealing Secret values:

```bash
kubectl get deployment java-app \
  --namespace java-mysql \
  --output jsonpath='{range .spec.template.spec.containers[0].env[*]}{.name}{"\n"}{end}'
```

The application expects:

```text
DB_SERVER
DB_NAME
DB_USER
DB_PWD
```

It does not currently read:

```text
DB_HOST
DB_PASSWORD
```

---

### MySQL authentication failure

Authentication failures differ from connection failures.

Typical authentication errors include:

```text
Access denied for user
```

Verify that the application username and password in the Kubernetes Secret match the MySQL user credentials.

Check the Secret keys:

```bash
kubectl describe secret java-app-db-secret \
  --namespace java-mysql
```

Do not display decoded credentials in screenshots, CI logs, documentation, or Git commits.

---

### Service has no endpoints

Check the Service selector:

```bash
kubectl describe service java-app \
  --namespace java-mysql
```

Check Pod labels:

```bash
kubectl get pods \
  --namespace java-mysql \
  --show-labels
```

Check endpoints:

```bash
kubectl get endpoints java-app \
  --namespace java-mysql
```

The Service selector must match labels applied to the Java Deployment Pods.

---

### Readiness probe failure

Describe the Pod:

```bash
kubectl describe pod <POD_NAME> \
  --namespace java-mysql
```

Check whether the configured readiness path exists:

```bash
kubectl port-forward \
  --namespace java-mysql \
  service/java-app \
  8082:8080
```

Then test:

```bash
curl --include http://127.0.0.1:8082/
```

Adjust the readiness probe path, delay, timeout, or failure threshold in `values.yaml` when necessary.

---

### Port already allocated

Check which process is using a local port:

```bash
lsof -nP -iTCP:8082 -sTCP:LISTEN
```

For database forwarding:

```bash
lsof -nP -iTCP:3306 -sTCP:LISTEN
```

Stop only the process occupying the required test port, or choose another local port.

---

### Helm release ownership warning

Running `kubectl apply` against resources originally created by Helm may produce warnings about:

```text
kubectl.kubernetes.io/last-applied-configuration
```

This occurs because Helm-created resources do not necessarily contain the annotation expected by `kubectl apply`.

Continue managing the resources using:

```bash
helm upgrade
```

or:

```bash
helmfile apply
```

Do not migrate resource ownership to `kubectl apply` unintentionally.

---

## Security Considerations

### Do not commit credentials

Never place real credentials in:

* `values.yaml`
* environment-specific values files
* Git commits
* README examples
* Docker commands stored in shell history
* CI/CD logs

Use:

* Kubernetes Secrets
* external secret-management systems
* CI/CD protected variables
* local Git-ignored environment files

### Protect Secret values

Do not run commands that expose decoded credentials during demonstrations or screenshots.

Avoid:

```bash
kubectl get secret <SECRET_NAME> \
  --output jsonpath='{.data.DB_PWD}' \
  | base64 --decode
```

unless it is strictly necessary in a secure local environment.

### Rotate exposed credentials

Any credential pasted into:

* chat messages
* terminal recordings
* screenshots
* issue trackers
* CI logs

should be treated as exposed and rotated.

### Use a non-root container

The Java application container should run as a non-root user.

Example security context:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
```

The exact configuration must remain compatible with the application image.

### Restrict network access

For production environments, use Kubernetes NetworkPolicies to permit:

* Ingress Controller to Java application Pods
* Java application Pods to MySQL on port `3306`

Block unnecessary Pod-to-Pod communication.

---

## Production Recommendations

For production use:

1. Use immutable image tags such as a Git commit SHA or semantic version.

```yaml
image:
  tag: "1.0.0"
```

2. Do not use:

```yaml
image:
  tag: latest
```

3. Configure CPU and memory requests and limits.

4. Use at least two Java application replicas when the workload and database connection model support it.

5. Enable readiness and liveness probes.

6. Use a PodDisruptionBudget.

7. Store secrets outside Git.

8. Use TLS for external Ingress traffic.

9. Add NetworkPolicies.

10. Run container-image vulnerability scanning in CI.

11. Run:

```bash
helm lint
```

and:

```bash
helmfile diff
```

before deployment.

12. Monitor application logs, restart counts, latency, error rates, CPU usage, and memory usage.

13. Configure automated rollback or deployment failure detection in CI/CD.

14. Use separate values files for development, staging, and production.

15. Keep Helm as the source of truth for Helm-managed resources.

---

## Recommended Deployment Workflow

```bash
helm lint ./java-app-helm-chart
```

```bash
helmfile --selector name=java-app diff
```

```bash
helmfile --selector name=java-app apply
```

```bash
kubectl rollout status deployment/java-app \
  --namespace java-mysql \
  --timeout=180s
```

```bash
kubectl get pods,service,endpoints \
  --namespace java-mysql
```

```bash
kubectl logs deployment/java-app \
  --namespace java-mysql \
  --tail=100
```

```bash
kubectl port-forward \
  --namespace java-mysql \
  service/java-app \
  8082:8080
```

```bash
curl --include http://127.0.0.1:8082/
```

