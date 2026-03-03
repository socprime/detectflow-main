# Full Deployment
_For the general architecrute and other info about DetectFlow, see [README.md](README.md)._

Deployment should be performed by a person with expertise in Kubernetes and admin access to the Kubernetes Cluster provided for this project.

Configuration files for K8S can be found in this [folder](https://github.com/socprime/detectflow-main/tree/main/detectflow_kubernetes_config_files) 

 > **Note:** All configuration files in this repository contain placeholder data (mock secrets) and are intended for demonstration purposes only.

Deployment consists of two parts:

1. Apache Flink (according to its official deployment recommendations)
2. DetectFlow itself

Additionally, an Apache Kafka instance is required. If you don't have one, you can install it using the [Apache Kafka Quickstart](https://kafka.apache.org/quickstart/)

## Apache Flink Deployment
	
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

## Docker Images Pulling 

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

## DetectFlow Deployment
DetectFlow consists of two parts:
- DetectFlow backend
- DetectFlow UI
### DetectFlow Backend Deployment
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


### DetectFlow UI Deployment


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
