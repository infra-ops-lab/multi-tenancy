# Kubernetes multi-tenancy: soft and hard isolation

How do you share one Kubernetes cluster between teams so that each team can work freely,
but cannot read, break, starve or escape into another team's space?

This lab answers that one layer at a time. It is about the **concepts**. Every step explains
what a control isolates, what it does **not** isolate, and then proves both with a small
experiment you run yourself: the attack before the guardrail, then the same attack after.

There is no install-everything script on purpose. Read the YAML, apply it, try to break it.

## Soft and hard isolation

**Soft isolation** means tenants share nodes, the kernel and the control plane, and are kept
apart by Kubernetes policy: namespaces, RBAC, quotas, network policy and pod security. It is
cheap and dense, and it is the right default for teams in one company who trust each other
not to attack the platform.

**Hard isolation** means tenants stop sharing something physical. In this lab that is the
**nodes**: a tenant's pods run only on its own nodes, and nobody else's pods can land there.
Further along the spectrum, tenants stop sharing the kernel, the control plane, or the
whole cluster.

```
 shared ◄──────────────────────────────────────────────────────────────► dedicated
 namespaces + policy   dedicated nodes    sandboxed runtime   virtual cluster   own cluster
 (steps 01-04)         (step 05)          (step 06)           (step 06)         (step 06)
 cheapest, densest                                                   strongest, most expensive
```

## The layers

Each step adds one layer. A tenant is only as isolated as its weakest layer.

```
  who are you?            identity (users, groups)            00-setup
  what may you do?        namespace + RBAC                    01-namespace-rbac
  how much may you use?   ResourceQuota + LimitRange          02-quota-limits
  whom may you talk to?   NetworkPolicy                       03-network-policy
  what may a pod do?      Pod Security Admission              04-pod-security
  where may it run?       dedicated nodes, enforced by policy 05-hard-isolation-nodes
  what is still shared?   kernel, control plane, cluster      06-beyond
```

## The lab world

| Tenant | Namespace | Group | User |
|---|---|---|---|
| Team A | `team-a` | `team-a-devs` | `alice` |
| Team B | `team-b` | `team-b-devs` | `bob` |

You are the **platform admin** (your normal kubectl context). Alice and Bob get their own
kubeconfig files in step 00, so you can switch between "admin" and "tenant" and see exactly
what each one is allowed to do.

## What you need

- An existing cluster and admin access with `kubectl`. Creating the cluster is not part of this lab.
- At least **3 worker nodes** for step 05 (two dedicated to team A, one shared). Steps 00-04 work with one.
- A network plugin that **enforces** NetworkPolicy (Calico, Cilium, or recent kind's default
  kindnet). Step 03 shows you how to tell.
- Kubernetes **1.36 or later** for step 05 as written (built-in MutatingAdmissionPolicy). Steps 00-04
  work on any supported version. Step 05 explains the alternative for older clusters.
- `openssl` for step 00.

Run every command from the root of this repository. Files for the lab users go into `users/`,
which git ignores.

## Steps

| Step | Concept | Isolation |
|---|---|---|
| [00-setup](00-setup/) | Kubernetes has no user objects: identity comes from certificates or tokens | – |
| [01-namespace-rbac](01-namespace-rbac/) | The namespace is the boundary, RBAC is the lock | soft |
| [02-quota-limits](02-quota-limits/) | One team must not starve the others | soft |
| [03-network-policy](03-network-policy/) | The pod network is flat until you say otherwise | soft |
| [04-pod-security](04-pod-security/) | One privileged pod defeats every layer above | soft |
| [05-hard-isolation-nodes](05-hard-isolation-nodes/) | Dedicated nodes, and why admission policy must enforce them | hard |
| [06-beyond](06-beyond/) | Observe each tenant; what is still shared, and when to move a tenant further | concept |

Each step's README has the same parts: **Concept**, **What it does not protect**, **Apply**,
**Try it**, and **Clean up**. The expected output shown in each step comes from a real run.

Tested on kind v0.24.0 with Kubernetes v1.36.1 (1 control-plane + 3 workers). See [TEST.md](TEST.md)
for what was checked and the results.
