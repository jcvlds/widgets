
# WidgetAPI Helm Chart

This chart deploys **WidgetAPI**, a simple internal HTTP upload/download service, onto Kubernetes.  
It is designed as a *reference application* for platform onboarding, showcasing recommended patterns
for:

- Gateway API–based ingress with `HTTPRoute`
- Persistent storage for stateful HTTP workloads
- External secret management (OCI Vault → ExternalSecrets → Kubernetes Secret)
- Network isolation (deny-all egress)
- GitOps deployment through FluxCD

This chart is published at:
oci://ghcr.io/jcvlds/charts/widgetapi

---

## Architecture

          +-------------------------+
          |        Gateway          |
          |  (public-gateway)       |
          +------------+------------+
                       |
                       |  HTTPRoute
                       v
           +-------------------------+
           |     Service (ClusterIP) |
           +------------+------------+
                       |
                    Pod/Deployment
          +---------------------------------+
          | Container: WidgetAPI            |
          | - Port 8080                     |
          | - Persistent data @ /widgetapi/data
          | - TOKEN from ExternalSecret     |
          +---------------------------------+
                       |
                       v
             PersistentVolumeClaim
                    (RWO)

---

## Features

### - Single-replica stateful web service  
Uses a `Deployment` and a `PersistentVolumeClaim` for `/widgetapi/data`.

### - Gateway API ingress  
Exposes the service using **HTTPRoute**, attaching to an existing cluster Gateway.

### - Complete external secret integration  
Relies on `ExternalSecret` + `SecretStore` (OCI Vault).  
The chart **does not create secrets**—they are supplied externally by the platform.

### - Strict network isolation  
A default deny-all **Egress** `NetworkPolicy` is included.

### - Production-ready YAML & GitOps-first design  
Chart supports FluxCD `HelmRelease` deployments out of the box.

---

## Installation

### Add the OCI registry (if necessary)

```sh
helm pull oci://ghcr.io/jcvlds/charts/widgetapi --version <version> 
```

## Example FluxCD HelmRelease
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: widgetapi
  namespace: widgets
spec:
  releaseName: widgetapi
  interval: 5m
  chart:
    spec:
      chart: widgetapi
      version: 0.1.x
      sourceRef:
        kind: HelmRepository
        name: widgetapi-charts
        namespace: flux-system
  values:
    gateway:
      host: widgetapi.example.com
    persistence:
      storageClassName: oci-bv
      size: 5Gi
    config:
      secretName: widgetapi-secret
      secretKeyToken: TOKEN
      uploadLimit: 10485760
```

## External Secret Requirement

This chart expects a secret containing the authentication token:
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: widgetapi-secret
  namespace: widgets
spec:
  secretStoreRef:
    name: oci-secretstore
    kind: SecretStore
  target:
    name: widgetapi-secret
    creationPolicy: Owner
  data:
    - secretKey: TOKEN
      remoteRef:
        key: widgetapi-token      # Name in OCI Vault
```
The Deployment reads this token as $TOKEN.

## Configuration
### Values Table
| Key                                 | Description                     | Default                        |
| ----------------------------------- | ------------------------------- | ------------------------------ |
| `replicaCount`                      | Number of pod replicas          | `1`                            |
| `image.repository`                  | Image repository                | `"mayth/simple-upload-server"` |
| `image.tag`                         | Image tag (optional)            | `""`                           |
| `service.port`                      | Internal service port           | `8080`                         |
| `gateway.enabled`                   | Enable Gateway API HTTPRoute    | `true`                         |
| `gateway.name`                      | Gateway name                    | `"public-gateway"`             |
| `gateway.namespace`                 | Gateway namespace               | `"networking"`                 |
| `gateway.listenerName`              | Listener sectionName            | `"http"`                       |
| `gateway.host`                      | External hostname               | `"widgetapi.example.com"`      |
| `persistence.storageClassName`      | Storage class                   | `""`                           |
| `persistence.size`                  | PVC size                        | `"1Gi"`                        |
| `config.secretName`                 | Name of secret providing TOKEN  | `"widgetapi-secret"`           |
| `config.secretKeyToken`             | Key inside the secret for token | `"TOKEN"`                      |
| `config.uploadLimit`                | Max upload limit                | `1048576`                      |
| `networkPolicy.enabled`             | Enable NetworkPolicy            | `true`                         |
| `networkPolicy.allowFromNamespaces` | Approved namespaces for ingress | `[]`                           |



## Testing
### Included Tests
This chart includes helm-unittest suites validating:

- HTTPRoute attaches to the correct Gateway
- PVC renders when persistence enabled
- NetworkPolicy correctly denies all egress
- Deployment mounts PVC correctly and injects TOKEN env var

Run tests:
```sh
helm unittest .
```
(Requires plugin: helm plugin install https://github.com/helm-unittest/helm-unittest)

## Smoke Testing Instructions (manual)

After deploy:

Upload a file
```sh
printf "sample data\n" | \
curl -XPUT -Ffile=@- \
  "https://widgetapi.example.com/files/demo.txt?token=<TOKEN>"
```

Retrieve it
```sh
curl "https://widgetapi.example.com/files/demo.txt?token=<TOKEN>"
```

Verify persistence through upgrades
```sh
kubectl rollout restart deployment/widgetapi -n widgets
curl "https://widgetapi.example.com/files/demo.txt?token=<TOKEN>"
```

## Operational Notes
### Persistence
Data is stored under /widgetapi/data on the PVC.
It remains across:
- Pod restarts
- Rescheduling to another node
- Helm upgrades
- Image tag changes

### Secret rotation
Rotation follows this chain:
- Update value in OCI Vault
- ExternalSecret controller syncs → Kubernetes Secret updates
- Deployment automatically restarts with new TOKEN

### NetworkPolicy
Outbound connections are blocked by default.
Only inbound HTTP traffic via the Gateway is allowed.

## Versioning

Chart versions follow semver, located at:
```ruby
oci://ghcr.io/jcvlds/charts/widgetapi
```

## Summary

This chart serves as the “golden example” for app teams moving from VMs/docker-compose to Kubernetes on the platform. Its intent is to demonstrate:
- Secure defaults
- GitOps readiness
- Cloud-native networking (Gateway API)
- External secret management
- Persistent state handling

Feel free to fork this chart as a template for future applications.

---

# Helm-Unittest Suite (`tests/` directory)

Create:
- tests/
- deployment_test.yaml
- pvc_test.yaml
- httproute_test.yaml
- networkpolicy_test.yaml

---

## `tests/deployment_test.yaml`

```yaml
suite: Deployment Rendering
templates:
  - templates/deployment.yaml

tests:
  - it: should include PVC volume mount
    asserts:
      - equal:
          path: spec.template.spec.volumes[0].persistentVolumeClaim.claimName
          value: widgetapi-widgetapi-data

  - it: should use correct container args
    asserts:
      - contains:
          path: spec.template.spec.containers[0].args
          content: "-document_root=/widgetapi/data"

  - it: should include TOKEN env var from secret
    asserts:
      - equal:
          path: spec.template.spec.containers[0].env[0].valueFrom.secretKeyRef.key
          value: TOKEN
```
## `tests/pvc_test.yaml`
```yaml
suite: PVC Rendering
templates:
  - templates/pvc.yaml

tests:
  - it: should render PVC by default
    asserts:
      - equal:
          path: spec.accessModes[0]
          value: ReadWriteOnce

  - it: should not render PVC when existingClaim is set
    set:
      persistence.existingClaim: "my-existing"
    asserts:
      - isNull:
          path: spec
```

## `tests/httproute_test.yaml`
```yaml
suite: HTTPRoute Rendering
templates:
  - templates/httproute.yaml

tests:
  - it: should include host from values
    set:
      gateway.host: widgetapi.test.com
    asserts:
      - equal:
          path: spec.hostnames[0]
          value: widgetapi.test.com

  - it: should reference correct Gateway
    asserts:
      - equal:
          path: spec.parentRefs[0].name
          value: public-gateway
```

## `tests/networkpolicy_test.yaml`
```yaml
suite: NetworkPolicy Rendering
templates:
  - templates/networkpolicy.yaml

tests:
  - it: should deny all egress
    asserts:
      - lengthEqual:
          path: spec.egress
          value: 0

  - it: should allow ingress on port 8080
    asserts:
      - equal:
          path: spec.ingress[0].ports[0].port
          value: 8080
```

# Optional CI: GitHub Actions Workflow (.github/workflows/helm-ci.yaml)
```yaml
name: Helm Lint & Tests

on:
  pull_request:
  push:
    branches: [ main ]

jobs:
  helm-tests:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Install Helm
        uses: azure/setup-helm@v4

      - name: Install helm-unittest plugin
        run: helm plugin install https://github.com/helm-unittest/helm-unittest

      - name: Helm Lint
        run: helm lint .

      - name: Run Unittests
        run: helm unittest .
```
