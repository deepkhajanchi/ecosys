# Deploying TigerGraph on Red Hat OpenShift

This document provides detailed instructions for deploying a TigerGraph cluster on the Red Hat OpenShift platform.

- [Deploying TigerGraph on Red Hat OpenShift](#deploying-tigergraph-on-red-hat-openshift)
  - [Prerequisites](#prerequisites)
  - [Customize podPidsLimit and resource permission on OpenShift](#customize-podpidslimit-and-resource-permission-on-openshift)
    - [Change the podPidsLimit value of OpenShift](#change-the-podpidslimit-value-of-openshift)
    - [Acquire special permission](#acquire-special-permission)
  - [Deploying TigerGraph Operator](#deploying-tigergraph-operator)
    - [Install cert-manager for OpenShift](#install-cert-manager-for-openshift)
    - [Install kubectl-tg plugin](#install-kubectl-tg-plugin)
    - [Install CRDs independently (Optional)](#install-crds-independently-optional)
    - [Install TigerGraph Operator](#install-tigergraph-operator)
      - [Step 1: Define Environment Variable](#step-1-define-environment-variable)
      - [Step 2: Grant Permissions to Service Accounts](#step-2-grant-permissions-to-service-accounts)
      - [Step 3: Choose Operator Scope](#step-3-choose-operator-scope)
      - [Step 4: Install the Operator](#step-4-install-the-operator)
      - [Step 5: Custom Installation Options](#step-5-custom-installation-options)
      - [Step 6: Verify Operator Deployment](#step-6-verify-operator-deployment)
  - [Deploy a TigerGraph Cluster](#deploy-a-tigergraph-cluster)
    - [Create a service account binding to `SecurityContextConstraints` resource anyuid-extra](#create-a-service-account-binding-to-securitycontextconstraints-resource-anyuid-extra)
    - [Providing a Private SSH Key Pair for Enhanced Security](#providing-a-private-ssh-key-pair-for-enhanced-security)
    - [Specify the StorageClass name](#specify-the-storageclass-name)
    - [Specify the additional Storage for mounting multiple PVs(Optional)](#specify-the-additional-storage-for-mounting-multiple-pvsoptional)
    - [Customize configurations for the TigerGraph system (Optional)](#customize-configurations-for-the-tigergraph-system-optional)
    - [Create TG cluster with specific options](#create-tg-cluster-with-specific-options)
  - [Connect to a TigerGraph cluster](#connect-to-a-tigergraph-cluster)
    - [Connecting to a TigerGraph cluster Pod](#connecting-to-a-tigergraph-cluster-pod)
    - [Access TigerGraph Services](#access-tigergraph-services)
      - [Access TigerGraph Services from inside the Kubernetes cluster](#access-tigergraph-services-from-inside-the-kubernetes-cluster)
      - [Access TigerGraph Services from outside the Kubernetes cluster](#access-tigergraph-services-from-outside-the-kubernetes-cluster)
  - [Upgrade a TigerGraph cluster](#upgrade-a-tigergraph-cluster)
  - [Scale a TigerGraph cluster](#scale-a-tigergraph-cluster)
    - [Change the HA factor of the TigerGraph cluster](#change-the-ha-factor-of-the-tigergraph-cluster)
  - [Update the resources(CPU and Memory) of the TigerGraph cluster](#update-the-resourcescpu-and-memory-of-the-tigergraph-cluster)
  - [Update system configurations and license of the TigerGraph cluster](#update-system-configurations-and-license-of-the-tigergraph-cluster)
  - [Destroy the TigerGraph cluster and the Kubernetes Operator](#destroy-the-tigergraph-cluster-and-the-kubernetes-operator)
    - [Destroy the TigerGraph cluster](#destroy-the-tigergraph-cluster)
    - [Uninstall TigerGraph Operator](#uninstall-tigergraph-operator)
    - [Uninstall CRD](#uninstall-crd)
  - [See also](#see-also)

## Prerequisites

Before you begin, ensure you have the following prerequisites:

- [Helm](https://helm.sh/docs/intro/install/): version >= 3.7.0 TigerGraph Kubernetes Operator is packaged as a Helm chart, so you need Helm installed.

- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/): version >= 1.23 `kubectl-tg` plugin requires kubectl for running commands against Kubernetes clusters.

- [OpenShift CLI](https://docs.openshift.com/container-platform/4.8/cli_reference/openshift_cli/getting-started-cli.html): Install the OpenShift CLI to acquire permissions within OpenShift.

- Create an [OpenShift Kubernetes cluster](https://docs.openshift.com/container-platform/4.10/installing/index.html) with admin role permission. OpenShift Container Platform version requirements are 4 and above.

## Customize podPidsLimit and resource permission on OpenShift

Before deploying a TigerGraph cluster using the TigerGraph Operator, you must customize the `podPidsLimit` and request special resource permissions by creating a SecurityContextConstraints resource.

### Change the podPidsLimit value of OpenShift

In a production environment, TigerGraph clusters require setting the podPidsLimit to 1 million. If your OpenShift cluster hasn't set this, use the following commands:

```bash
kubectl label machineconfigpool worker custom-crio=high-pid-limit
kubectl label machineconfigpool worker custom-kubelet=small-pods
```

```bash
eval "cat << EOF
apiVersion: machineconfiguration.openshift.io/v1
kind: KubeletConfig
metadata:
  name: worker-kubeconfig-fix
spec:
  machineConfigPoolSelector:
    matchLabels:
      custom-kubelet: small-pods 
  kubeletConfig:
      podPidsLimit: 1024000
EOF" | kubectl apply -f -

eval "cat << EOF
apiVersion: machineconfiguration.openshift.io/v1
kind: ContainerRuntimeConfig
metadata:
 name: set-pids-limit
spec:
 machineConfigPoolSelector:
   matchLabels:
     custom-crio: high-pid-limit
 containerRuntimeConfig:
   pidsLimit: 1024000 
EOF" | kubectl apply -f -
```

- [Verify the value of podPidsLimit](https://access.redhat.com/solutions/5366631)

1. Monitor /sys/fs/cgroup/pids/pids.current when the application is running to verify java.lang.OutOfMemoryError: unable to create new native thread or similar errors happen when it hits 1024 (or 4096 in OCP 4.11+).
2. For OCP 4.10 and previous releases, check if the CRI-O pids_limit is being set on the node where the application container is running:

    ```bash
    $ crio config | grep pids_limit
    INFO[2022-01-31 12:14:27.407346183Z] Starting CRI-O, version: 1.21.4-4.rhaos4.8.git84fa55d.el8, git: () 
    INFO Using default capabilities: CAP_CHOWN, CAP_DAC_OVERRIDE, CAP_FSETID, CAP_FOWNER, CAP_SETGID, CAP_SETUID, CAP_SETPCAP, CAP_NET_BIND_SERVICE, CAP_KILL 
    pids_limit = 4096
    ```

3. Verify the kubelet podPidsLimit is being set in /etc/kubernetes/kubelet.conf and SupportPodPidsLimit (only in 4.10 and older) is set running the following command:

    ```bash
    $ oc debug node/[node_name] -- cat /host/etc/kubernetes/kubelet.conf | jq '.podPidsLimit, .featureGates'
    Starting pod/[node_name]-debug ...
    To use host binaries, run `chroot /host`
    Removing debug pod ...

    2048
    {
    "LegacyNodeRoleBehavior": false,
    "NodeDisruptionExclusion": true,
    "RotateKubeletServerCertificate": true,
    "SCTPSupport": true,
    "ServiceNodeExclusion": true,
    "SupportPodPidsLimit": true
    }
    ```

    In newer releases, it's not a json file, so use the following command instead:

    ```bash
    $ oc debug node/[node_name] -- cat /host/etc/kubernetes/kubelet.conf | grep podPidsLimit
    podPidsLimit: 4096
    ```

    If not configured, the default (1024) applies (or 4096 in OCP 4.11+).

4. Verify the labels for the ContainerRuntimeConfig (only in OCP 4.10 and previous release) and KubeletConfig were created and applied:

    ```bash
    $ oc get kubeletconfig,containerruntimeconfig
    NAME                                 AGE
    kubeletconfig/worker-kubeconfig-fix  9d

    NAME                                   AGE
    containerruntimeconfig/set-pids-limit  15d

    $ oc get mcp/worker -o json | jq '.metadata.labels'
    {
    "custom-crio": "high-pid-limit",
    "custom-kubelet": "small-pods",
    "machineconfiguration.openshift.io/mco-built-in": "",
    "pools.operator.machineconfiguration.openshift.io/worker": ""
    }

    $ oc get kubeletconfig/worker-kubeconfig-fix -o json | jq '.status.conditions[]'
    {
    "lastTransitionTime": "2022-02-10T04:46:17Z",
    "message": "Success",
    "status": "True",
    "type": "Success"
    }
    ```

### Acquire special permission

Starting from TigerGraph version 3.9.0 and Operator version 0.0.4, significant security enhancements have been introduced in the TigerGraph Kubernetes (k8s) Docker image. These enhancements are designed to reinforce the overall security posture of the TigerGraph deployment. Specifically, two notable changes have been made:

1. **SUDO Permission Removal**: The SUDO permission has been removed from the TigerGraph image. This change aligns with best practices in security by reducing unnecessary privileges within the containerized environment.

2. **Static Private SSH Key Removal**: Static private SSH key files have been eliminated from the TigerGraph image. This removal further enhances the security of your TigerGraph deployment by reducing potential vulnerabilities associated with static keys.

To ensure a seamless deployment of the TigerGraph cluster with these enhanced security measures, it is essential to perform an additional operation within the OpenShift environment. Failure to complete this step may lead to issues where the Operator successfully creates the StatefulSet, but the Pod for TigerGraph fails to generate due to insufficient permissions. Consequently, querying the status of the StatefulSet will yield the following warning events:

```bash
Normal   SuccessfulCreate  119s                 statefulset-controller  create Claim tg-data-test-cluster-0 Pod test-cluster-0 in StatefulSet test-cluster success
Warning  FailedCreate      37s (x15 over 119s)  statefulset-controller  create Pod test-cluster-0 in StatefulSet test-cluster failed error: 
pods "test-cluster-0" is forbidden: unable to validate against any security context constraint: [provider "anyuid": Forbidden: not usable by user or serviceaccount, 
provider restricted: .spec.securityContext.fsGroup: Invalid value: []int64{1000}: 1000 is not an allowed group, 
spec.containers[0].securityContext.runAsUser: Invalid value: 1000: must be in the ranges: [1000680000, 1000689999], 
provider "nonroot": Forbidden: not usable by user or serviceaccount, 
provider "hostmount-anyuid": Forbidden: not usable by user or serviceaccount, 
provider "machine-api-termination-handler": Forbidden: not usable by user or serviceaccount, 
provider "hostnetwork": Forbidden: not usable by user or serviceaccount, 
provider "hostaccess": Forbidden: not usable by user or serviceaccount, 
provider "node-exporter": Forbidden: not usable by user or serviceaccount, 
provider "privileged": Forbidden: not usable by user or serviceaccount]
```

- Create a SecurityContextConstraints
  
> [!IMPORTANT]
> Prior to **TigerGraph Operator version 1.6.0, the following** `SecurityContextConstraints` must be created before deploying the Operator and installing the TigerGraph cluster.

  Execute the following command:

  ```bash
  cat <<EOF | kubectl apply -f -
  kind: SecurityContextConstraints
  apiVersion: security.openshift.io/v1
  allowHostDirVolumePlugin: false
  allowHostIPC: false
  allowHostNetwork: false
  allowHostPID: false
  allowHostPorts: false
  allowPrivilegedContainer: true
  metadata:
    name: anyuid-extra
    annotations:
      kubernetes.io/description: anyuid-extra provides all features of the anyuid SCC
          but add SYS_CHROOT and AUDIT_WRITE capabilities.
  priority: 10
  readOnlyRootFilesystem: false
  requiredDropCapabilities:
  - MKNOD
  runAsUser:
    type: RunAsAny
  seLinuxContext:
    type: MustRunAs
  fsGroup:
    type: RunAsAny
  supplementalGroups:
    type: RunAsAny
  allowedCapabilities:
  - AUDIT_WRITE
  - SYS_ADMIN
  - SYS_PTRACE
  - SYSLOG
  groups:
  - system:cluster-admins
  volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - persistentVolumeClaim
  - projected
  - secret
  EOF
  ```

> [!IMPORTANT]
> Starting from **TigerGraph Operator version 1.6.0**, the following `SecurityContextConstraints` must be created before deploying the Operator and installing the TigerGraph cluster. Otherwise, you may encounter resource permission issues when deploying the Operator or installing the TigerGraph cluster on OpenShift.

  Execute the following command:

  ```bash
  cat <<EOF | kubectl apply -f -
  kind: SecurityContextConstraints
  apiVersion: security.openshift.io/v1
  allowHostDirVolumePlugin: false
  allowHostIPC: false
  allowHostNetwork: false
  allowHostPID: false
  allowHostPorts: false
  allowPrivilegeEscalation: true
  allowPrivilegedContainer: true
  allowedCapabilities:
  - AUDIT_WRITE
  defaultAddCapabilities: null
  metadata:
    name: anyuid-extra
    annotations:
      kubernetes.io/description: anyuid-extra provides all features of the anyuid SCC
          but add SYS_CHROOT and AUDIT_WRITE capabilities.
  priority: 10
  readOnlyRootFilesystem: false
  requiredDropCapabilities:
  - MKNOD
  runAsUser:
    type: RunAsAny
  seLinuxContext:
    type: MustRunAs
  fsGroup:
    type: RunAsAny
  supplementalGroups:
    type: RunAsAny
  seccompProfiles:
  - runtime/default
  groups:
  - system:cluster-admins
  volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - persistentVolumeClaim
  - projected
  - secret
  - csi
  - hostPath
  - ephemeral
  EOF
  ```

> [!WARNING]
> After creating the SecurityContextConstraints resource anyuid-extra, you still need to bind it to the service account used to deploy the TigerGraph Operator and TigerGraph cluster.

## Deploying TigerGraph Operator

To deploy the TigerGraph Operator, follow these steps:

### Install cert-manager for OpenShift

The TigerGraph Operator uses the Admission Webhooks feature and relies on [cert-manager](https://github.com/jetstack/cert-manager) for provisioning certificates for the webhook server.

Admission webhooks are HTTP callbacks that receive admission requests and do something with them. It is registered with Kubernetes and will be called by Kubernetes to validate or mutate a resource before being stored.

Follow these commands to install cert-manager:

> [!WARNING]
> Please check whether cert-manager has been installed before execute the following command.

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.17/cert-manager.yaml 
# Verify installation of cert-manager 
kubectl wait deployment -n cert-manager cert-manager --for condition=Available=True --timeout=90s
kubectl wait deployment -n cert-manager cert-manager-cainjector --for condition=Available=True --timeout=90s
kubectl wait deployment -n cert-manager cert-manager-webhook --for condition=Available=True --timeout=90s
```

### Install kubectl-tg plugin

kubectl-tg is a plugin for deploying and managing the Operator and TigerGraph clusters imperatively. Ensure you meet the following requirements before installing the kubectl-tg plugin:

- [helm](https://helm.sh/docs/intro/install/): version >= 3.7.0
- [jq](https://jqlang.github.io/jq/download/): version >= 1.6
- [yq](https://github.com/mikefarah/yq): version >= 4.18.1

> [!IMPORTANT]
> The kubectl-tg plugin is only verified on GNU/Linux systems.
>
> If you are using MacOS, you may encounter issues due to the differences between GNU and MacOS commands.
> Please refer to the [troubleshooting document](../06-troubleshoot/kubectl-tg-plugin.md) for more information.
>
> If you are using Windows, please run the commands in a WSL environment.
> Please refer to [Windows Subsystem for Linux Documentation](https://learn.microsoft.com/en-us/windows/wsl/) for more information.

Here's an example of installing the latest kubectl-tg, you can change the latest to your desired version, such as 1.6.0:

```bash
wget https://dl.tigergraph.com/k8s/latest/kubectl-tg -O kubectl-tg
sudo install kubectl-tg /usr/local/bin/
```

Display kubectl-tg version information:

```bash
kubectl tg version
```

Show help Information

```bash
kubectl tg help
```

### Install CRDs independently (Optional)

This step is optional. You can skip it if you have privileged permissions in your Kubernetes environment. The required component will be automatically installed during the Operator installation process.

CustomResourceDefinitions (CRDs) are non-namespaced entities accessible across all namespaces. Installing CRDs requires privileged permissions from the Kubernetes cluster. You may prefer to install CRDs independently from the Operator installation:

```bash
kubectl apply -f https://dl.tigergraph.com/k8s/latest/tg-operator-crd.yaml
```

### Install TigerGraph Operator

If you want to install the TigerGraph Operator using Helm, refer to [Deploy TigerGraph Operator with Helm](./deploy-operator-with-helm.md).

The example below demonstrates how to install the TigerGraph Operator using the `kubectl-tg` plugin.

#### Step 1: Define Environment Variable

To simplify the installation process, define your target namespace:

```bash
  export YOUR_NAMESPACE="tigergraph"
```

#### Step 2: Grant Permissions to Service Accounts

> [!IMPORTANT]
> The service accounts tigergraph-operator-controller-manager and tg-crd-upgrade-service-account are automatically created during the TigerGraph Operator installation.
>
> - tigergraph-operator-controller-manager is used for deploying the Operator.
> - tg-crd-upgrade-service-account is used for handling Operator upgrades.

Ensure both service accounts are bound to the previously created SecurityContextConstraints (e.g., anyuid-extra):

```bash
oc adm policy add-scc-to-user -n ${YOUR_NAMESPACE} -z tigergraph-operator-controller-manager anyuid-extra
oc adm policy add-scc-to-user -n ${YOUR_NAMESPACE} -z tg-crd-upgrade-service-account anyuid-extra
```

#### Step 3: Choose Operator Scope

TigerGraph Operator supports two deployment scopes:

- **Namespace-scoped Operator**: Watches and manages resources within a single namespace.

- **Cluster-scoped Operator**: Watches and manages resources across the entire cluster.

Choose a namespace-scoped Operator when:

- You want to isolate workloads per team, project, or tenant.

- Security or compliance requires limited access.

- You plan to run multiple Operator instances across different namespaces.

Choose a cluster-scoped Operator when:

- The Operator must manage resources across multiple namespaces.

- The application uses global policies or configurations.

- A single Operator instance should manage the whole cluster.

> [!IMPORTANT]
> Namespace-scoped Operators must use the same Operator version across all namespaces.

#### Step 4: Install the Operator

- **Install a namespace-scoped Operator**:

    ```bash
    kubectl tg init --cluster-scope false --namespace ${YOUR_NAMESPACE}
    ```

- **Install a cluster-scoped Operator** (default behavior if not specified):

    ```bash
    kubectl tg init --cluster-scope true --namespace ${YOUR_NAMESPACE}
    ```

#### Step 5: Custom Installation Options

You can customize the installation by specifying additional options such as Operator version, deployment size, resource limits, and concurrency settings. Example:

```bash
kubectl tg init --cluster-scope false --version ${OPERATOR_VERSION} --operator-size 3 --operator-watch-namespace ${YOUR_NAMESPACE} \
--operator-cpu 1000m  --operator-memory 1024Mi \
--max-tg-concurrent-reconciles 4 \
--max-backup-concurrent-reconciles 4 \
--max-backup-schedule-concurrent-reconciles 4 \
--max-restore-concurrent-reconciles 2 \
--namespace ${YOUR_NAMESPACE}
  ```

> [!IMPORTANT]
> For high availability, set the --operator-size to 2 or greater.
> [!NOTE]
> You can configure the maximum number of concurrent reconciliations during installation or update it later.
The default value of 2 is sufficient for most use cases, but a higher value is recommended if the Operator manages many TigerGraph clusters in a single Kubernetes cluster.

For a complete list of available options, run:

```bash
kubectl tg init --help
```

#### Step 6: Verify Operator Deployment

```bash
kubectl wait deployment tigergraph-operator-controller-manager --for condition=Available=True --timeout=120s -n ${YOUR_NAMESPACE}
```

## Deploy a TigerGraph Cluster

This section explains how to deploy a TigerGraph cluster on OpenShift using the kubectl-tg plugin and CR (Custom Resource) YAML manifest.

To simplify TigerGraph cluster deployment, define environment variables:

```bash
  export YOUR_NAMESPACE="tigergraph"
  export YOUR_CLUSTER_NAME="test-tg-cluster"
  export YOUR_SSH_KEY_SECRET_NAME="ssh-key-secret"
  export SERVICE_ACCOUNT_NAME="tg-service-account"
```

### Create a service account binding to `SecurityContextConstraints` resource anyuid-extra

- Create or use an exist service account

  Create a service account as follows:

  ```bash
  kubectl create serviceaccount ${SERVICE_ACCOUNT_NAME} -n ${YOUR_NAMESPACE}
  ```

- Add the service account name `${SERVICE_ACCOUNT_NAME}` to the above SecurityContextConstraints

  Ensure that the service account name `${SERVICE_ACCOUNT_NAME}` is added to the previously created SecurityContextConstraints:

  ```bash
  oc adm policy add-scc-to-user -n ${YOUR_NAMESPACE} -z ${SERVICE_ACCOUNT_NAME} anyuid-extra
  ```

With these steps completed, you can now use the service account name `${SERVICE_ACCOUNT_NAME}` to create the TigerGraph cluster on OpenShift.

> [!NOTE]
> If you are using the kubectl-tg plugin to create the TigerGraph cluster, you can specify the service account name using the `--service-account-name` option.
> [!WARNING]
> If you choose to install a cluster-scoped Operator, it is essential to create the aforementioned service account for each namespace in which you intend to deploy the TigerGraph cluster.

### Providing a Private SSH Key Pair for Enhanced Security

Starting from Operator version 0.0.4, users are required to provide their private SSH key pair for enhanced security before creating a cluster. Follow these steps:

- Step 1: create a private SSH key pair file:

  ```bash
  echo -e 'y\n' | ssh-keygen -b 4096 -t rsa -f $HOME/.ssh/tigergraph_rsa -q -N ''
  ```

- Step 2: Create a Secret Object

> [!IMPORTANT]
> The namespace of the Secret object must be the same as that of the TigerGraph cluster.

  Create a secret object based on the private SSH key file generated in step 1. Ensure that the key name of the secret for the private SSH key is `private-ssh-key`, and the key name for the public SSH key is `public-ssh-key`. Do not alter these key names:

  ```bash
  kubectl create secret generic ${YOUR_SSH_KEY_SECRET_NAME} --from-file=private-ssh-key=$HOME/.ssh/tigergraph_rsa --from-file=public-ssh-key=$HOME/.ssh/tigergraph_rsa.pub --namespace ${YOUR_NAMESPACE}
  ```

  For Operator versions 0.0.4 and above, when creating a cluster using the `kubectl tg create command`, you must set the `--private-key-secret` option to `${YOUR_SSH_KEY_SECRET_NAME}`.

These steps enhance the security of your cluster by utilizing your private SSH key pair.

### Specify the StorageClass name

Before creating the TigerGraph cluster with the Operator, specify the StorageClass, which defines the various "classes" of storage available. Use the following command to identify the name of the StorageClass:

You can determine the name of the StorageClass using the following command:

```bash
kubectl get storageclass

NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
standard (default)   kubernetes.io/gce-pd    Delete          WaitForFirstConsumer   true                   37m
standard-csi         pd.csi.storage.gke.io   Delete          WaitForFirstConsumer   true                   37m
```

With the StorageClass identified, you can proceed to create clusters using the create command. When specifying the --storage-class option, choose standard as its value.

This process ensures that the appropriate StorageClass is assigned to your TigerGraph cluster creation, optimizing storage provisioning and management.

### Specify the additional Storage for mounting multiple PVs(Optional)

You can specify multiple PVs for TigerGraph Pods by specifying the `--additional-storages` option. The value of this option is a YAML file configuration. For example:

> [!NOTE]
> Other parameters required to create a cluster are omitted here.

```bash
kubectl tg create --cluster-name ${YOUR_CLUSTER_NAME} --additional-storages additional-storage-tg-logs.yaml
```

Example additional storage YAML file:

```YAML
additionalStorages:
    - name: tg-kafka
    storageSize: 5Gi
    - name: tg-log
    storageSize: 5Gi
    - name: tg-sidecar
    storageClassName: efs-sc
    storageSize: 5Gi
    accessMode: ReadWriteMany
    volumeMode: Filesystem
    - name: tg-backup
    storageSize: 5Gi
    mountPath: /home/tigergraph/backup
    accessMode: ReadWriteOnce
    volumeMode: Filesystem
```

You can also specify the multiple PVs using CR configuration, For more information, see [Multiple persistent volumes mounting](../03-deploy/multiple-persistent-volumes-mounting.md)

### Customize configurations for the TigerGraph system (Optional)

You can customize the configurations for the TigerGraph system by specifying the `--tigergraph-config` option. The value of this option should be key-value pairs separated by commas. For example:

```bash
 --tigergraph-config "System.Backup.TimeoutSec=900,Controller.BasicConfig.LogConfig.LogFileMaxSizeMB=40"
```

 The key-value pairs are the same as the configurations that can be set by `gadmin config set` command. For more information, see [Configuration Parameters](https://docs.tigergraph.com/tigergraph-server/current/reference/configuration-parameters). All configurations will be applied to the TigerGraph system when the cluster is initializing.

### Create TG cluster with specific options

You can create a new TigerGraph cluster with specific options, such as size, high availability, version, license, and resource specifications.

> [!IMPORTANT]
> Choosing the right compute resources (CPU and memory) and storage size to host your TigerGraph system is crucial for achieving the right balance between cost and performance. We provide general guidelines for hardware selection based on simple hypothetical assumptions, but your actual hardware requirements will vary depending on your data size, workload, and performance needs.

- Hardware Recommendations

The sizing recommendations below apply to each TigerGraph node. If you have more than several hundred gigabytes of data, you should consider deploying a cluster of multiple nodes, to distribute your data.

| Deployment env | CPU  | Memory | Storage size |
|----------|----------|----------|----------|
| Personal Use | 4 cores | 8GB | ≥ 50GB |
| Development, UAT, or SIT System | 16 cores | 32GB | ≥ 300GB |
| Production System | 32 cores | 64GB | ≥ 500GB |

- Configuring HA settings

TigerGraph's HA (High Availability) service provides load balancing when all components are operational, and automatic failover in the event of a service disruption. For detailed information, please refer to the [official documents](https://docs.tigergraph.com/tigergraph-server/current/cluster-and-ha-management/ha-cluster).

The minimum value for the replication factor (HA) is 1, meaning high availability is not configured for the cluster. The partitioning factor is not explicitly set by the user; instead, TigerGraph determines it using the following formula:

`partitioning factor = number of pods / replication factor`

If the result is not an integer, some machines will remain unused. For example, in a 7-node cluster with a replication factor of 2, the system will configure 2-way HA with a partitioning factor of 3, leaving one machine unused.

In general, we recommend setting the replication factor (HA) to 2 and using a cluster size that is a power of 2 (e.g., 4, 8, 16)

- Export license key as an environment variable

  You must provide your license key when creating cluster. Contact TigerGraph support for help finding your license key.

  ```bash
  export LICENSE=<LICENSE_KEY>
  ```

- Create TigerGraph cluster with kubectl-tg plugin

  ```bash
  kubectl tg create --cluster-name ${YOUR_CLUSTER_NAME} --private-key-secret ${YOUR_SSH_KEY_SECRET_NAME} --size 4 --ha 2 --version 4.2.1 --license ${LICENSE} \
  --storage-class standard --storage-size 100G --cpu 6000m --memory 16Gi --namespace ${YOUR_NAMESPACE}
  ```

> [!NOTE]
> **Protect your license by using Secret**
>
> When you use option `--license` to set license, the license will be stored in the TigerGraph CR as plain text.
> To protect your license, you can create a K8s Secret to store the license and use the `--license-secret` option to set the license.
> For example:
>
> ```bash
> kubectl create secret generic ${YOUR_CLUSTER_NAME}-license --from-literal=license=${LICENSE} --namespace ${YOUR_NAMESPACE}
>
> ```
>
> Then, when creating the TigerGraph cluster, use the `--license-secret` option to set the license:
>
> ```bash
> kubectl tg create --cluster-name ${YOUR_CLUSTER_NAME} --private-key-secret ${YOUR_SSH_KEY_SECRET_NAME} --size 4 --ha 2 --version 4.2.1 \
> --license-secret ${YOUR_CLUSTER_NAME}-license --storage-class standard --storage-size 10G --cpu 2000m --memory 6Gi --namespace ${YOUR_NAMESPACE}
> ```
>
> [!IMPORTANT]
> You can use only one of the `--license` and `--license-secret` options to set the license when creating the TigerGraph cluster.
> If you set both options, the creation of CR will be rejected.

- Alternatively, create a TigerGraph cluster with a CR YAML manifest:

> [!NOTE]
> Please replace the TigerGraph docker image version (e.g., 4.2.1) with your desired version.
> If you want to use license secret instead of license, please replace the `license` field with `licenseSecretName` in the following CR YAML manifest.

  ```bash
  cat <<EOF | kubectl apply -f -
  apiVersion: graphdb.tigergraph.com/v1alpha1
  kind: TigerGraph
  metadata:
    name: ${YOUR_CLUSTER_NAME}
    namespace: ${YOUR_NAMESPACE}
  spec:
    image: docker.io/tigergraph/tigergraph-k8s:4.2.1
    imagePullPolicy: IfNotPresent
    ha: 2
    license: ${LICENSE}
    listener:
      type: LoadBalancer
    privateKeyName: ${YOUR_SSH_KEY_SECRET_NAME}
    replicas: 3
    resources:
      limits:
        cpu: "4"
        memory: 8Gi
      requests:
        cpu: "4"
        memory: 8Gi
    storage:
      type: persistent-claim
      volumeClaimTemplate:
        resources:
          requests:
            storage: 100G
        storageClassName: standard
  EOF
  ```

To ensure the successful deployment of the TigerGraph cluster, use the following command:

```bash
kubectl wait pods -l tigergraph.com/cluster-pod=${YOUR_CLUSTER_NAME} --for condition=Ready --timeout=15m --namespace ${YOUR_NAMESPACE}

kubectl wait --for=condition=complete --timeout=10m  job/${YOUR_CLUSTER_NAME}-init-job --namespace ${YOUR_NAMESPACE}
```

## Connect to a TigerGraph cluster

This section explains how to log into a TigerGraph cluster pod and access the `RESTPP`,`GUI`, and `Metrics` services.

### Connecting to a TigerGraph cluster Pod

To log into a single container within the TigerGraph cluster and execute commands like `gadmin status`, use the following command:

```bash
kubectl tg connect --cluster-name ${YOUR_CLUSTER_NAME} --namespace ${YOUR_NAMESPACE}
```

### Access TigerGraph Services

TigerGraph Operator provides two methods for accessing TigerGraph services. You can configure the spec.listener option to create an external service, allowing access to TigerGraph services from outside the Kubernetes cluster.

Additionally, during TigerGraph cluster creation, the operator automatically creates a headless service to facilitate communication between internal TigerGraph pods. This service can be leveraged to access TigerGraph services from within the Kubernetes cluster.

#### Access TigerGraph Services from inside the Kubernetes cluster

Query the headless service:

```bash
kubectl get svc ${YOUR_CLUSTER_NAME}-internal-service --namespace ${YOUR_NAMESPACE}
```

Output example:

```bash
NAME                            TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                        AGE
test-cluster-internal-service   ClusterIP   None         <none>        9000/TCP,10022/TCP,14240/TCP   6m23s
```

When using the headless service to access the TigerGraph service, you must log into one of the pods in the Kubernetes cluster. The following example demonstrates how to access it from one of the TigerGraph pods.

- Verify the API service

  ```bash
  curl ${YOUR_CLUSTER_NAME}-internal-service.${YOUR_NAMESPACE}.svc.cluster.local:14240/api/ping
  {"error":false,"message":"pong","results":null}
  ```

- Verify the RESTPP API service

  ```bash
  curl ${YOUR_CLUSTER_NAME}-internal-service.${YOUR_NAMESPACE}.svc.cluster.local:14240/restpp/echo
  {"error":false, "message":"Hello GSQL"}
  ```

- Verify the Metrics API service

  ```bash
  curl ${YOUR_CLUSTER_NAME}-internal-service.${YOUR_NAMESPACE}.svc.cluster.local:14240/informant/metrics/get/network -d '{"LatestNum":"1"}'
  {"NetworkMetrics":[{"EventMeta":{"Targets":[{"ServiceName":"IFM"}],"EventId":"702d134a162542ca8109de3dcbc51dea","TimestampNS":"1738829179030264667","Source":{"ServiceName":"EXE","Partition":1}},"HostID":"m1","CollectTimeStamps":"1738829179023049798","Network":{"IP":"10.20.128.210","TCPConnectionNum":382,"IncomingBytesNum":"49794423","OutgoingBytesNum":"48252486"}},{"EventMeta":{"Targets":[{"ServiceName":"IFM"}],"EventId":"dff3d22e6bfd4cf3a2f1d7fb252a4ed6","TimestampNS":"1738829180030864889","Source":{"ServiceName":"EXE","Partition":2}},"HostID":"m2","CollectTimeStamps":"1738829180024734149","Network":{"IP":"10.20.128.74","TCPConnectionNum":284,"IncomingBytesNum":"40035133","OutgoingBytesNum":"38002711"}},{"EventMeta":{"Targets":[{"ServiceName":"IFM"}],"EventId":"e00644e8c3d5414583605e5fccc41e67","TimestampNS":"1738829178029349154","Source":{"ServiceName":"EXE","Partition":3}},"HostID":"m3","CollectTimeStamps":"1738829178022643026","Network":{"IP":"10.20.128.10","TCPConnectionNum":279,"IncomingBytesNum":"36818317","OutgoingBytesNum":"25420121"}}]}
  ```

> [!NOTE]
> You can also access the TigerGraph service through a specific TigerGraph pod by prefixing the pod name. For example:
>
> ```bash
> curl ${YOUR_CLUSTER_NAME}-0.${YOUR_CLUSTER_NAME}-internal-service.${YOUR_NAMESPACE}.svc.cluster.local:14240/api/ping
>{"error":false,"message":"pong","results":null}
> ```

#### Access TigerGraph Services from outside the Kubernetes cluster

> [!IMPORTANT]
> Please ensure that the external service is configured before proceeding with the following steps. You can configure the external service using the `spec.listener` field when creating or updating the TigerGraph cluster. Please refer to [External access service](../08-reference/configure-tigergraph-cluster-cr-with-yaml-manifests.md#external-access-service) for more details.

Query the external service address:

  ```bash
  export EXTERNAL_SERVICE_ADDRESS=$(kubectl get svc/${YOUR_CLUSTER_NAME}-nginx-external-service --namespace ${YOUR_NAMESPACE} -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')
  ```

- Verify the API service

  ```bash
  curl http://${EXTERNAL_SERVICE_ADDRESS}:14240/api/ping

  {"error":false,"message":"pong","results":null}
  ```

To access the TigerGraph Suite, open it in your browser using the following URL: http://${EXTERNAL_SERVICE_ADDRESS}:14240, replacing `EXTERNAL_SERVICE_ADDRESS` with the actual service address.

- Verify the RESTPP API service

  ```bash
  curl http://${EXTERNAL_SERVICE_ADDRESS}:14240/restpp/echo

  {"error":false, "message":"Hello GSQL"}
  ```

- Verify the Metrics API service

  ```bash
  curl http://${EXTERNAL_SERVICE_ADDRESS}/informant/metrics/get/network -d '{"LatestNum":"1"}'

  {"NetworkMetrics":[{"EventMeta":{"Targets":[{"ServiceName":"IFM"}],"EventId":"1ebeaf2a380f4941b371efaaceb3467b","TimestampNS":"1703666521019463773","Source":{"ServiceName":"EXE","Partition":2}},"HostID":"m2","CollectTimeStamps":"1703666521008230613","Network":{"IP":"10.244.0.79","TCPConnectionNum":89,"IncomingBytesNum":"1654215","OutgoingBytesNum":"1466486"}},{"EventMeta":{"Targets":[{"ServiceName":"IFM"}],"EventId":"2c54ed5d6ba14e789db03fd9e023219c","TimestampNS":"1703666521020024563","Source":{"ServiceName":"EXE","Partition":3}},"HostID":"m3","CollectTimeStamps":"1703666521011409133","Network":{"IP":"10.244.0.78","TCPConnectionNum":90,"IncomingBytesNum":"1637413","OutgoingBytesNum":"1726712"}},{"EventMeta":{"Targets":[{"ServiceName":"IFM"}],"EventId":"c3478943ca134530bcd3aa439521c626","TimestampNS":"1703666521019483903","Source":{"ServiceName":"EXE","Partition":1}},"HostID":"m1","CollectTimeStamps":"1703666521009116924","Network":{"IP":"10.244.0.77","TCPConnectionNum":107,"IncomingBytesNum":"1298257","OutgoingBytesNum":"1197920"}}]}
  ```

## Upgrade a TigerGraph cluster

> [!WARNING]
> TigerGraph's exceptional performance comes with certain considerations regarding high availability during upgrading operations. Currently, TigerGraph does not provide dedicated high-availability upgrade support, and some downtime is involved.

Upgrading a TigerGraph cluster is supported from a lower version to a higher version.

> [!WARNING]
> For TigerGraph 3.9.3 and later versions, the use of passwords to log in to Pods is disabled, which enhances security. If you plan to upgrade your TigerGraph cluster to version 3.9.3, it is essential to first upgrade the Operator to version 0.0.9.
>
> [!WARNING]
> Operator 0.0.9 has disabled TG downgrades from a higher version (e.g., 3.9.3) to any lower version (e.g., 3.9.2). Therefore, the upgrade job will fail if you attempt to downgrade.

Assuming the current version of the cluster is 4.1.3, you can upgrade it to version 4.2.1 with the following command:

```bash
kubectl tg update --cluster-name ${YOUR_CLUSTER_NAME} --version 4.2.1  --namespace ${YOUR_NAMESPACE}
```

If you prefer using a CR YAML manifest, update the `spec.version` and `spec.image` field, and then apply it.

Ensure the successful upgrade with these commands:

```bash
kubectl rollout status --watch --timeout=900s statefulset/${YOUR_CLUSTER_NAME} --namespace ${YOUR_NAMESPACE}

kubectl wait --for=condition=complete --timeout=15m  job/${YOUR_CLUSTER_NAME}-upgrade-job --namespace ${YOUR_NAMESPACE}
```

## Scale a TigerGraph cluster

> [!WARNING]
> TigerGraph's exceptional performance comes with certain considerations regarding high availability during scaling operations. Currently, TigerGraph does not provide dedicated high-availability scale support, and some downtime is involved.

Before scaling the cluster, scale the corresponding node pool to provide sufficient resources for new instances. Use the following command to scale the TigerGraph cluster:

```bash
kubectl tg update --cluster-name ${YOUR_CLUSTER_NAME} --size 6 --ha 2  --namespace ${YOUR_NAMESPACE}
```

The above command scales the cluster to a size of 6 with a high availability factor of 2. If you prefer to use a CR (Custom Resource) YAML manifest for scaling, update the `spec.replicas` and `spec.ha` fields accordingly.

### Change the HA factor of the TigerGraph cluster

From Operator version 1.0.0, you can change the HA factor of the TigerGraph cluster without updating size by using the following command:

```bash
kubectl tg update --cluster-name ${YOUR_CLUSTER_NAME} --ha ${NEW_HA} --namespace ${YOUR_NAMESPACE}
```

## Update the resources(CPU and Memory) of the TigerGraph cluster

Modify the CPU and memory resources of your TigerGraph cluster using the following command:

```bash
kubectl tg update --cluster-name ${YOUR_CLUSTER_NAME} --cpu 8 --memory 16Gi  --cpu-limit 8 --memory-limit 16Gi  --namespace ${YOUR_NAMESPACE}
```

For CR YAML manifests, update the `spec.resources.requests` and `spec.resources.limits` fields and apply the changes.

## Update system configurations and license of the TigerGraph cluster

Use the following command to update the system configurations of the TigerGraph cluster:

```bash
kubectl tg update --cluster-name ${YOUR_CLUSTER_NAME} --tigergraph-config "System.Backup.TimeoutSec=900,Controller.BasicConfig.LogConfig.LogFileMaxSizeMB=40"  --namespace ${YOUR_NAMESPACE}
```

Use the following command to update the license of the TigerGraph cluster:

```bash
kubectl tg update --cluster-name ${YOUR_CLUSTER_NAME} --license ${LICENSE}  --namespace ${YOUR_NAMESPACE}
```

If you want to update both the system configurations and license of the TigerGraph cluster, please provide these two options together in one command(**recommanded**) instead of two separate commands:

```bash
kubectl tg update --cluster-name ${YOUR_CLUSTER_NAME} --tigergraph-config "System.Backup.TimeoutSec=900,Controller.BasicConfig.LogConfig.LogFileMaxSizeMB=40" --license ${LICENSE}  --namespace ${YOUR_NAMESPACE}
```

## Destroy the TigerGraph cluster and the Kubernetes Operator

### Destroy the TigerGraph cluster

To delete a TigerGraph cluster, use the following command. Note that this command does not remove Persistent Volume Claims (PVCs) and Persistent Volumes (PVs) associated with the cluster. To delete these components, manually delete the PVCs.

- Delete the TigerGraph cluster and retain the PVs:

  ```bash
  kubectl tg delete --cluster-name ${YOUR_CLUSTER_NAME} -n ${YOUR_NAMESPACE}
  ```

- Delete the PVCs related to the specified cluster:

  ```bash
  # Identify the PVCS to delete by specific labels of PVC.
  kubectl get pvc -l tigergraph.com/cluster-name=${YOUR_CLUSTER_NAME} -n ${YOUR_NAMESPACE}

  # Delete the PVCS related to the specified cluster.
  kubectl delete pvc -l tigergraph.com/cluster-name=${YOUR_CLUSTER_NAME} -n ${YOUR_NAMESPACE}
  ```

### Uninstall TigerGraph Operator

Uninstall the TigerGraph Kubernetes Operator within a specified namespace:

```bash
kubectl tg uninstall -n ${YOUR_NAMESPACE}
```

### Uninstall CRD

Uninstall CRDs if needed:

> [!NOTE]
> Replace the variable `${OPERATOR_VERSION}` to the Operator version you installed.

```bash
kubectl delete -f https://dl.tigergraph.com/k8s/${OPERATOR_VERSION}/tg-operator-crd.yaml
```

## See also

If you are interested in the details of deploying a TigerGraph cluster using the CR (Custom Resource) YAML manifest, refer to the following document:

- [Configuring TigerGraph Clusters on K8s using TigerGraph CR](../08-reference/configure-tigergraph-cluster-cr-with-yaml-manifests.md)
