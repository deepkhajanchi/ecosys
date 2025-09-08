# Operator 1.0.0 Release notes

## Overview

**Operator 1.0.0** is now available, designed to work seamlessly with **TigerGraph version 3.10.0**.

> [!WARNING]
> Operator 1.0.0 introduces a breaking change to TigerGraph CRD, TigerGraphBackup, and TigerGraphRestore CRD. It is crucial to uninstall the previous Operator version, remove all old CRDs, and install the new version. Remember to retain the PVC of the existing cluster to recreate it after the upgrade.

In this release, **Operator 1.0.0** brings significant enhancements:

- Customizing and updating TigerGraph configurations via TigerGraph CR or kubectl-tg plugin
- Pausing and resuming TigerGraph cluster
- Customizing SecurityContext of TigerGraph Container
- Customizing labels and annotations of TigerGraph Pod
- Introducing lifecycle hooks for TigerGraph CR with the addition of postInitAction
- Independent modification of replication factor
- Mounting multiple PVC and PV for pods of TigerGraph
- Customizing volume mounts for TigerGraph container
- Customizing ingressClassName of ingress external service

Operator 1.0.0 has refactored TigerGraph CRD, TigerGraphBackup, and TigerGraphRestore CRD, simplifying cluster provision and improving usability. These changes have two significant impacts:

1. If you have deployed TigerGraph using the Operator and wish to upgrade to 1.0.0, carefully follow the upgrade documentation. [How to upgrade TigerGraph Kubernetes Operator](../04-manage/operator-upgrade.md)
2. For accessing TigerGraph services outside Kubernetes, only one external service is now used to access RESTPP, GUI, and Metrics services. Update related configurations if your client application depends on these services.

### kubectl plugin installation

To install the kubectl plugin for Operator 1.0.0, execute the following command:

```bash
curl https://dl.tigergraph.com/k8s/1.0.0/kubectl-tg  -o kubectl-tg
sudo install kubectl-tg /usr/local/bin/
```

### Operator upgrading

This new operator version upgrade brings breaking changes. Refer to the documentation [How to upgrade TigerGraph Kubernetes Operator](../04-manage/operator-upgrade.md) for details.

- Delete the existing TG cluster and retain the PVCs:

```bash
# You should take note of the cluster size, HA and so on before you delete it, you'll use it when you recreate the cluster
# You can export the yaml resource file of TG cluster for the later restoring
kubectl tg export --cluster-name ${YOUR_CLUSTER_NAME} -n ${NAMESPACE_OF_CLUSTER}
kubectl tg delete --cluster-name ${YOUR_CLUSTER_NAME} -n ${NAMESPACE_OF_CLUSTER}
```

- Uninstall the old version of the Operator:

```bash
kubectl tg uninstall -n ${NAMESPACE_OF_OPERATOR}
```

- Delete old versions of TG CRDs:

```bash
kubectl delete crd tigergraphs.graphdb.tigergraph.com
kubectl delete crd tigergraphbackups.graphdb.tigergraph.com
kubectl delete crd tigergraphbackupschedules.graphdb.tigergraph.com
kubectl delete crd tigergraphrestores.graphdb.tigergraph.com
```

- Reinstall the new version of the Operator:

```bash
kubectl tg init -n ${NAMESPACE_OF_OPERATOR}
```

- Recreate the TigerGraph cluster if necessary:

Extract parameters from the backup YAML resource file generated in step 1, or modify the YAML resource file and apply it directly.

```bash
# You can get the following parameters from the backup yaml resoure file in step 1
kubectl tg create --cluster-name ${YOUR_CLUSTER_NAME} -n ${NAMESPACE_OF_CLUSTER} \
--size ${CLUSTER_size} --ha ${CLUSTER_HA} --private-key-secret ${YOUR_PRIVATE_KEY_SECRET} \
--version ${TG_VERSION} --storage-class ${YOUR_STORAGE_CLASS} --storage-size ${YOUR_STORAGE_SIZE} --cpu 3000m --memory 6Gi
```

## New features

- Support customizing and updating TigerGraph configurations via TigerGraph CR or kubectl-tg plugin.

- Support pausing and resuming TigerGraph cluster.

- Support customizing SecurityContext of TigerGraph.

- Support customizing labels and annotations of TigerGraph Pod.

- Support Lifecycle hooks for TigerGraph CR: postInitAction.

- Support mounting multiple PVC and PV for pods of TG.

- Support customized volume mounts for TG container.

- Support configuring additional storage and custom volume mounts of the TG container in kubectl-tg plugin.

- Support customizing ingressClassName of ingress external service.

- Support customizing custom volume with a new option --custom-volume in kubectl-tg plugin.

- Supports creating and updating clusters without external services via kubectl-tg.

- Supports independent modification of replication factor via TigerGraph CR.

## Improvements

- Improve state transitions of TigerGraph CR.

- Remove the redundant configuration of Tigergraph CRD and refactor the status output.

- Removing dynamic pod labels and RESTPP external service.

- Remove field spec.InitTGConfig and place its remaining fields into subfields of spec and Status.

- Remove the InitJob field to improve usability.

- Refactor and improve the status output of TG CR.

- Improve TigerGraphBackup and TigerGraphRestore CRD.

- Removing the pods of scale down automatically after executing shrinking successfully.

- Invokes switch_version.sh to switch new version of TG to decouple the DB upgrade business logic.

- Improve the failover process of expansion/shrinking for the new error code of ETCD.

- Use `gadmin start all --local` to start local services.

## Bugfixes

- Fix the issue of readiness check when the license is expired.

- Add retry logic when resetting services and apply new configuration during expansion/shrinking.
