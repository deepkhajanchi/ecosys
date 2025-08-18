# Required Permissions for TigerGraph Operator

Since TigerGraph Operator 1.6.0, we seperate the permissions required by Operator into two categories:

1. **Required Cluster-Scope Permissions**: These permissions are required for the Operator to function properly. They are related to the cluster-scope resources, e.g. CRDs, PVCs, etc.
2. **Required Namespace-Scope Permissions**: These permissions are required for the Operator to function properly. They are related to the namespace-scope resources, e.g. Pods, Services, etc.

To limit the permissions of the Operator, we separate the permissions into two roles: `tigergraph-operator-manager-clusterrole` and `tigergraph-operator-manager-role`.
When TigerGraph Operator is deployed as a namespace-scoped operator, the `tigergraph-operator-manager-role`(Role) is created in the namespace where the Operator is deployed.
The `tigergraph-operator-manager-clusterrole`(ClusterRole) is created in the cluster scope and is used to manage cluster-scoped resources.
When TigerGraph Operator is deployed as a cluster-scoped operator, the `tigergraph-operator-manager-role` will also be created as a ClusterRole in the cluster scope.
Here is a table to describe how we limit the permissions of the Operator:

| Scope of Operator | Scope of  `tigergraph-operator-manager-role` | Scope of  `tigergraph-operator-manager-clusterrole` |
|-------------------|----------------------------------------------|--------------------------------------------------|
| Namespace-scoped  | Role(only in the namespace of the Operator) | ClusterRole                                      |
| Cluster-scoped    | ClusterRole                                  | ClusterRole                                      |

To know all permissions required by TigerGraph Operator, please refer to the [YAML file](../10-samples/deploy/role.yaml) that defines the ClusterRole and Role.