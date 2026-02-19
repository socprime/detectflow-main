# SOC Prime DetectFlow OSS

Real-Time threat detection at the stream layer (Apache Kafka + Apache Flink). Detect threats in-flight – before logs hit SIEM:
- Solution that runs detection logic on event streaming
- Tags/enriches events in real time
- Routes only what matters downstream
- Applies thousands of Sigma rules on streaming events
- No changes required to SIEM; works with existing ingestion architecture
- Designed for air-gapped and cloud-connected deployment constraints (data stays in your control)

## Features
- **Real-time Dashboard** - Live visualization of pipeline performance and metrics
- **Pipeline Management** - Create, configure, and monitor Flink-based ETL pipelines
- **Log Source Configuration** - Define parsing scripts and field mappings for event transformation
- **Rule Management** - Manage Sigma detection rules from cloud (SOC Prime Platform or SigmaHQ GitHub repository) and local repositories
- **Filter Configuration** - Create pre-filters to reduce false positives
- **Topic Synchronization** - Discover and manage Kafka topics
- **Hot-Reload Support** - Update rules, filters, and parsers without pipeline restart

## System Architecture

![System Architecture](static/image/system_architecture.png)

DetectFlow consists of the following projects, each in a separate repo:
- [DetectFlow Backend](https://github.com/socprime/detectflow-backend)
- [DetectFlow UI](https://github.com/socprime/detectflow-ui)
- [DetectFlow MatchNode](https://github.com/socprime/detectflow-matchnode)
- [DetectFlow Schema Parser](https://github.com/socprime/detectflow-parser)

Consult each project for more details.
## Requirements
### Technology stack:
- Apache Kafka 3.8+ (https://kafka.apache.org/)
- Kubernetes 1.28+ (https://kubernetes.io/)
- PostgreSQL 14+ (https://www.postgresql.org/)
- Apache Flink 1.13+ (can be deployed together with DetectFlow)

### Network and infrastructure:
- Internal access to deployed resources
- Network connectivity between deployed resources
- Kubernetes cluster with at least 3 nodes(workers) (min 8 vCPUs, 32 GiB RAM each). For a more detailed calculation
- Outbound internet access allowed:
    - to *.socprime.com for API access to the SOC Prime Platform for synchronizing detections (optional, API key needed)
    - to *.github.com for pulling open-source detections using GitHub Integration with public open source repositories, including SigmaHQ, Microsoft, Splunk, and Elastic (optional)

In terms of calculating required resources, the following components of DetectFlow should be considered, with individual requirements for each component summed up to estimate the total amount:
- **User Interface Node (Admin Panel)** is the control center to manage event matching processes. This component requires at least 2 CPUs and 4 GB RAM
- **Task Manager Node**, the default component to control the operation of pipelines (based on match nodes, the Apache Flink technology). This component requires at least 1 CPU and 4 GB RAM
- **Match Node**, a pipeline controlled via Admin Panel that matches detection rules to Apache Kafka topic events. The number of Match Nodes (pipelines) is custom, and you should estimate the required Kubernetes cluster resources based on the number of pipelines you're going to create. 

## Deployment
Deployment should be performed by a person with expertise in Kubernetes and admin access to the Kubernetes Cluster provided for this project.

Configuration files for K8S can be found in this [folder](https://github.com/socprime/detectflow-main/tree/main/detectflow_kubernetes_config_files) 

Deployment consists of two parts:

1. Apache Flink (according to its official deployment recommendations)
2. DetectFlow itself

Additionally, an Apache Kafka instance is required. If you don't have one, you can install it using the [Apache Kafka Quickstart](https://kafka.apache.org/quickstart/)

### Apache Flink Deployment
	
1. Install Cert Manager according to [https://cert-manager.io/docs/installation/](https://cert-manager.io/docs/installation/)

2. Get acquainted with the key concepts of Apache Flink Kubernetes Operator here: [https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-main/](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-main/)

3. Deploy Apache Flink Kubernetes Operator using Helm.
A Helm chart manages the installation of the operator. To install the operator (and also the helm chart) to a specific namespace, run the following command:

```
> helm repo add flink-operator-repo https://downloads.apache.org/flink/flink-kubernetes-operator-1.13.0/

Out: "flink-operator-repo" has been added to your repositories

> helm repo update

Out: Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "flink-operator-repo" chart repository

...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈

> helm install flink-kubernetes-operator flink-operator-repo/flink-kubernetes-operator --namespace flink-operator --create-namespace

Out: NAME: flink-kubernetes-operator
LAST DEPLOYED: Thu Feb  5 12:07:08 2026
NAMESPACE: flink-operator
STATUS: deployed
REVISION: 1
TEST SUITE: None

```

Source: [https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-main/docs/operations/helm/](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-main/docs/operations/helm/)

4. Deploy ConfigMap with static configuration for Flink Sigma Detector.

Run the following command to create a NameSpace:

```
> kubectl create ns flink

Out: namespace/flink created
```

Open the `flink-configmap.yaml` from this [folder](https://github.com/socprime/detectflow-main/tree/main/detectflow_kubernetes_config_files) and specify values for the following parameters:

- `namespace: flink`

For all the other parameters, keep the default values.

Run the following command:

```
> kubectl apply -f flink-configmap.yaml -n flink

Out: configmap/flink-config created
```

5. Deploy Secrets for Flink Sigma Detector.

Open the `flink-secret.yaml` from this [folder](https://github.com/socprime/detectflow-main/tree/main/detectflow_kubernetes_config_files) and specify the Base64 values for the following parameters (if needed, either leave by default):

- `KAFKA_BOOTSTRAP_SERVERS:`
- `KAFKA_AUTH_METHOD:`


Run the following command:

```
> kubectl apply -f flink-secret.yaml -n flink

Out: secret/flink-secret created
```

6. Deploy PersistentVolumeClaims for Flink State Storage.

Open the `flink-pvcs.yaml` from this [folder](https://github.com/socprime/detectflow-main/tree/main/detectflow_kubernetes_config_files) and specify values for the following parameters in each name: block:

- `namespace:`
- `storage:`
- `storageClassName:`


Run the following command:

```
> kubectl apply -f flink-pvcs.yaml -n flink

Out: 

persistentvolumeclaim/flink-checkpoints-pvc created
persistentvolumeclaim/flink-ha-pvc created
persistentvolumeclaim/flink-savepoints-pvc created
```

Optional for PVC, add a selector/label for nodes by running the following command:

```
> kubectl label nodes NAME spec-node-ns-pod-disk=pvc-security
```

7. Deploy RBAC Configuration for Flink ServiceAccount. It grants Flink permission to create and manage TaskManager pods.

Open the `flink-rbac.yaml` from this [folder](https://github.com/socprime/detectflow-main/tree/main/detectflow_kubernetes_config_files) and specify a value for the following parameter:

- `Namespace: flink`

Run the following command:

```
> kubectl apply -f flink-rbac.yaml -n flink

Out:

serviceaccount/flink created
role.rbac.authorization.k8s.io/flink created
rolebinding.rbac.authorization.k8s.io/flink-role-binding created
role.rbac.authorization.k8s.io/event-reader created
rolebinding.rbac.authorization.k8s.io/grant-event-reader created
```

### Docker Images Pulling 

Build Docker images locally according to instructions from README.md for each project:

- [DetectFlow Backend](https://github.com/socprime/detectflow-backend)
- [DetectFlow UI](https://github.com/socprime/detectflow-ui)
- [DetectFlow MatchNode](https://github.com/socprime/detectflow-matchnode)

1. Upload the images to your container registry that is accessible from your Kubernetes cluster.

2. Edit the following fields/variables in deployments and configmaps, so that the image address matches your container registry.

- `file: admin-panel-be-deployment.yaml` 
    - `field: spec/template/spec/containers/image`

- `file: admin-panel-ui-deployment.yaml`
    - `field: spec/template/spec/containers/image`

- `file: admin-panel-be-configmap.yaml`
    - `variable: FLINK_IMAGE`

Note: these files will be applied in later steps.

### DetectFlow Deployment
DetectFlow consists of two parts:
- DetectFlow backend
- DetectFlow UI
#### DetectFlow Backend Deployment
1. Deploy ConfigMap with static configuration for DetectFlow Backend.

Open `admin-panel-be-configmap.yaml` from this [folder](https://github.com/socprime/detectflow-main/tree/main/detectflow_kubernetes_config_files) and specify a value for the following parameter:

- `KAFKA_BOOTSTRAP_SERVERS:`

Set IP/Hostname and port of Kafka Server.

Run the following command:

```
> kubectl apply -f admin-panel-be-configmap.yaml -n flink

Out: configmap/admin-panel-be-config created
```

2. Deploy Secret with sensitive data configuration for DetectFlow Backend.

Open `admin-panel-be-secret.yaml` from this [folder](https://github.com/socprime/detectflow-main/tree/main/detectflow_kubernetes_config_files) and specify a value for the following parameter:

- `DATABASE_URL:`

`DATABASE_URL` has the following format: `postgresql+asyncpg://user:password@postgres_url:postgres_port/db_name`. It should be converted to Base64.

Run the following command:

```
> kubectl apply -f admin-panel-be-secret.yaml -n flink

Out: secret/admin-panel-be-secret created
```

3. Deploy PersistentVolumeClaims for DetectFlow Backend using `admin-panel-be-pvc.yaml` from this [folder](https://github.com/socprime/detectflow-main/tree/main/detectflow_kubernetes_config_files).
Run the following command:

```
> kubectl apply -f admin-panel-be-pvc.yaml -n flink

Out: persistentvolumeclaim/admin-panel-metrics-pvc created
```

4. Create a Deployment resource for DetectFlow Backend using `admin-panel-be-deployment.yaml` from this [folder](https://github.com/socprime/detectflow-main/tree/main/detectflow_kubernetes_config_files).

Run the following command:

```
> kubectl apply -f admin-panel-be-deployment.yaml -n flink

Out: deployment.apps/admin-panel-be created
```

5. Deploy Network Service resource for DetectFlow Backend using `admin-panel-be-service.yaml` from this [folder](https://github.com/socprime/detectflow-main/tree/main/detectflow_kubernetes_config_files).

Run the following command:

```
> kubectl apply -f admin-panel-be-service.yaml -n flink

Out: service/admin-panel-be created
```

6. Check that Admin-Panel-Backend-API is running and accessible following the next link:

`http://network_service_ip:8000`


#### DetectFlow UI Deployment


1. Deploy ConfigMap with a static configuration for DetectFlow UI.

Open `admin-panel-ui-configmap.yaml` from this [folder](https://github.com/socprime/detectflow-main/tree/main/detectflow_kubernetes_config_files) and specify a value for the following parameter:

Note: `NAME.NAMESPACE.CLUSTER_DOMAIN_NAME` (default: `svc.cluster.local`)

- `VITE_PREVIEW_ALLOWED_HOSTS:`

Set internal hostname of admin-panel-be, for example: `admin-panel-be.flink.svc.cluster.local`

Run the following command:

```
> kubectl apply -f admin-panel-ui-configmap.yaml -n flink

Out: configmap/admin-panel-ui-config created
```

2. Create a Deployment resource for DetectFlow UI using `admin-panel-ui-deployment.yaml` from this [folder](https://github.com/socprime/detectflow-main/tree/main/detectflow_kubernetes_config_files).

Run the following command:

```
> kubectl apply -f admin-panel-ui-deployment.yaml -n flink

Out: deployment.apps/admin-panel-ui created
```

3. Deploy Network Service resource for DetectFlow UI using `admin-panel-ui-service.yaml` from this [folder](https://github.com/socprime/detectflow-main/tree/main/detectflow_kubernetes_config_files).
Run the following command:

```
> kubectl apply -f admin-panel-ui-service.yaml -n flink

Out: service/admin-panel-ui created
```


4. Check that Admin-Panel-UI is running and accessible following the next link:
- `http://network-service-ip:4173`

Default: 
- Email: `admin@soc.local`
- Password: `admin`

## Getting Started
DetectFlow matches log events from Kafka topics with Sigma detection rules and tags each matched event with additional metadata related to the matched rules:
- ID
- Title
- Severity
- MITRE ATT&CK (sub-)technique IDs

To get started:
1. Ensure your Kafka instance has:
    - Topics with log source events you want to match with detection rules (source topics for DetectFlow)
    - Topics for tagged events that are consumed by a SIEM/EDR/Data Lake (destination topics for DetectFlow)
2. Create repositories with detection rules in Sigma format:
    - Local repositories
    - Cloud repositories that are synchronized with the SOC Prime Platform (optional, API key required)
    - Cloud repositories that are synchronized with GitHub public open source repositories, including SigmaHQ, Microsoft, Splunk, and Elastic (optional)
3. Create Log Source configurations to parse events and map their fields to the fields of detection rules.
4. Optionally, create Filters to filter out some events before matching.
5. Create pipelines using topics, repositories, log sources, and filters from the previous steps.
6. Monitor pipeline operation on the Dashboard.
7. Use tagged events for threat inference: pass them to a correlation ML model, such as MITRE TIE, or build aggregations directly in your SIEM based on the metadata added to the events.

### Accessing the Application

1. Navigate to the Admin Panel URL
2. Log in with your credentials set up at deployment
3. You'll be redirected to the Dashboard upon successful authentication

### Dashboard

The Dashboard provides a real-time overview of your detection pipeline infrastructure with live metrics and visualizations.

The center of the dashboard displays an interactive graph showing:

- **Source Topics** (upper left side) - Kafka topics that feed events into pipelines
- **Repositories** (lower left side) - Rule repositories with active rule counts
- **Pipelines** (center) - Active pipelines
- **Destination Topics** (right side) - Output topics where tagged events are written

**Interactions:**
- Click on any node to view detailed information
- Nodes are color-coded based on their status and activity
- Connections between nodes represent data flow

Click the pipelines icon to view detailed statistics for all pipelines:
- Pipeline name and status
- Source and destination topics
- Input/output throughput (events per second)
- Kafka consumer lag (pending records)
- Processing status (running, suspended, failed)

### Pipelines

Pipelines are the core processing units that consume events from Kafka, apply detection rules, and output tagged events.

#### Creating a Pipeline

1. Navigate to **Pipelines** → **New Pipeline**
2. Fill in the required fields:

Required Fields

- **Pipeline Name** - Unique identifier for the pipeline
- **Source Topic** - Kafka topic to consume events from (must exist)
- **Destination Topic** - Kafka topic to write tagged events to (must exist)
- **Repositories** - One or more rule repositories to apply (at least one required)
- **Log Source** - Pre-configured log source with parsing script and mapping. **Important:** Ensure your pipeline's source topic and repositories match the LogSource configuration for optimal results.

Optional Fields

- **Save Untagged Events** - Toggle to retain events that don't match any rules
- **Filters** - Pre-filters to reduce false positives (optional)
- **Custom Fields** - Static key-value pairs in YAML format to enrich all events

**Important Notes:**
- Changing **Source Topic** or **Destination Topic** as well as enabling or disabling the pipeline requires a pipeline restart (60-90s downtime)
- Changing **Repositories**, **Filters**, **Log Source**, or **Custom Fields** uses hot-reload (zero downtime)
- Pipeline must be **enabled** for changes to take effect

#### Enabling/Disabling Pipelines

- **Enable** - Creates FlinkDeployment in Kubernetes and starts processing
- **Disable** - Stops processing and removes FlinkDeployment

### Log Sources

Log Sources define how raw events are parsed and transformed before rule matching. Each log source contains:

1. **Parsing Script** - DSL query to extract and transform fields
2. **Field Mapping** - YAML mapping to align parsed fields with Sigma rule fields

#### Creating a Log Source

1. Navigate to **Settings** → **Log Sources** → **New Log Source**
2. Fill in the required fields:

- **Name** - Unique identifier for the log source
- **Source Topics** - Kafka topics to use for testing (at least one required)
- **Parsing Script** - DSL query using parser functions (see [Parser Functions Reference](https://github.com/socprime/detectflow-parser))
- **Mapping (YAML)** - Field mapping configuration
- **Repositories** - Repositories to use for mapping generation 

### Repositories and Rules

Repositories are collections of Sigma detection rules. There are two types:

1. **Cloud Repositories** - Synced from SOC Prime Platform or open-source GitHub repositories via API
2. **Local Repositories** - Manually created and managed

Details on the scope of supported Sigma rules can be found [here](https://github.com/socprime/detectflow-parser/blob/main/SUPPORTED_SIGMAS.md).

#### Cloud Repositories

Cloud repositories are automatically synced via API:
- From the SOC Prime Platform (API key required)
- From open-source GitHub repositories

#### Local Repositories

Local repositories allow you to create and manage rules independently.

Creating a Local Repository

1. Navigate to **Settings** → **Repositories**
2. Click **New Repository** (or **+** button)
3. Enter repository name
4. Click **Create Repository**

#### Managing Rules in Local Repositories
Rules must be in Sigma YAML format.
**Adding Rules:**

1. Select a local repository from the sidebar
2. Click **Add Rule** button
3. Enter rule name
4. Paste the Sigma rule YAML in the editor
5. Click **Create**

**Uploading Multiple Rules:**

1. Select a local repository
2. Click **Upload** button
3. Drag and drop YAML files or click to select. You can also upload a password-protected archive.
4. Rules are automatically parsed and uploaded
5. Review upload results

### Topics

Topics represent Kafka topics in your cluster. The system automatically discovers topics and shows their associations with pipelines.

Topics are used in:

1. **Pipeline Configuration** - As source and destination topics
2. **Log Source Testing** - As test topics for parsing scripts
3. **Dashboard Visualization** - Displayed in the pipeline graph

**Note:** Topics must exist in Kafka before they can be used in pipelines.

### Filters

Filters are pre-detection filters that reduce false positives by filtering events before rule matching.

#### Creating a Filter

1. Navigate to **Settings** → **Filters** → **New Filter**
2. Fill in the required fields:

Required Fields

- **Filter Name** - Unique identifier for the filter
- **Filter (YAML)** - Sigma detection condition in YAML format

Filters use Sigma detection syntax:

```yaml
detection:
  condition: selection
  selection:
    EventID: 4624
    LogonType: 3
    SubjectUserName|re: '.*\$'  # Exclude machine accounts
```

Filter Logic:

- Filters are applied **before** rule matching
- Events that match the filter condition are **excluded** from rule matching
- Multiple filters can be combined (AND logic)
- Filters use the same syntax as Sigma detection conditions

## License
This SOC Prime DetectFlow software is made available under a dual licensing model: (i) European Union Public License, version 1.2 and (ii) SOC Prime Commercial License. Depending on your intended use, you must choose one of the two licenses. See the [LICENSE](LICENSE) file for details.

