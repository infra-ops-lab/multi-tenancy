# 04 · Pod Security: one privileged pod defeats every layer above

## Concept

All tenants on a node share **one Linux kernel**. A container is only a set of kernel
restrictions around a process. If a tenant may remove those restrictions (a `privileged`
container, host namespaces, `hostPath` mounts, extra capabilities), it is effectively root on the
node. From there it can read other tenants' files and logs, see their processes, and use the
node's credentials. RBAC, quotas and NetworkPolicy no longer matter.

**Pod Security Admission** is built into the API server. It checks every new pod against one of
three standard levels, chosen by a **namespace label**:

| Level | Meaning |
|---|---|
| `privileged` | no checks |
| `baseline` | blocks the known escalations: privileged, host namespaces, hostPath, extra capabilities |
| `restricted` | baseline, plus: run as non-root, drop all capabilities, no privilege escalation, a seccomp profile |

Tenant namespaces should use **`restricted`**. The label has three modes: `enforce` rejects the
pod, while `warn` and `audit` only report. Use `warn` to find out what would break before you enforce.

The label sits on the namespace, and step 01's role gives tenants no rights on namespaces, so a
tenant cannot switch it off.

## What it does not protect

- It checks **pod specs only**, at creation. It cannot see what the program inside does, and it
  does not protect against a kernel bug that lets a well-behaved container escape. That risk is
  why sandboxed runtimes exist (step 06).
- It has three fixed levels. For your own rules (allowed registries, required labels, which
  nodes a tenant may use) you need admission policies: the built-in ones (step 05) or a policy engine
  such as Kyverno or OPA Gatekeeper.
- Pods that are already running are not stopped when you turn it on.

## Apply

First see the problem. Alice creates a privileged pod with the node's disk mounted at `/host`
([privileged-pod.yaml](privileged-pod.yaml)):

```console
$ alice -n team-a apply -f 04-pod-security/privileged-pod.yaml
pod/node-reader created
$ alice -n team-a exec node-reader -- ls /host/var/log/pods
kube-system_kindnet-4krlm_f0965a40-2ce5-467c-af36-2f723a7e2224
kube-system_kube-proxy-r6wj6_8f59b2b4-c7d4-4280-be9f-c89b1ea91061
team-a_node-reader_8f6c3398-0b2d-4d69-8e3c-8eee84359ced
team-b_web-f68db8c4b-pmwt4_50bbc9ad-7d7f-4b9d-9675-67a585207f01
```

That last line is **team B's pod**, which happened to share the node. Its logs are one `cat`
away. Every isolation step so far is bypassed: RBAC never saw a request for team B's resources,
and no network traffic was involved.

Your output depends on which pods share the node with `node-reader`, but you will always see
system pods there.

Before enforcing, ask the API server what would break. A server-side dry run shows warnings for
existing pods:

```console
$ kubectl label --dry-run=server --overwrite ns team-a pod-security.kubernetes.io/enforce=restricted
Warning: existing pods in namespace "team-a" violate the new PodSecurity enforce level "restricted:latest"
Warning: node-reader: privileged, allowPrivilegeEscalation != false, unrestricted capabilities, restricted volume types, runAsNonRoot != true, seccompProfile
namespace/team-a labeled (server dry run)
```

Only `node-reader` is reported. `web` and `client` were written to the restricted level from
the start (see the `securityContext` in [web.yaml](../01-namespace-rbac/web.yaml)).

Remove the pod, then enforce on both tenants:

```bash
alice -n team-a delete -f 04-pod-security/privileged-pod.yaml
kubectl label --overwrite ns team-a pod-security.kubernetes.io/enforce=restricted
kubectl label --overwrite ns team-b pod-security.kubernetes.io/enforce=restricted
```

## Try it

The same pod is now rejected, with every reason listed:

```console
$ alice -n team-a apply -f 04-pod-security/privileged-pod.yaml
Error from server (Forbidden): error when creating "04-pod-security/privileged-pod.yaml": pods "node-reader" is forbidden: violates PodSecurity "restricted:latest": privileged (container "shell" must not set securityContext.privileged=true), allowPrivilegeEscalation != false (container "shell" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "shell" must set securityContext.capabilities.drop=["ALL"]), restricted volume types (volume "host" uses restricted volume type "hostPath"), runAsNonRoot != true (pod or container "shell" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "shell" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
```

A pod written for the restricted level is admitted ([compliant-pod.yaml](compliant-pod.yaml)),
and the compliant `web` keeps working when its pods are recreated:

```console
$ alice -n team-a apply -f 04-pod-security/compliant-pod.yaml
pod/compliant created
$ kubectl -n team-a rollout restart deploy/web
deployment.apps/web restarted
$ kubectl -n team-a rollout status deploy/web
deployment "web" successfully rolled out
```

Alice cannot switch the guardrail off:

```console
$ alice label --overwrite ns team-a pod-security.kubernetes.io/enforce=privileged
Error from server (Forbidden): namespaces "team-a" is forbidden: User "alice" cannot get resource "namespaces" in API group "" in the namespace "team-a"
```

## Clean up

```bash
alice -n team-a delete -f 04-pod-security/compliant-pod.yaml
```

That completes **soft isolation**: identity, API access, resources, network and pod privileges.
The tenants still share nodes. Step 05 changes that.
