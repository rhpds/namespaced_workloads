# ocp4_workload_tenant_namespace

Creates one or more OpenShift namespaces for a tenant user, applies resource controls, and grants RBAC access.

## What it does

- Creates namespaces named `{username}-{suffix}`, or a single namespace named after the user when no suffixes are defined
- Applies a `LimitRange` to every namespace to set container resource defaults
- Creates a `ClusterResourceQuota` (default) selecting namespaces by the user's `openshift.io/requester` annotation, giving the user a shared resource pool
- Creates one `AdminNetworkPolicy` per tenant when `ocp4_workload_tenant_namespace_admin_network_policy` is true. It is off by default. Priority 30, so that tenant's namespaces can reach each other and cannot reach other tenants
- Grants the user the configured RBAC role in each namespace

## Quota and LimitRange

The full default quota and LimitRange are in `defaults/main.yml`. A catalog item sets `ocp4_workload_tenant_namespace_default_quota` or `ocp4_workload_tenant_namespace_default_limit_range` to replace individual keys. Omitted keys stay at the builtin value. Setting `limits.memory: 4000Gi` does not drop the secrets quota. The LimitRange overlay merges one level down, so `default.memory` can change without dropping `default.cpu`.

Set `ocp4_workload_tenant_namespace_admin_network_policy: true` to create the per-tenant AdminNetworkPolicy. Leave it false for an internal tenant that does not need one. Destroy removes it.

The policy only does two things: this tenant's namespaces may talk to each other, and namespaces with `openshift.io/requester` for anyone else may not. There is no per-tenant allow list. Traffic out of the cluster, and any platform destination every tenant needs, belongs on the cluster security policy.

## Usage

Set `ocp4_workload_tenant_namespace_username` and optionally define the namespaces to create:

```yaml
ocp4_workload_tenant_namespace_username: "user-{{ guid }}"

ocp4_workload_tenant_namespace_suffixes:
- suffix: myapp
- suffix: mydb
```

Leave `suffixes` empty to create a single namespace named after the user.

See [`defaults/main.yml`](defaults/main.yml) for all variables and their descriptions, including quota sizing and how to switch between ClusterResourceQuota and per-namespace ResourceQuota.

## Cluster quota scope

The role sets `openshift.io/requester` to `ocp4_workload_tenant_namespace_username` on every namespace it creates, including a single namespace with no suffixes, regardless of quota mode. Existing labels and other metadata are preserved; the requester annotation takes precedence over custom metadata. OpenShift also sets it on projects requested by that user through the ProjectRequest API (for example, `oc new-project`), so those projects share the same quota without needing a tenant label.

Additional namespaces created directly by privileged automation, including GitOps, must explicitly carry the same requester annotation to join the quota. Quota membership does not apply this role's LimitRange or RBAC to those namespaces. Cleanup still deletes only the namespaces declared through this role, not other namespaces matching the quota.

With `ocp4_workload_tenant_namespace_use_cluster_quota: false`, a per-namespace ResourceQuota is applied instead of a ClusterResourceQuota — the requester annotation is still set.

## Provision UUID label

Every namespace the role creates also gets a `demo.redhat.com/tenant-uuid` label set to `ocp4_workload_tenant_namespace_uuid` (defaults to `guid`), giving operators a pod-to-namespace-to-tenant trace path for cleanup.
