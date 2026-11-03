# VProfile Helm Chart

A Helm chart for deploying the VProfile application and supporting services to Kubernetes. The chart templates application, MySQL, Memcached, and RabbitMQ Deployments; ClusterIP Services; a MySQL PersistentVolumeClaim; Kubernetes Secrets; and an optional AWS Application Load Balancer Ingress.

> **Scope:** This README documents the chart files in this repository. It does not claim the chart has been installed or tested against a live Kubernetes cluster. Review and validate the chart values and cloud-specific settings for your own cluster before deployment.

## Table of contents

- [Project overview](#project-overview)
- [Helm and Kubernetes architecture](#helm-and-kubernetes-architecture)
- [Technology stack](#technology-stack)
- [Repository structure](#repository-structure)
- [Chart metadata and values](#chart-metadata-and-values)
- [Kubernetes templates](#kubernetes-templates)
- [Prerequisites and Helm installation](#prerequisites-and-helm-installation)
- [Lint and render validation](#lint-and-render-validation)
- [Install, upgrade, and uninstall](#install-upgrade-and-uninstall)
- [Customizing values](#customizing-values)
- [Verify a Kubernetes deployment](#verify-a-kubernetes-deployment)
- [Troubleshooting](#troubleshooting)
- [Security considerations](#security-considerations)

## Project overview

The chart is named `vprofile` and is a Helm v2 application chart. It deploys the VProfile web application alongside database, cache, and message-broker workloads, and can optionally create an Ingress and image-pull Secret. The chart is located at the repository root; **all commands below use the current directory (`.`) as the chart path**.

The chart templates resources, but a successful installation depends on a working Kubernetes cluster, suitable storage provisioning, and (when Ingress is enabled) a compatible Ingress implementation. Resource rendering or a successful Helm command alone does not demonstrate that the application is healthy or reachable.

## Helm and Kubernetes architecture

Helm renders `Chart.yaml`, `values.yaml`, and the templates into Kubernetes manifests. Kubernetes then schedules the requested workloads and manages the resulting resources.

At the default values, the intended connections are:

```text
External HTTP/HTTPS
   │
   ▼
Ingress vpro-ingress (enabled; class alb; host from ingress.host)
   │ HTTP to vproapp-service:8080
   ▼
VProfile app Deployment: vproapp (1 replica, container port 8080)
   ├── vprodb Service → MySQL Deployment (1 replica, container port 3306)
   │                    └── PVC db-pv-claim mounted at /var/lib/mysql
   ├── vprocache01 Service → Memcached Deployment (1 replica, port 11211)
   └── vpromq01 Service → RabbitMQ Deployment (1 replica, port 5672)
```

All four services are `ClusterIP` services. The app Deployment has init containers that wait for DNS resolution of the `vprodb` and `vprocache01` Services. These checks do not test database or cache protocol readiness. The app template does not have a RabbitMQ wait init container.

The Ingress template is configured for the `alb` class and includes AWS ALB-specific annotations, including an internet-facing scheme, IP targets, a certificate ARN, HTTP/HTTPS listeners, and an HTTPS redirect. The cluster must have the corresponding Ingress controller and cloud permissions/configuration; this repository does not provision them.

## Technology stack

| Component | Configuration in this chart |
|---|---|
| Packaging | Helm chart API `v2`, chart version `0.1.0` |
| Orchestration | Kubernetes Deployments, Services, PersistentVolumeClaim, Secret, and Ingress API `networking.k8s.io/v1` |
| Application runtime | Configurable application image and tag; default container port `8080` |
| Database | Configurable MySQL image; default database container/service port `3306`; persistent volume claim |
| Cache and messaging | Memcached on port `11211`; RabbitMQ on port `5672` |
| Ingress | AWS ALB-specific annotations and `alb` Ingress class when enabled |
| Optional image pull credentials | Conditional `kubernetes.io/dockerconfigjson` Secret and app/database `imagePullSecrets` configuration |

This describes the chart's declared resources and defaults; it does not guarantee image versions, service compatibility, or cluster integrations.

## Repository structure

```text
.
├── Chart.yaml                         # Chart metadata and version
├── values.yaml                        # Default image, replica, port, storage, ingress, and secret values
└── templates/
    ├── app-deployment.yaml            # VProfile application Deployment and service-wait init containers
    ├── db-deployment.yaml             # MySQL Deployment, Secret reference, PVC mount, cleanup init container
    ├── dockerregistry-secret.yaml     # Optional registry authentication Secret
    ├── ingress.yaml                   # Optional AWS ALB Ingress
    ├── mc-deployment.yaml             # Memcached Deployment
    ├── pvc.yaml                       # MySQL PersistentVolumeClaim
    ├── rmq-deployment.yaml            # RabbitMQ Deployment
    ├── secret.yaml                    # Opaque database and RabbitMQ password Secret
    └── services.yaml                  # App, MySQL, Memcached, and RabbitMQ ClusterIP Services
```

## Chart metadata and values

### `Chart.yaml`

`Chart.yaml` declares:

- `apiVersion: v2` — Helm chart format for Helm 3.
- `name: vprofile` and `type: application`.
- Chart `version: 0.1.0` — the chart package version.
- `appVersion: "1.0.0"` — descriptive application version metadata; templates do not use it to choose the app image tag.

### `values.yaml`

The checked-in defaults are:

| Value | Default | Used for |
|---|---|---|
| `defaultUser` | `guest` | RabbitMQ default username |
| `app.image`, `app.tag` | `442426880114.dkr.ecr.us-east-1.amazonaws.com/vprofileappimg`, `75e7322` | App image and tag |
| `app.replicas`, `app.containerPort`, `app.servicePort` | `1`, `8080`, `8080` | App replica count and ports |
| `db.image`, `db.tag` | `vprocontainers/vprofiledb`, `latest` | MySQL image and tag |
| `db.replicas`, `db.containerPort`, `db.servicePort` | `1`, `3306`, `3306` | MySQL replica count and ports |
| `db.storageClass`, `db.storageSize` | `gp2`, `3Gi` | PVC storage class and requested size |
| `memcached.image`, `memcached.tag` | `memcached`, `latest` | Memcached image and tag |
| `memcached.replicas`, `memcached.containerPort`, `memcached.servicePort` | `1`, `11211`, `11211` | Memcached replica count and ports |
| `rabbitmq.image`, `rabbitmq.tag` | `rabbitmq`, `latest` | RabbitMQ image and tag |
| `rabbitmq.replicas`, `rabbitmq.containerPort`, `rabbitmq.servicePort` | `1`, `5672`, `5672` | RabbitMQ replica count and ports |
| `initcontainers.image` | `busybox` | Image used for app DNS-wait init containers |
| `initcontainers.command` | `sh`, `-c` | Present in values; the deployment template defines its own commands and does not reference this value |
| `ingress.enabled`, `ingress.host`, `ingress.servicePort` | `true`, `vprofile.glossplus.in`, `8080` | Optional Ingress, hostname, backend port |
| `secrets.dbPassword`, `secrets.rmqPassword` | `vprodbpass`, `guest` | Values rendered into `app-secret` |
| `dockerregistry.enabled` | `false` | Controls registry Secret and app/database pull-secret use |
| `dockerregistry.server`, `username`, `password`, `email` | Empty strings | Registry authentication inputs when enabled |

The chart's defaults include a public hostname, an AWS account/certificate-specific Ingress annotation, image references, and placeholder/development credentials. Review and override them for your environment. The values for credentials should not be treated as secure merely because Kubernetes stores them in a Secret.

## Kubernetes templates

- **`templates/app-deployment.yaml`** — creates the `vproapp` Deployment, using `app.image:app.tag`, the configured replica count and container port, and `imagePullPolicy: Always`. It conditionally adds `docker-registry-secret` as an image pull secret. Two BusyBox init containers wait for DNS entries for `vprodb` and `vprocache01` in the pod namespace before starting the application container.
- **`templates/db-deployment.yaml`** — creates the `vprodb` Deployment from `db.image:db.tag`. The MySQL root password is read from key `db-pass` in `app-secret`. The container mounts claim `db-pv-claim` at `/var/lib/mysql`; a BusyBox init container removes `/var/lib/mysql/lost+found` from the mounted volume. Registry pull credentials are conditionally attached when enabled.
- **`templates/dockerregistry-secret.yaml`** — conditionally creates Secret `docker-registry-secret` of type `kubernetes.io/dockerconfigjson` when `dockerregistry.enabled` is true. Its credentials come from the corresponding `dockerregistry.*` values. The app and database Deployments reference this Secret conditionally; the Memcached and RabbitMQ templates do not.
- **`templates/ingress.yaml`** — conditionally creates Ingress `vpro-ingress` using `networking.k8s.io/v1`. It routes `/` for `ingress.host` to `vproapp-service` on `ingress.servicePort`, with the AWS ALB annotations described above.
- **`templates/mc-deployment.yaml`** — creates Deployment `vpromc` from the configured Memcached image, replica count, and container port.
- **`templates/pvc.yaml`** — creates the `db-pv-claim` PersistentVolumeClaim with `ReadWriteOnce`, `db.storageClass`, and `db.storageSize`.
- **`templates/rmq-deployment.yaml`** — creates Deployment `vpromq01` from the configured RabbitMQ image and port. It sets `RABBITMQ_DEFAULT_USER` from `defaultUser` and gets `RABBITMQ_DEFAULT_PASS` from `app-secret` key `rmq-pass`.
- **`templates/secret.yaml`** — creates Opaque Secret `app-secret` with `db-pass` and `rmq-pass` from `secrets.dbPassword` and `secrets.rmqPassword` using `stringData`.
- **`templates/services.yaml`** — creates four `ClusterIP` Services: `vproapp-service`, `vprodb`, `vprocache01`, and `vpromq01`; their selectors and named target ports route to the corresponding workload pods.

## Prerequisites and Helm installation

You need:

- A Kubernetes cluster and a configured `kubectl` context with permission to create the chart's resources.
- Helm 3, consistent with the chart's `apiVersion: v2`.
- A provisionable StorageClass matching `db.storageClass` (default `gp2`) for the database PVC.
- Reachable image registries and permission to pull each selected image. If private registry credentials are needed, configure the optional Docker registry Secret or use a separately managed image pull Secret.
- If leaving Ingress enabled, a compatible AWS Load Balancer Controller/ALB integration, a valid certificate for the selected hostname, and appropriate AWS-side setup. The chart references an explicit certificate ARN in the template; changing `ingress.host` alone does not change that ARN.

Install Helm using the official instructions at [helm.sh/docs/intro/install](https://helm.sh/docs/intro/install/). Verify the tools and cluster context:

```bash
helm version
kubectl version --client
kubectl config current-context
kubectl cluster-info
```

## Lint and render validation

From the repository root, validate the chart using `.` as the chart path:

```bash
helm lint .
helm template vprofile . --namespace vprofile
```

To inspect the manifests with the default Ingress setting and save the rendered output:

```bash
helm template vprofile . --namespace vprofile > rendered.yaml
```

To validate with overrides without changing chart files:

```bash
helm template vprofile . \
  --namespace vprofile \
  --set ingress.enabled=false \
  --set app.replicas=2
```

`helm template` renders manifests locally; it does not submit them to Kubernetes. Consider validating the rendered output against your target Kubernetes version and cluster policies before installation.

## Install, upgrade, and uninstall

Review and override the chart's default values before installation. To install into a dedicated namespace:

```bash
helm upgrade --install vprofile . \
  --namespace vprofile \
  --create-namespace
```

To use a values file created for your deployment:

```bash
helm upgrade --install vprofile . \
  --namespace vprofile \
  --create-namespace \
  --values my-values.yaml
```

Upgrade a release after changing a local values file or chart values:

```bash
helm upgrade vprofile . \
  --namespace vprofile \
  --values my-values.yaml
```

Uninstall the release:

```bash
helm uninstall vprofile --namespace vprofile
```

The database PVC may outlive the Helm release depending on Kubernetes/Helm resource behavior and cluster storage configuration. Check for retained claims and data before cleanup; do not delete persistent data unless that is intentional.

## Customizing values

Use a separate, untracked or appropriately protected values file rather than editing the chart defaults for environment-specific settings. For example:

```yaml
app:
  image: example.invalid/team/vprofile
  tag: "replace-with-approved-tag"
  replicas: 1

db:
  storageClass: your-storage-class
  storageSize: 5Gi

ingress:
  enabled: false
```

The image above is an illustrative placeholder and must be replaced with a real, accessible image. Override additional values using `--set` for non-sensitive values, or `--values` for repeatable configuration. For credentials, avoid command-line arguments and plaintext values files in source control; prefer a secret-management approach appropriate to your cluster. The chart currently renders its own `app-secret` from the `secrets.*` values, so ensure any method used to supply these values does not expose them in Git, shell history, CI logs, or release metadata.

Check which values Helm will use and what manifests result before applying changes:

```bash
helm get values vprofile --namespace vprofile --all
helm template vprofile . --namespace vprofile --values my-values.yaml
```

## Verify a Kubernetes deployment

After installation, inspect the release and resources:

```bash
helm status vprofile --namespace vprofile
helm list --namespace vprofile
kubectl get deployments,pods,services,pvc,ingress,secrets \
  --namespace vprofile
```

Watch pod startup and events:

```bash
kubectl get pods --namespace vprofile --watch
kubectl get events --namespace vprofile --sort-by=.lastTimestamp
```

Inspect an individual pod or workload when it is not ready:

```bash
kubectl describe pod <pod-name> --namespace vprofile
kubectl logs deployment/vproapp --namespace vprofile
kubectl logs deployment/vprodb --namespace vprofile
```

Check Ingress status and details if enabled:

```bash
kubectl describe ingress vpro-ingress --namespace vprofile
kubectl get ingress vpro-ingress --namespace vprofile
```

An Ingress address may depend on the cluster's external controller and cloud provisioning. The existence of an Ingress resource does not itself prove external access.

## Troubleshooting

| Symptom | What to check |
|---|---|
| `helm lint` or template rendering fails | Confirm commands run from the chart root and use `.`; inspect YAML indentation and any overridden values/types. |
| App pod waits in init | The app init containers wait for DNS for `vprodb` and `vprocache01`. Check that the Services exist, their selectors match ready pods, and CoreDNS is functioning. These checks do not establish service readiness. |
| Pod reports `ImagePullBackOff` | Confirm image name/tag, registry reachability, and pull permissions. If using the optional registry Secret, set its values and `dockerregistry.enabled: true`; check that the secret was rendered and referenced. |
| MySQL pod is pending | Check PVC status, storage class availability, access mode support, and cluster storage provisioner events. |
| MySQL fails to initialize | Inspect pod logs/events, validate the database image and secret value, and check PVC contents and permissions. The cleanup init container operates on the persistent data directory. |
| RabbitMQ fails to start or authenticate | Verify `app-secret` contains the expected `rmq-pass` and check the rendered `defaultUser` and RabbitMQ image configuration. |
| Ingress receives no address or is unreachable | Confirm an ALB-compatible controller is installed and authorized, the `alb` class/annotations are supported, the certificate ARN and host are correct, and the backend Service/endpoints exist. |
| Deployment is created but app is unhealthy | Review pod events and logs, service endpoint readiness, and required application dependencies. Resource creation alone is not an application health check. |

Useful diagnostic commands include `kubectl describe`, `kubectl logs`, and `kubectl get events` in the release namespace. Avoid sharing output that contains credentials or other sensitive configuration.

## Security considerations

- `values.yaml` contains default database and RabbitMQ passwords. Replace them before any non-disposable use and do not commit real credentials in values files.
- Kubernetes Secret objects are not automatically encrypted merely because they are called Secrets. Restrict RBAC access, enable cluster secret encryption at rest where available, and use an approved external secrets/secret-management integration when appropriate.
- The current `secret.yaml` template renders credentials with `stringData`; values supplied to Helm can also be present in Helm release data. Do not pass production secrets through public Git history, command-line flags, CI logs, or unprotected release state.
- `dockerregistry-secret.yaml` accepts a registry username and password as chart values and constructs a Docker config Secret. Protect those inputs and the resulting Secret. Consider workload identity or an approved credential provider instead of long-lived registry passwords where supported.
- The default Ingress is internet-facing and has a certificate ARN embedded in the template. Confirm that internet exposure, domain, certificate, listeners, and TLS redirect meet your requirements before enabling it.
- The chart uses mutable tags such as `latest` for database, Memcached, and RabbitMQ by default. Pin reviewed versions or immutable digests for reproducibility and supply-chain control.
- The database PVC uses `ReadWriteOnce`; plan backups, retention, recovery, and storage-class behavior separately. A PVC is not a backup strategy.
- The chart does not define NetworkPolicies, resource requests/limits, readiness/liveness probes, or pod security settings in the inspected templates. Evaluate and add appropriate controls for your cluster and workload before production use.
