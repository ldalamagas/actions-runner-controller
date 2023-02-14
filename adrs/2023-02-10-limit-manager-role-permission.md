# ADR 0007: Limit Permissions for Service Accounts in Actions-Runner-Controller
**Date**: 2023-02-10

**Status**: Pending

## Context

- `actions-runner-controller` is a Kubernetes CRD (with controller) built using https://github.com/kubernetes-sigs/controller-runtime

- [controller-runtime](https://github.com/kubernetes-sigs/controller-runtime) has a default cache based k8s API client.Reader to make query k8s API server more efficiency. 

- The cache-based API client requires cluster scope `list` and `watch` permission for any resource the controller may query.

- This documentation only scopes to the AutoscalingRunnerSet CRD and its controller.

## Service accounts and their role binding in actions-runner-controller

There are 3 service accounts involved for a working `AutoscalingRunnerSet` based `actions-runner-controller`

1. Service account for each Ephemeral runner Pod

This should have the lowest privilege (not any `RoleBinding` nor `ClusterRoleBinding`) by default, in the case of `containerMode=kubernetes`, it will get certain write permission with `RoleBinding` to limit the permission to a single namespace.

> References:
> - ./charts/auto-scaling-runner-set/templates/no_permission_serviceaccount.yaml
> - ./charts/auto-scaling-runner-set/templates/kube_mode_role.yaml
> - ./charts/auto-scaling-runner-set/templates/kube_mode_role_binding.yaml
> - ./charts/auto-scaling-runner-set/templates/kube_mode_serviceaccount.yaml

2. Service account for AutoScalingListener Pod

This has a `RoleBinding` to a single namespace with a `Role` that has permission to `PATCH` `EphemeralRunnerSet` and `EphemeralRunner`.

3. Service account for the controller manager

Since the CRD controller is a singleton installed in the cluster that manages the CRD across multiple namespaces, the service account of the controller manager pod has a `ClusterRoleBinding` to a `ClusterRole` with broader permissions.

The current `ClusterRole` has the following permissions:

- Get/List/Create/Delete/Update/Patch/Watch on `AutoScalingRunnerSets` (with `Status` and `Finalizer` sub-resource)
- Get/List/Create/Delete/Update/Patch/Watch on `AutoScalingListeners` (with `Status` and `Finalizer` sub-resource)
- Get/List/Create/Delete/Update/Patch/Watch on `EphemeralRunnerSets` (with `Status` and `Finalizer` sub-resource)
- Get/List/Create/Delete/Update/Patch/Watch on `EphemeralRunners` (with `Status` and `Finalizer` sub-resource)

- Get/List/Create/Delete/Update/Patch/Watch on `Pods` (with `Status` sub-resource)
- **Get/List/Create/Delete/Update/Patch/Watch on `Secrets`**
- Get/List/Create/Delete/Update/Patch/Watch on `Roles`
- Get/List/Create/Delete/Update/Patch/Watch on `RoleBindings`
- Get/List/Create/Delete/Update/Patch/Watch on `ServiceAccounts`

> Full list can be found at: https://github.com/actions/actions-runner-controller/blob/facae69e0b189d3b5dd659f36df8a829516d2896/charts/actions-runner-controller-2/templates/manager_role.yaml

## Limit cluster role permission on Secrets

The cluster scope `List` `Secrets` permission might be a blocker for adopting `actions-runner-controller` for certain customers as they may have certain restriction in their cluster that simply doesn't allow any service account to have cluster scope `List Secrets` permission. 

To help these customers and improve security for `actions-runner-controller` in general, we will try to limit the `ClusterRole` permission of the controller manager's service account down to the following:

- Get/List/Create/Delete/Update/Patch/Watch on `AutoScalingRunnerSets` (with `Status` and `Finalizer` sub-resource)
- Get/List/Create/Delete/Update/Patch/Watch on `AutoScalingListeners` (with `Status` and `Finalizer` sub-resource)
- Get/List/Create/Delete/Update/Patch/Watch on `EphemeralRunnerSets` (with `Status` and `Finalizer` sub-resource)
- Get/List/Create/Delete/Update/Patch/Watch on `EphemeralRunners` (with `Status` and `Finalizer` sub-resource)

- List/Watch on `Pods`
- List/Watch on `Roles`
- List/Watch on `RoleBindings`
- List/Watch on `ServiceAccounts`

> We will change the default cache-based client to bypass cache on reading `Secrets`, so we can eliminate the need for `List` and `Watch` `Secrets` permission in cluster scope.

Introduce a new `ClusterRole` for the namespace that each `AutoScalingRunnerSet` deployed with the following permission

- Get/Create/Delete/Update/Patch on `Secrets`
- Get/Create/Delete/Update/Patch on `Pods`
- Get/Create/Delete/Update/Patch on `Roles`
- Get/Create/Delete/Update/Patch on `RoleBindings`
- Get/Create/Delete/Update/Patch on `ServiceAccounts`

The `RoleBinding` for the new `ClusterRole` will happen during `helm install demo oci://ghcr.io/actions/actions-runner-controller-charts/auto-scaling-runner-set` to grant the controller's service account required permissions to operate in the namespace the `AutoScalingRunnerSet` deployed.