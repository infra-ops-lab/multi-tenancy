# 02 · Quota + limits: no team may starve the others

## Concept

Every pod declares two numbers per container:

- **requests**: what the scheduler **reserves** on a node. This decides where the pod fits.
- **limits**: the most it may **use** at runtime. Over the CPU limit it is throttled; over the
  memory limit it is killed (OOMKilled).

On a shared cluster, nothing stops one tenant from reserving every free CPU, so the next team's
pods stay `Pending`. Two objects, both owned by the platform team, prevent that:

- **ResourceQuota**: a budget for the **whole namespace** (total requests, total limits, pod
  count, number of load balancers...). The API server rejects any new pod that would go over it.
- **LimitRange**: rules for **each container**: defaults, plus minimum and maximum.

They need each other. Once a quota covers `requests.cpu`, a pod *without* requests is rejected.
The LimitRange fills in defaults, so developers are not blocked by a missing field. And its `max`
stops a single container from taking the whole budget at once.

```
 namespace team-a  ┌──────────── ResourceQuota: requests.cpu 2 ─────────────┐
                   │ [pod 0.5] [pod 0.5] [pod 0.5] [pod 0.5]  ✗ next pod     │
                   └─────── each container: LimitRange max 1 CPU ──────────┘
```

## What it does not protect

- A quota is a **budget, not a guarantee**. If the platform hands out more quota than the
  cluster has (overcommit), all teams can be within quota and still not fit.
- A quota does not evict anything. Pods running before the quota existed keep running.
- Limits do not isolate everything on a shared node: disk I/O, network bandwidth, kernel
  resources and noisy caches are still shared. That is one reason for dedicated nodes (step 05).
- The API server itself is shared. Kubernetes protects it with API Priority and Fairness, not
  with quotas.

## Apply

First see the problem. Alice runs a "forgotten" batch job: 10 replicas × 0.5 CPU ([hog.yaml](hog.yaml)).

```console
$ alice -n team-a apply -f 02-quota-limits/hog.yaml
deployment.apps/hog created
$ kubectl -n team-a rollout status deploy/hog
deployment "hog" successfully rolled out
```

All 10 run. Nothing stopped her from reserving 5 CPUs of shared capacity. Remove it and wait for
the pods to go:

```bash
alice -n team-a delete -f 02-quota-limits/hog.yaml
alice -n team-a wait --for=delete pod -l app=hog --timeout=120s
```

Now apply the guardrails as the admin ([quota.yaml](quota.yaml), [limitrange.yaml](limitrange.yaml)):

```bash
kubectl apply -f 02-quota-limits/quota.yaml
kubectl apply -f 02-quota-limits/limitrange.yaml
```

## Try it

Alice runs the same job again:

```console
$ alice -n team-a apply -f 02-quota-limits/hog.yaml
deployment.apps/hog created
$ alice -n team-a get pods -l app=hog --no-headers | wc -l
4
```

Only 4 pods: 4 × 0.5 = 2 CPUs, the whole budget. The Deployment does not show an error. The
reason is in the ReplicaSet's events, where few people look:

```console
$ alice -n team-a get events --field-selector reason=FailedCreate
LAST SEEN   TYPE      REASON         OBJECT                      MESSAGE
20s         Warning   FailedCreate   replicaset/hog-85d48cb456   Error creating: pods "hog-85d48cb456-x4crv" is forbidden: exceeded quota: team-a-quota, requested: requests.cpu=500m, used: requests.cpu=2, limited: requests.cpu=2
```

Alice can see her budget, but cannot change it (step 01):

```console
$ alice -n team-a describe quota team-a-quota
Resource                Used   Hard
--------                ----   ----
limits.cpu              2      4
limits.memory           256Mi  4Gi
pods                    5      10
requests.cpu            2      2
requests.memory         256Mi  2Gi
services.loadbalancers  0      0
```

`pods` is 5 because the `web` pod from step 01 counts too. It was created before the quota and
LimitRange, so it has no requests and counts as 0 CPU: a quota never touches what already runs.

Bob's team is untouched: `kubectl -n team-b rollout status deploy/web` still reports ready.

Free the budget, then see the LimitRange at work:

```bash
alice -n team-a delete -f 02-quota-limits/hog.yaml
alice -n team-a wait --for=delete pod -l app=hog --timeout=120s
```

A pod with no resources at all is admitted, with defaults filled in:

```console
$ alice -n team-a run nores --image=registry.k8s.io/pause:3.10
pod/nores created
$ alice -n team-a get pod nores -o jsonpath='{.spec.containers[0].resources.requests}'
{"cpu":"100m","memory":"128Mi"}
```

A container that asks for more than the per-container maximum ([too-big.yaml](too-big.yaml)) is rejected:

```console
$ alice -n team-a apply -f 02-quota-limits/too-big.yaml
Error from server (Forbidden): error when creating "02-quota-limits/too-big.yaml": pods "too-big" is forbidden: maximum cpu usage per Container is 1, but limit is 2
```

## Clean up

```bash
alice -n team-a delete pod nores
```

Keep the quota, the LimitRange and both `web` deployments. Step 03 uses them.
