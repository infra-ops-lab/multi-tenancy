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
