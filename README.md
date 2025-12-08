
# WidgetAPI Helm Chart

This chart deploys **WidgetAPI** onto Kubernetes (K8s).  
This chart uses recommended K8s deployment patterns including:

- Persistent storage across rescheduling and version changes
- Gateway API–based ingress with HTTPRoute
- Network policy allowing ingress from inside the cluster (currently from all namespaces), and enabling app isolation (deny-all egress)
- External secret management (Cloud Vault → K8s ExternalSecrets → Kubernetes Secret)
- GitOps deployment through FluxCD

### This chart is published at:
`oci://ghcr.io/jcvlds/charts/widgetapi`
### This chart is deployed to JC's personal K8s cluster and the app is accessible at:
`https://widgetapi.juancarlosvaldes.com`

---

## Architecture
https://github.com/jcvlds/widgets/blob/main/WidgetAPI_K8s_Architecture.png
 
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

### Helm Chart OCI registry published at:
```
ghcr.io/jcvlds/charts/widgetapi
```

## External Secret Requirement

This chart expects a secret containing the authentication token under secret 'widgetapi-token', key 'TOKEN'

The Deployment reads this token from the secret.

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

## Smoke Testing Instructions (manual)

After deploy:

Upload a file
```sh
curl -XPUT -Ffile=@./demo.txt \
  "https://widgetapi.juancarlosvaldes.com/files/demo.txt?token=TOKEN"

or

curl -v \
  -H "Authorization: Bearer TOKEN" \
  -F "file=@./demo.txt" \
  https://widgetapi.juancarlosvaldes.com/upload
```

Retrieve it
```sh
curl "https://widgetapi.juancarlosvaldes.com/files/demo.txt?token=TOKEN"

or

curl -v -H "Authorization: Bearer TOKEN" https://widgetapi.juancarlosvaldes.com/files/demo.txt
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
- Update value in Cloud Vault
- ExternalSecret controller syncs → Kubernetes Secret updates
- Deployment automatically restarts with new TOKEN

### NetworkPolicy
Outbound connections are blocked by default.
Inbound HTTP traffic via the Gateway and internal namespaces is allowed.

## Versioning

Chart versions follow semver, located at:
```
oci://ghcr.io/jcvlds/charts/widgetapi
```

---
