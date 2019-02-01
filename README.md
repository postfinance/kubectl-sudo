# kubectl sudo

## TL;DR
This plugin allows users to run kubernetes commands with the security privileges of another user:

```bash
$ kubectl get nodes
Error from server (Forbidden): nodes is forbidden: User "bofh" cannot list nodes at the cluster scope
```

```bash
$ kubectl sudo get nodes
NAME                     STATUS   ROLES    AGE   VERSION
kubelet1.example.com     Ready    <none>   96d   v1.11.2
kubelet2.example.com     Ready    <none>   96d   v1.11.2
```

With audit log containing the origin and the impersonated user and group, if configured correctly:
```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1beta1",
  "level": "Metadata",
  "stage": "ResponseComplete",
  "requestURI": "/api/v1/nodes?limit=500",
  "verb": "list",
  "user": {
    "username": "bofh",
    "groups": [
      "bofh_accounts",
      "system:authenticated"
    ]
  },
  "impersonatedUser": {
    "username": "bofh",
    " groups": [
      "system:masters"
    ]
  },
  "objectRef": {
    "resource": "nodes",
    "apiVersion": "v1"
  },
}
```

## Why
Kubernetes cluster administrators have great power. A mistake could
cause the cluster to become unhealthy or insecure and, as such, could impact
any or all tenants sharing the cluster. A simple `kubectl -f` with the wrong namespace
can end badly.

## How
This project does not really introduce a kubectl plugin, but a concept
of how to provide a sudo like system for kubernetes access.

To reduce the surface of unwanted or unexpected actions you can reduce the default priviledges
a cluster administrator has to the level of an unprivileged account and give them the ability to impersonate users and groups.
When cluster administrators need to do more priviledged actions, they can switch
the group to `system:masters` or another group or user according to the needed privilidge level.

In order to implement that concept, you need to declare a `ClusterRole` for
[impersonation](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#user-impersonation):

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: impersonator
rules:
- apiGroups: [""]
  verbs: ["impersonate"]
  resources: ["users", "groups", "serviceaccounts"]
```

Now you can assign this ClusterRole to the cluster administrators (e.g. group `bofh_accounts`):
```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-administrators
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: impersonator
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: bofh_accounts
```

Any user which has the group `bofh_accounts` can now do administration tasks with:

```
kubectl --as=$USER --as-group=system:masters delete node kubelet3.example.com
```

The provided kubectl plugin is just a wrapper for `kubectl` to shorten the `--as` and `--as-group` part.

## Installation
Place [kubectl-sudo](bash/kubectl-sudo) anywhere in your `$PATH` with execute permissions.
For further information, see the offical [plugin documentation](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/).

## Plugin Compatibility
Works on systems with `/bin/sh` and kubectl >= 1.12. `kubectl` must be inside `$PATH`.

## Configuration
This plugin can be configured using environment variables:
- `KUBECTL_SUDO_PROMPT=false` whether or not the plugin prompts the user before executing the kubectl command. Default value is `true`.
