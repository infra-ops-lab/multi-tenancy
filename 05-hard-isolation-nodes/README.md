# 05 · Hard isolation: dedicated nodes, enforced by policy

## Concept

After step 04, tenants are well separated, but they still **share nodes**. On a shared node they
share the kernel, CPU caches, disk and network bandwidth, and a node failure or a kernel exploit
hits everyone on it. For tenants with stricter needs (a regulated workload, a noisy batch
platform, a customer that pays for dedicated capacity) the next step is **dedicated nodes**.

Two node features do this, and you need both, because each does only half the job:

| | What it does | What it does not do |
|---|---|---|
| **Taint** on the node (`tenant=team-a:NoExecute`) | Repels every pod that does not tolerate it. `NoExecute` also evicts pods already running there. | Does not attract anything. Team A's pods without a toleration are repelled too. |
| **Label** on the node + **nodeSelector** on the pod | Attracts pods that ask for it. | Does not keep anyone else out. |

And both are **requests written in the pod spec by the tenant**. Any tenant can write
"tolerate team A's taint and select team A's nodes". RBAC and Pod Security do not look at
those fields. So the boundary only holds if a **policy engine** writes and checks them:

```
                      ┌───────────── Kyverno (admission) ─────────────┐
 pod from team-a ───► │ mutate: add nodeSelector + toleration          │ ───► team-a nodes only
 pod from team-b ───► │ validate: no team-a toleration, no wildcard,   │ ───► denied, if it tries
                      │           no spec.nodeName                     │
                      └────────────────────────────────────────────────┘
   nodes:   [ worker: team-a ]  [ worker2: team-a ]  [ worker3: shared ]
```

- [team-a-placement.yaml](team-a-placement.yaml) **attracts**: every pod created in `team-a` gets
  the nodeSelector and the toleration added on its way in. Developers write normal manifests.
- [tenant-node-boundaries.yaml](tenant-node-boundaries.yaml) **keeps out**, for every namespace
  with a `tenant` label:
  - a pod may only tolerate its **own** tenant's taint;
  - no wildcard toleration (`operator: Exists` with no key tolerates **every** taint);
  - no `spec.nodeName`, which skips the scheduler and therefore the taint check.

## What it does not protect

- Tenants still share the **control plane**: the API server, etcd and the cluster DNS. A tenant
  that floods the API affects everyone (API Priority and Fairness helps).
- Tenants still share the **cluster network**. Step 03's policies are still what keeps them apart.
- Cluster-wide objects are still shared: CRDs, ClusterRoles, StorageClasses, admission webhooks.
  A tenant that needs its own CRDs or versions needs its own control plane (step 06).
- The policy engine is now part of your security boundary. If it is down and its webhook fails
  open, the boundary is gone. Kyverno's validating webhook fails closed by default (`svc-fail`).
- It costs more: dedicated nodes are paid for even when the tenant is idle.

## Apply

This step needs at least 3 worker nodes: two for team A and one shared. Pick yours:

```bash
kubectl get nodes
A1=<first team-a node>  A2=<second team-a node>     # e.g. mt-lab-worker mt-lab-worker2
```

With fewer workers, dedicate one node to team A and keep one shared. The ideas are the same.

Label and taint team A's nodes:

```bash
kubectl label node $A1 $A2 tenant=team-a
kubectl taint node $A1 $A2 tenant=team-a:NoExecute
```

`NoExecute` evicts every pod on those nodes that does not tolerate the taint, **including team A's
own**. Pods owned by a Deployment come back on the shared node. Bare pods, like the `client` pods
from step 03, are deleted for good if they were there.

### Before the policies

Team A restarts its `web` with 4 replicas:

```console
$ alice -n team-a scale deploy/web --replicas=4
$ alice -n team-a rollout restart deploy/web
$ kubectl -n team-a get pods -l app=web -o wide          # NODE column
... all 4 on mt-lab-worker3
```

The taint **repels**, but it does not attract. Team A's pods all went to the shared node.

Now Bob writes a pod that asks for team A's nodes ([sneaky-pod.yaml](sneaky-pod.yaml)):

```console
$ bob -n team-b apply -f 05-hard-isolation-nodes/sneaky-pod.yaml
pod/sneaky created
$ bob -n team-b get pod sneaky -o wide                   # NODE column
... mt-lab-worker
```

Team B is running on team A's "dedicated" node. Nothing stopped it.

```bash
bob -n team-b delete -f 05-hard-isolation-nodes/sneaky-pod.yaml
```

### Install Kyverno and the policies

Kyverno is a policy engine that runs as an admission webhook. This lab was tested with chart
`3.9.2` (Kyverno `v1.19.2`):

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm install kyverno kyverno/kyverno --version 3.9.2 -n kyverno --create-namespace --wait
```

Read both policies, then apply them:

```bash
kubectl apply -f 05-hard-isolation-nodes/team-a-placement.yaml
kubectl apply -f 05-hard-isolation-nodes/tenant-node-boundaries.yaml
kubectl wait --for=condition=Ready clusterpolicy --all
```

## Try it

Team A restarts `web`. The new pods are changed on admission and land on team A's nodes:

```console
$ alice -n team-a rollout restart deploy/web
$ kubectl -n team-a get pods -l app=web -o wide          # NODE column
... mt-lab-worker, mt-lab-worker2 only
$ alice -n team-a get pods -l app=web -o jsonpath='{.items[0].spec.nodeSelector}'
{"tenant":"team-a"}
```

Alice's Deployment does not mention nodes at all. The platform decided where it runs.

Bob tries the same sneaky pod again:

```console
$ bob -n team-b apply -f 05-hard-isolation-nodes/sneaky-pod.yaml
Error from server: error when creating "05-hard-isolation-nodes/sneaky-pod.yaml": admission webhook "validate.kyverno.svc-fail" denied the request:

resource Pod/team-b/sneaky was blocked due to the following policies

tenant-node-boundaries:
  no-foreign-tenant-tolerations: Pods may only tolerate their own tenant's taint (tenant=team-b).
```

He tries a wildcard toleration ([wildcard-pod.yaml](wildcard-pod.yaml)):

```console
$ bob -n team-b apply -f 05-hard-isolation-nodes/wildcard-pod.yaml
...
tenant-node-boundaries:
  no-wildcard-tolerations: A toleration without a key tolerates every taint, including other tenants' nodes.
```

He tries to skip the scheduler and name the node directly ([nodename-pod.yaml](nodename-pod.yaml)):

```console
$ sed "s/REPLACE-WITH-A-TEAM-A-NODE/$A1/" 05-hard-isolation-nodes/nodename-pod.yaml | bob -n team-b apply -f -
...
tenant-node-boundaries:
  no-node-name: spec.nodeName bypasses the scheduler and taints. Let the scheduler place pods.
```

Team B's own workloads are unaffected and stay on the shared node:

```console
$ kubectl -n team-b rollout restart deploy/web
$ kubectl -n team-b get pods -l app=web -o wide          # NODE column
... mt-lab-worker3
```

## Clean up

```bash
alice -n team-a scale deploy/web --replicas=1
```

Keep everything for now. [06-beyond](../06-beyond/) explains what is still shared, and has the
full clean-up.
