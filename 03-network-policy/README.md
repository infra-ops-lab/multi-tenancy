# 03 · NetworkPolicy: the pod network is flat until you say otherwise

## Concept

By default, **every pod can connect to every other pod in the cluster**, across all namespaces.
Namespaces and RBAC (step 01) say nothing about network traffic. Alice cannot *list* Bob's pods,
but her pods can still *call* them.

A **NetworkPolicy** selects pods and lists what traffic is allowed to and from them. Three rules
explain almost all of it:

1. A pod that **no policy selects** allows everything (the default above).
2. Once **any policy selects a pod** for a direction (Ingress or Egress), only traffic that some
   policy allows in that direction gets through.
3. Policies are **additive allow-lists**. There is no "deny" rule. You deny by selecting and
   not allowing.

So the tenant pattern is: **deny everything, then allow what the tenant needs.**

```
           team-a                          team-b
   ┌──────────────────────┐        ┌──────────────────────┐
   │ client ──► web   ✓   │ ──✗──► │ web                  │
   │    │                 │ ◄──✗── │ client ──► web   ✓   │
   └────┼─────────────────┘        └──────────────────────┘
        └──► kube-dns :53 ✓  (kube-system)
```

The policies here ([default-deny](default-deny.yaml), [allow-dns](allow-dns.yaml),
[allow-same-namespace](allow-same-namespace.yaml)) belong to the platform team. Step 01's role gives
tenants read-only access to them, so a tenant cannot delete its own isolation.

## What it does not protect

- **NetworkPolicy is only enforced if your network plugin enforces it.** On a plugin that does
  not, the policies are accepted and silently ignored. The "Try it" below is how you find out.
- It works on IPs and ports (layer 3/4). It does not know about HTTP paths, and it does not
  encrypt anything. For identity-based or layer-7 rules, add a service mesh or a plugin like Cilium.
- `hostNetwork` pods use the node's network and bypass these rules. Step 04 stops tenants from
  creating them.
- Traffic leaving the cluster is now blocked too. Real tenants usually need some egress
  (a database, an API). Add explicit allow rules for those destinations.

## Apply

Both teams start a small client pod, which we use to make requests:

```bash
alice -n team-a run client --image=curlimages/curl:8.11.1 --command -- sleep 36000
bob   -n team-b run client --image=curlimages/curl:8.11.1 --command -- sleep 36000
```

A helper to print only the HTTP status code (`000` means no connection):

```bash
http() { kubectl --kubeconfig users/$1.kubeconfig -n $2 exec client -- curl -s -m 3 -o /dev/null -w '%{http_code}\n' $3; }
```

**Before any policy**, Alice's pod reaches Bob's service:

```console
$ http alice team-a http://web.team-b
200
```

Now deny everything:

```bash
kubectl apply -f 03-network-policy/default-deny.yaml
```

```console
$ http alice team-a http://web
000
command terminated with exit code 28
```

Even Alice's own service fails. Egress is denied, and that includes the DNS lookup for `web`.
Allow DNS and same-namespace traffic:

```bash
kubectl apply -f 03-network-policy/allow-dns.yaml
kubectl apply -f 03-network-policy/allow-same-namespace.yaml
```

## Try it

```console
$ http alice team-a http://web
200
$ http alice team-a http://web.team-b
000
command terminated with exit code 28
$ http bob team-b http://web.team-a
000
command terminated with exit code 28
$ http bob team-b http://web
200
```

Each tenant reaches its own pods, and nothing across the boundary. Exit code 28 is curl's
timeout: the packets are dropped, not refused.

**If the cross-tenant calls still return `200`**, your network plugin does not enforce
NetworkPolicy. Install one that does (Calico, Cilium) before relying on this layer.

Alice cannot remove the isolation:

```console
$ alice -n team-a delete networkpolicy default-deny
Error from server (Forbidden): networkpolicies.networking.k8s.io "default-deny" is forbidden: User "alice" cannot delete resource "networkpolicies" in API group "networking.k8s.io" in the namespace "team-a"
```

## Clean up

Keep the policies and the client pods. Step 04 continues in the same namespaces.
