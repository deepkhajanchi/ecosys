# TigerGraph Operator 1.6.0 Release notes

## Overview

**TigerGraph Operator 1.6.0** is now available, designed to work seamlessly with **TigerGraph version 4.2.1**.

> [!IMPORTANT]
> Operator versions prior to 1.5.0 relied on the Docker image `gcr.io/kubebuilder/kube-rbac-proxy` to secure the metrics endpoint and prevent exposure of sensitive data. However, this image will no longer be available after May 2025. Starting from version 1.5.0, the Operator now uses the Docker image `quay.io/brancz/kube-rbac-proxy` instead. To ensure successful deployment without image pull errors, you must upgrade the Operator to version 1.5.0 or later.

> [!NOTE]
> Starting from Operator versions 1.6.0, the Operator removed the dependency on the Docker image quay.io/brancz/kube-rbac-proxy by leveraging the WithAuthenticationAndAuthorization feature provided by Controller-Runtime to secure the metrics endpoint.

This release introduces significant new features, enhancements, and bug fixes, including:

- Support configuring the TigerGraph license using a secret name in the TigerGraph CR.
- Adding additional storage after cluster creation.
- Support backup and restore with Azure Blob and Google Cloud Storage.
- Support compliance with GKE Policy (Gatekeeper) requirements through configuration of the TigerGraph Cluster custom resource.
- Separate the necessary cluster and namespace roles to enhance security for the namespace-scoped TG Operator installation.
- Output the correct operator version when executing kubectl tg version.
- Executing gadmin backup on another pod when pod-0 is down.
- Upgrading k8s API version to 1.31, Golang version to 1.23, and controller-runtime 0.19.
- Removed the dependency on the Docker image quay.io/brancz/kube-rbac-proxy by leveraging the WithAuthenticationAndAuthorization feature provided by Controller-Runtime to secure the metrics endpoint.

- Fix the stderr output issue in pre-stop and post-start hooks.

For further details, see the sections below.

> [!IMPORTANT]
> TigerGraph Operator has had a breaking change since version 1.0.0. If you are still using a version older than 1.0.0, it is strongly recommended that you upgrade to version 1.6.0. Versions older than 1.0.0 have been deprecated.

### kubectl plugin installation

To install the kubectl plugin for TigerGraph Operator 1.6.0, execute the following command:

```bash
curl https://dl.tigergraph.com/k8s/1.6.0/kubectl-tg  -o kubectl-tg
sudo install kubectl-tg /usr/local/bin/
```

### TigerGraph Operator upgrading

#### Upgrading from TigerGraph Operator 1.0.0+ to 1.6.0

There are no breaking changes in the Custom Resource Definitions (CRDs) for version 1.6.0 compared to versions 1.0.0 and above. If you are running Operator 1.0.0 or later, upgrade using the following command:

> [!NOTE]
> There is currently no support for upgrading or deleting CRDs when upgrading or uninstalling the TigerGraph Operator due to the risk of unintentional data loss. It is necessary to upgrade TigerGraph CRDs manually for the operator version prior to 1.3.0. However, starting from Operator version 1.3.0, we use [Helm chart’s pre-upgrade hook](https://helm.sh/docs/topics/charts_hooks/) to upgrade the CRDs automatically. You can ignore the first step if you upgrade the operator to version 1.3.0 or above.

> [!IMPORTANT]
> Please ensure that you have installed the `kubectl-tg` version 1.6.0 before upgrading TigerGraph Operator to version 1.6.0.

Ensure you have installed the correct version of kubectl-tg:

```bash
kubectl tg version

Version: 1.6.0
Default version of TigerGraph cluster: 4.2.1
```

Upgrade TigerGraph Operator using kubectl-tg plugin:

```bash
kubectl tg upgrade --namespace ${YOUR_NAMESPACE_OF_OPERATOR} --operator-version 1.6.0
```

#### Upgrading from TigerGraph Operator Versions Prior to 1.0.0

This TigerGraph Operator version upgrade introduces breaking changes if you are upgrading from TigerGraph Operator versions prior to 1.0.0. You need to upgrade the TigerGraph Operator, CRD, and the TigerGraph cluster following specific steps.

Refer to the documentation [How to upgrade TigerGraph Kubernetes Operator](../04-manage/operator-upgrade.md) for details.

## New features

- Support configuring the TigerGraph license using a secret name in the TigerGraph CR.
- Adding additional storage after cluster creation.
- Support backup and restore with Azure Blob and Google Cloud Storage.
- Support compliance with GKE Policy (Gatekeeper) requirements through configuration of the TigerGraph Cluster custom resource.

## Improvements

- Separate the necessary cluster and namespace roles to enhance security for the namespace-scoped TG Operator installation.
- Executing gadmin backup on another pod when pod-0 is down.
- Upgrading the Kubernetes API version to 1.31, the Go (Golang) version to 1.23, and controller-runtime to 0.19.
- Removed the dependency on the Docker image quay.io/brancz/kube-rbac-proxy by leveraging the WithAuthenticationAndAuthorization feature provided by Controller-Runtime to secure the metrics endpoint.
- Support license updates while the cluster is in the ResumeRoll state.
- Ability to skip the readiness check for a specific TigerGraph service.
- Skip the license status check in the readiness probe for TG versions that support keeping all services online after the license expires.

## Bug Fixes

- Output the correct operator version when executing kubectl tg version.
- Fix the stderr output issue in pre-stop and post-start hooks.
- Enable simultaneous upgrade and scaling operations on the TigerGraph cluster using the kubectl tg plugin.
- Check the executor status before stopping the local service in the pre-stop hook of the TigerGraph container.
- Upgrade jq to the latest version (1.8.0) in the TigerGraph Kubernetes image to fix vulnerabilities.
- Allow simultaneous upgrade and scaling operations on the TG cluster via kubectl tg plugin.
- Avoid getting stuck in the ResumeRoll state when upgrading the operator.
