# 06 · Beyond: what is still shared, and when to move further

## What you have built

| Layer | Control | Step | Kind |
|---|---|---|---|
| Identity | users and groups from certificates (OIDC in production) | 00 | – |
| API access | namespace + RBAC bound to groups | 01 | soft |
| Resources | ResourceQuota + LimitRange | 02 | soft |
| Network | default-deny NetworkPolicy | 03 | soft |
| Pod privileges | Pod Security `restricted` | 04 | soft |
| Nodes | dedicated nodes: taint + label, enforced by admission policies | 05 | hard |

This is the common shape of an internal developer platform's "namespace as a service". In
practice the same objects are created by automation (GitOps, an operator, or a tool such as
Capsule) whenever a team is onboarded, rather than by hand.

## Observe each tenant

Every guardrail in this lab says "no" somewhere a tenant rarely looks. A rejected pod never
exists, so `kubectl get pods` shows nothing wrong. The reason sits in a ReplicaSet event (step
02). Platform teams also have to answer "who has access to this tenant, and how much quota is
left?", which means reading RoleBindings and quotas across many namespaces.

[tenant-insights](https://github.com/infra-ops-lab/k8s-operator-lab/tree/main/tenant-insights)
is a small, read-only operator that keeps one report per tenant: access, quota left, and the
workloads being blocked right now, with the reason and a fix hint. It groups namespaces into
tenants by a label, so it can read this lab's `tenant` label directly.

### Install

It needs `docker`, `kubectl` and `helm`. The script builds the image and, on kind, loads it into
the nodes:

```bash
git clone https://github.com/infra-ops-lab/k8s-operator-lab
cd k8s-operator-lab/tenant-insights
TENANT_LABEL=tenant ./scripts/install.sh
export PATH="$PWD/plugin:$PATH"          # the kubectl tenants plugin
cd -                                     # back to this lab
```

The operator runs in its own namespace, which has no `tenant` label, so this lab's policies do not
apply to it. On the tainted cluster from step 05 it lands on the shared node.

### Read the reports

```console
$ kubectl tenants
TENANT  NAMESPACES  OWNERS  CPU USED/HARD  CPU LEFT  MEM LEFT  BLOCKED  STATUS
team-a  1           1       100m / 2       1.9       1.9Gi     0        OK
team-b  1           1       100m / 2       1.9       1.9Gi     0        OK

$ kubectl tenants describe team-a
Tenant:      team-a
Namespaces:  team-a
Status:      OK

Access (namespace RoleBindings only):
  team-a           ClusterRole/tenant-developer ← Group team-a-devs

Quota:
  NAMESPACE  RESOURCE                USED   HARD   LEFT   USED%
  team-a     limits.cpu              500m   4      3.5    █░░░░░░░░░  12.5%
  team-a     limits.memory           256Mi  4.0Gi  3.8Gi  █░░░░░░░░░   6.2%
  team-a     pods                    1      10     9      █░░░░░░░░░  10.0%
  team-a     requests.cpu            100m   2      1.9    ░░░░░░░░░░   5.0%
  team-a     requests.memory         128Mi  2.0Gi  1.9Gi  █░░░░░░░░░   6.2%
  team-a     services.loadbalancers  0      0      0      ░░░░░░░░░░   0.0%
  Total CPU 100m / 2 (left 1.9) · memory 128Mi / 2.0Gi (left 1.9Gi)

Blocked workloads:
  none
```

Steps 01 and 02 in one view: the group binding, and the quota with what is left.

### Watch a tenant get blocked

Alice applies the `hog` Deployment from step 02 again:

```bash
alice -n team-a apply -f 02-quota-limits/hog.yaml
```

The Deployment is created, but `alice -n team-a get deploy hog` shows `0/10` and no reason.
Within 30 seconds the report has one:

```console
$ kubectl tenants
TENANT  NAMESPACES  OWNERS  CPU USED/HARD  CPU LEFT  MEM LEFT  BLOCKED  STATUS
team-a  1           1       100m / 2       1.9       1.9Gi     1        Blocked
team-b  1           1       100m / 2       1.9       1.9Gi     0        OK

$ kubectl tenants describe team-a
...
Blocked workloads:
  team-a Deployment/hog  (PodSecurity, 35 attempts)
    Reason: pods "hog-85d48cb456-6jd2x" is forbidden: violates PodSecurity "restricted:latest": allowPrivilegeEscalation != false (container "hog" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "hog" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "hog" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "hog" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
    Fix:    the pod breaks the namespace's Pod Security level: fix its securityContext
```

`hog.yaml` was written for step 02, before Pod Security was enforced, and it does not meet the
`restricted` level. This is what happens to real teams when a platform turns on Pod Security:
old manifests stop starting, without an error anywhere they look. If you skipped step 04, the
report shows the quota from step 02 instead (`exceeded quota: team-a-quota`, with "ask for a
bigger quota, or scale something down").

The cluster view compares quota handed out with real capacity:

```console
$ kubectl tenants cluster
RESOURCE  CAPACITY  QUOTA GIVEN  OVERCOMMIT  REQUESTED  FREE TO REQUEST
cpu       32        4            0.12×       1.5        30.5
memory    94.8Gi    4.0Gi        0.04×       824Mi      94.0Gi
```

Remove `hog` and the report returns to `OK` within 30 seconds:

```bash
alice -n team-a delete -f 02-quota-limits/hog.yaml
```

The report does not show **security posture**. A tenant with a privileged pod and no Pod Security
label still reports `OK`. Keep that in mind: an observability tool only sees what it was built to
look for.

## What is still shared

Even with dedicated nodes, every tenant in this cluster shares:

- **The kernel**, on any shared node. A kernel bug can turn one container escape into access to
  every pod on that node.
- **The control plane**: API server, etcd, scheduler, cluster DNS. One tenant's load or one bad
  controller affects all.
- **Cluster-scoped objects**: CRDs, ClusterRoles, StorageClasses, webhooks, the Kubernetes version.
  Two tenants cannot have different versions of the same CRD.
- **The blast radius of a cluster upgrade or an outage**.

Each of these has a next step along the spectrum.

## Further along the spectrum

| Option | What it stops sharing | Cost and trade-offs |
|---|---|---|
| **Sandboxed runtime** (gVisor, Kata Containers) via a `RuntimeClass` | The host kernel: each pod gets a user-space kernel or a lightweight VM. | Slower syscalls and I/O, some features unsupported, node setup required. Good for running untrusted code. |
| **Virtual cluster** (for example vCluster) | The API server, etcd and cluster-scoped objects: each tenant gets its own control plane running as pods. | Tenants can install their own CRDs and be "cluster admin" of their own cluster. Workloads still run on the host cluster's nodes, so steps 02-05 still apply underneath. |
| **Dedicated cluster** per tenant | Everything. | Strongest isolation and simplest compliance story. Highest cost and the most clusters to upgrade, so it needs fleet tooling (Cluster API, GitOps). |

## When to move a tenant further

Move along the spectrum when a tenant's **requirements** change, not by default:

- **Untrusted code** (customer workloads, CI for external contributors, plugins) → sandboxed runtime.
- **Needs its own CRDs, operators or Kubernetes version** → virtual cluster.
- **Compliance or contract demands** physical separation (PCI, a regulated customer) → dedicated
  nodes at least, often a dedicated cluster.
- **Its load hurts others**, even within quota → dedicated nodes.
- **Its failure must not take others down**, or the reverse → dedicated cluster.

The hard part is not choosing one model for every tenant. It is making it cheap to move a tenant
along the spectrum as it grows: the same manifests, the same pipeline, and a placement decision
that lives in configuration.

## Clean up

Remove everything this lab created. Use the same node names as in step 05.

```bash
# tenant-insights, if you installed it: run this in the k8s-operator-lab/tenant-insights folder you cloned.
# DELETE_CRDS=true also removes the report types.
#   DELETE_CRDS=true ./scripts/uninstall.sh

kubectl delete -f 05-hard-isolation-nodes/team-a-placement.yaml -f 05-hard-isolation-nodes/tenant-node-boundaries.yaml --ignore-not-found

kubectl taint node $A1 $A2 tenant-
kubectl label node $A1 $A2 tenant-

kubectl delete namespace team-a team-b --ignore-not-found
kubectl delete clusterrole tenant-developer --ignore-not-found
kubectl delete csr alice bob --ignore-not-found
rm -rf users
unalias alice bob
```

Deleting the namespaces removes everything inside them: deployments, quotas, limit ranges, network
policies and role bindings. The users' certificates stay valid until they expire (24 hours), but
with their RoleBindings gone they can no longer do anything.
