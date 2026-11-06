<p align="center">
  <a href="https://primedetect.live"><img src="static/image/Prime Detect_x2__1280px.png" alt="Prime Detect"></a>
</p>

<p align="center">
  <a href="https://socprime.com/detectflow/#demo-form"><img src="static/image/Button Contact sales.png" alt="Contact sales" height="48"></a>
  &nbsp;&nbsp;&nbsp;
  <a href="https://www.youtube.com/watch?v=FYI4y1YiW7c"><img src="static/image/Button Youtube.png" alt="Watch video" height="48"></a>
</p>

# Full Deployment
_For the general architecture and other info about Prime Detect, see [README.md](README.md). DetectFlow is being renamed to Prime Detect. The new name will be reflected across all resources and assets in the next release._

Deployment should be performed by a person with expertise in Kubernetes and admin access to the Kubernetes Cluster provided for this project.

Deployment consists of four parts:

- Apache Flink (according to its official deployment recommendations)
- PostgreSQL
- Apache Kafka
- Prime Detect itself

## Apache Flink Deployment

Install Cert Manager according to https://cert-manager.io/docs/installation/

Run the following command for creating NameSpace:

```
> kubectl create ns flink
```

Out: `namespace/flink created`

### Deploy PostgreSQL for Prime Detect

Run the following command:

```
> kubectl apply -f postgresql.yml -n flink
```

### Deploy Apache Kafka (One node) for Prime Detect

Run the following command:

```
> kubectl apply -f kafka-kraft.yml -n flink
```

Get acquainted with the key concepts of Apache Flink Kubernetes Operator here: https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-main/

### Deploy Apache Flink Kubernetes Operator using Helm.

A Helm chart manages the installation of the operator. To install the operator (and also the Helm chart) to a specific namespace, run the following command:

```
> helm repo add flink-operator-repo https://downloads.apache.org/flink/flink-kubernetes-operator-1.13.0/
```

Out: `"flink-operator-repo" has been added to your repositories`

```
> helm repo update
```

Out:

```
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "flink-operator-repo" chart repository

...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
```

```
> helm install flink-kubernetes-operator flink-operator-repo/flink-kubernetes-operator --namespace flink-operator --create-namespace
```

Out:

```
NAME: flink-kubernetes-operator
LAST DEPLOYED: Thu Feb  5 12:07:08 2026
NAMESPACE: flink-operator
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

Source: https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-main/docs/operations/helm/

### Deploy ConfigMap with static configuration for Flink Sigma Detector.

Open the `flink-configmap.yaml` from the installation package and specify values for the following parameters:

```
namespace: flink
```

For all the other parameters, keep the default values.

Run the following command:

```
> kubectl apply -f flink-configmap.yaml -n flink
```

Out: `configmap/flink-config created`

### Deploy Secrets for Flink Sigma Detector.

Open the `flink-secret.yaml` from the installation package and specify the Base64 values for the following parameters (if needed, either leave by default):

```
KAFKA_BOOTSTRAP_SERVERS:
KAFKA_AUTH_METHOD:
```

Run the following command:

```
> kubectl apply -f flink-secret.yaml -n flink
```

Out: `secret/flink-secret created`

### Deploy PersistentVolumeClaims for Flink State Storage.

Open the `flink-pvcs.yaml` from the installation package and specify values for the following parameters in each `name:` block:

```
namespace:
storage:
storageClassName:
```

Run the following command:

```
> kubectl apply -f flink-pvcs.yaml -n flink
```

Out:

```
persistentvolumeclaim/flink-checkpoints-pvc created
persistentvolumeclaim/flink-ha-pvc created
persistentvolumeclaim/flink-savepoints-pvc created
```

Optional for PVC: add a selector/label for nodes by running the following command:

```
> kubectl label nodes NAME spec-node-ns-pod-disk=pvc-security
```

### Deploy RBAC Configuration for Flink ServiceAccount. It grants permissions for Flink to create and manage TaskManager pods.

Open the `flink-rbac.yaml` from the installation package and specify a value for the following parameter:

```
Namespace: flink
```

Run the following command:

```
> kubectl apply -f flink-rbac.yaml -n flink
```

Out:

```
serviceaccount/flink created
role.rbac.authorization.k8s.io/flink created
rolebinding.rbac.authorization.k8s.io/flink-role-binding created
role.rbac.authorization.k8s.io/event-reader created
rolebinding.rbac.authorization.k8s.io/grant-event-reader created
```

### Docker Images Pulling

Some of the images are stored at the Private SOC Prime Docker Image Registry Repo at Docker HUB.

SOC Prime will provide login credentials for Private Docker HUB.

Set environments:

```bash
export DOCKERHUB_USER="your_username" 
export DOCKERHUB_TOKEN="your_token"
```

Check environment settings:

```bash
echo $DOCKERHUB_USER
echo $DOCKERHUB_TOKEN
```

Set credentials for namespace flink

```bash
kubectl create secret docker-registry dockerhub-creds \
    --docker-server=https://index.docker.io/v1/ \
    --docker-username="${DOCKERHUB_USER}" \
    --docker-password="${DOCKERHUB_TOKEN}" \
    -n flink \
    --dry-run=client -o yaml | kubectl apply -f -
```

## Prime Detect Deployment

Prime Detect consists of two parts:

- Admin Panel backend
- Admin Panel UI

### Admin Panel Backend Deployment

#### Deploy ConfigMap with static configuration for Admin Panel Backend.

Open `admin-panel-be-configmap.yaml` from the installation package and specify a value for the following parameter:

```
KAFKA_BOOTSTRAP_SERVERS:
```

Set IP/Hostname and port of the Kafka Server

Run the following command:

```
> kubectl apply -f admin-panel-be-configmap.yaml -n flink
```

Out: `configmap/admin-panel-be-config created`

#### Deploy a Secret with sensitive data configuration for Admin Panel Backend.

Open `admin-panel-be-secret.yaml` from the installation package and specify a value for the following parameter:

```
# DATABASE_URL has the following format: postgresql+asyncpg://user:password@postgres_url:postgres_port/db_name
```

It should be converted to Base64

```
DATABASE_URL:
```

Run the following command:

```
> kubectl apply -f admin-panel-be-secret.yaml -n flink
```

Out: `secret/admin-panel-be-secret created`

#### Deploy PersistentVolumeClaims for Admin Panel Backend using `admin-panel-be-pvc.yaml` from the installation package.

Run the following command:

```
> kubectl apply -f admin-panel-be-pvc.yaml -n flink
```

Out: `persistentvolumeclaim/admin-panel-metrics-pvc created`

#### Create a Deployment resource for Admin Panel Backend using `admin-panel-be-deployment.yaml` from the installation package.

Run the following command:

```
> kubectl apply -f admin-panel-be-deployment.yaml -n flink
```

Out: `deployment.apps/admin-panel-be created`

#### Deploy Network Service resource for Admin Panel Backend using `admin-panel-be-service.yaml` from the installation package.

Run the following command:

```
> kubectl apply -f admin-panel-be-service.yaml -n flink
```

Out: `service/admin-panel-be created`

Check that Admin-Panel-Backend-API is running and accessible by following the next link:
`http://network_service_ip:8000`

### Admin Panel UI Deployment

#### Deploy ConfigMap with a static configuration for Admin Panel UI.

Open `admin-panel-ui-configmap.yaml` from the installation package and specify a value for the following parameter:

> **Note:** NAME.NAMESPACE.CLUSTER_DOMAIN_NAME (default: svc.cluster.local)

```
VITE_PREVIEW_ALLOWED_HOSTS:
```

Set internal hostname of admin-panel-be, for example: `admin-panel-be.flink.svc.cluster.local`

Run the following command:

```
> kubectl apply -f admin-panel-ui-configmap.yaml -n flink
```

Out: `configmap/admin-panel-ui-config created`

#### Create a Deployment resource for Admin Panel UI using `admin-panel-ui-deployment.yaml` from the installation package.

Run the following command:

```
> kubectl apply -f admin-panel-ui-deployment.yaml -n flink
```

Out: `deployment.apps/admin-panel-ui created`

#### Deploy Network Service resource for Admin Panel UI using `admin-panel-ui-service.yaml` from the installation package.

Run the following command:

```
> kubectl apply -f admin-panel-ui-service.yaml -n flink
```

Out: `service/admin-panel-ui created`

Check that Admin-Panel-UI is running and accessible by following the next link:
`http://network-service-ip:4173`

Default:

- Email: admin@soc.local
- Password: admin

## Prime Detect Agent

Two manifests for the LLM-based Prime Detect agent - a ConfigMap for runtime settings and a Secret for the API key.

### detectflow-agent-config-configmap.yaml

ConfigMap `detectflow-agent-config` — non-sensitive runtime settings for the LLM agent.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: detectflow-agent-config
  namespace: flink
data:
  LLM_BASE_URL: http://localhost:8000/v1/   # LLM endpoint the agent calls
  LLM_CALLS_PER_MINUTE: "60"                 # rate limit — max requests/min
  LLM_MAX_RETRIES: "3"                       # retry attempts on a failed call
  LLM_MAX_TOKENS: "2048"                     # max tokens per completion
  LLM_MODEL: model                           # model name/identifier to request
  LLM_PROMPT_CHAR_BUDGET: "24000"            # max prompt size in characters
  MAX_CONCURRENT_LLM_REQUESTS: "1"           # concurrency cap for in-flight calls
```

> **Note:** `LLM_BASE_URL` points at `localhost:8000` — the agent expects the LLM endpoint reachable as a sidecar/local proxy in its own pod, not the cluster-wide `admin-panel-be` service. Confirm this matches the actual agent deployment topology before rollout.

### detectflow-agent-secret.yaml

Secret `detectflow-agent-secrets` — holds the LLM API key. Ships empty in the manifest — must be populated with a base64-encoded value before applying, otherwise the agent will fail to authenticate to the LLM endpoint.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: detectflow-agent-secrets
  namespace: flink
data:
  LLM_API_KEY: ""   # base64-encoded API key — empty by default, fill before apply
type: Opaque
```

To populate and apply without hand-encoding base64:

```bash
kubectl -n flink create secret generic detectflow-agent-secrets \
  --from-literal=LLM_API_KEY='<your-api-key>' \
  --dry-run=client -o yaml | kubectl apply -f -
```

## Enabling a Self-Signed LLM CA Certificate for the Agent Job

Follow this section in cases where the LLM is configured to work over HTTPS.

### ConfigMap - Full Example

This is the ConfigMap the backend reads for agent-job settings (default name `detectflow-agent-config`, or whatever `DETECTFLOW_AGENT_CONFIGMAP_NAME` is set to on the backend). `SSL_CERT_FILE` is the key relevant to this guide; the rest are the agent’s normal LLM tuning knobs, shown for context:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: detectflow-agent-config
  namespace: <namespace where agent-job runs>
data:
  # LLM endpoint the agent calls for enrichment
  LLM_BASE_URL: "https://<llm-host>:<port>/v1/"
  LLM_MODEL: "<model-name>"
  LLM_CALLS_PER_MINUTE: "60"
  LLM_MAX_RETRIES: "3"
  LLM_MAX_TOKENS: "2048"
  LLM_PROMPT_CHAR_BUDGET: "24000"
  MAX_CONCURRENT_LLM_REQUESTS: "1"
  # The key this guide is about - path to the CA bundle inside the container.
  # Directory part (/etc/ssl/agent-ca) becomes the volume mountPath, filename
  # part (ca.pem) becomes the Secret key the backend projects. A path
  # directly under / is rejected (would shadow the container root).
  SSL_CERT_FILE: "/etc/ssl/agent-ca/ca.pem"
  # Testing-only escape hatch - skips TLS verification entirely. Leave unset
  # (or "true") in production.
  # LLM_SSL_VERIFY: "false"
```

### Secret - Full Example

This is the Secret the backend reads for agent-job credentials (default name `detectflow-agent-secrets`, or `DETECTFLOW_AGENT_SECRET_NAME`). The CA cert key must match the filename from `SSL_CERT_FILE` above exactly (`ca.pem` here):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: detectflow-agent-secrets
  namespace: <namespace where agent-job runs>
type: Opaque
stringData:
  # API key for LLM_BASE_URL, if it requires auth. Empty ("") is valid when
  # the endpoint needs none (e.g. a local/internal LLM).
  LLM_API_KEY: ""
  # CA bundle for LLM_BASE_URL's self-signed/internal certificate. Plain PEM
  # text - stringData handles the encoding, no base64 needed. Key name must
  # equal basename(SSL_CERT_FILE) from the ConfigMap above.
  ca.pem: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
```

### Certificate Must Carry a SAN

The cert needs a subjectAltName matching the LLM endpoint’s actual hostname/IP - a CN-only certificate is rejected by Python’s ssl module (IP address mismatch) even once correctly mounted and trusted. Check:

```bash
openssl x509 -in ca.pem -noout -text | grep -A1 "Subject Alternative Name"
```

If missing, regenerate the cert with a SAN entry for the LLM server’s real address.

### Verify

Recreate the agent-job pod, then:

```bash
kubectl exec -it <agent-job-pod> -c flink-main-container -- cat /etc/ssl/agent-ca/ca.pem
kubectl exec -it <agent-job-pod> -c flink-main-container -- \
  curl --cacert /etc/ssl/agent-ca/ca.pem https://<llm-host>:<port>/v1/ -v
```

## Upgrade Procedure

All five images live in the same Docker Hub repo (`socprimecom/detectflow`), tagged by component.

### New Images in v2.3.2

| Component | Image |
|---|---|
| Admin Panel Backend | `socprime/detectflow-oss:admin-panel-be-oss.v2.3.2` |
| Admin Panel UI | |
| Flink Sigma Detector | `socprime/detectflow-oss:flink-sigma-detector-oss.v2.3.2` |
| DF Agent job | `socprime/detectflow-oss:df-agent-oss.v2.3.2` |
| DF Correlation job | `socprime/detectflow-oss:df-correlation-oss.v2.3.2` |

## Appendix A

In terms of calculating required resources, the following components of Prime Detect should be considered, with individual requirements for each component summed up to estimate the total amount:

- **Admin Panel** is the control center to manage event matching processes. This component requires at least 2 CPUs and 4 GB RAM
- **Task Manager**, the default component to control the operation of pipelines (based on match nodes, the Apache Flink technology). This component requires at least 1 CPU and 4 GB RAM
- **Match Node**, a pipeline controlled via Admin Panel that matches detection rules to Apache Kafka topic events. The number of Match Nodes (pipelines) is custom, and you should estimate the required Kubernetes cluster resources based on the number of pipelines you're going to create, the data you're processing (EPS), and how many Sigma rules you'll apply. Reach out to SOC Prime for a detailed resource calculation.
