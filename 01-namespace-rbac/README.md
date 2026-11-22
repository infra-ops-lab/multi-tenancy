# 01 · Namespace + RBAC: the boundary and the lock

## Concept

A **namespace** is a name scope: two teams can both have a Deployment called `web` without
colliding. On its own it isolates nothing. **RBAC** is what turns it into a tenant boundary:

- A **ClusterRole** is a reusable list of permissions. It grants nothing by itself.
- A **RoleBinding** grants that list to a subject **inside one namespace**.
- So one ClusterRole `tenant-developer`, bound once per tenant namespace, gives every team the same
  rights, each in its own space only.

```
 group team-a-devs ──RoleBinding (in team-a)──► ClusterRole tenant-developer
 group team-b-devs ──RoleBinding (in team-b)──► ClusterRole tenant-developer
```

Two design choices matter more than the YAML itself:

1. **Bind groups, not users.** Onboarding and offboarding then happen in the identity provider.
2. **Don't give tenants the built-in `edit` role.** It looks safe, but it lets the tenant create
   and delete NetworkPolicies, which means a tenant could delete the network isolation you add in
   step 03. Check it yourself:

   ```console
   $ kubectl describe clusterrole edit | grep networkpolicies
     networkpolicies.networking.k8s.io   []   []   [create delete deletecollection patch update get list watch]
   ```

   Our [tenant-developer](tenant-developer-clusterrole.yaml) role lets the tenant run and debug
   workloads, and only **read** the guardrails the platform team owns: quotas, limit ranges and
   network policies. It has no RBAC rights at all, so a tenant cannot promote itself.

## What it does not protect

RBAC controls **API calls**. Everything below the API is still shared and wide open:

- Alice's pods can use as much CPU and memory as the nodes have (step 02).
- Alice's pods can connect to Bob's pods over the network (step 03).
- Alice can create a privileged pod that reads the node's disk, including other tenants' data (step 04).
- Alice's pods can land on any node, next to anyone (step 05).

Also note: anyone who can create a pod in a namespace can read every Secret in it, by mounting it.
Keep secrets for different trust levels in different namespaces.

## Apply

Read both files first, then apply them as the admin:

```bash
kubectl apply -f 01-namespace-rbac/tenant-developer-clusterrole.yaml
kubectl apply -f 01-namespace-rbac/rolebindings.yaml
```

## Try it

Each team deploys a small web server ([web.yaml](web.yaml)). We use it again in steps 02 to 05.

```console
$ alice -n team-a apply -f 01-namespace-rbac/web.yaml
deployment.apps/web created
service/web created
$ bob -n team-b apply -f 01-namespace-rbac/web.yaml
deployment.apps/web created
service/web created
```

Now try to cross the boundary:

```console
$ alice get secrets -n team-b
Error from server (Forbidden): secrets is forbidden: User "alice" cannot list resource "secrets" in API group "" in the namespace "team-b"

$ bob get pods -n team-a
Error from server (Forbidden): pods is forbidden: User "bob" cannot list resource "pods" in API group "" in the namespace "team-a"

$ alice get namespaces
Error from server (Forbidden): namespaces is forbidden: User "alice" cannot list resource "namespaces" in API group "" at the cluster scope
```

Try to remove the guardrails:

```console
$ alice -n team-a create rolebinding me-admin --clusterrole=admin --user=alice
error: failed to create rolebinding: rolebindings.rbac.authorization.k8s.io is forbidden: User "alice" cannot create resource "rolebindings" in API group "rbac.authorization.k8s.io" in the namespace "team-a"

$ alice auth can-i delete networkpolicies -n team-a
no
$ alice auth can-i update resourcequotas -n team-a
no
$ alice auth can-i get resourcequotas -n team-a
yes
```

See everything Alice may do in her namespace:

```console
$ alice auth can-i --list -n team-a
Resources                             Non-Resource URLs   Resource Names   Verbs
pods                                  []                  []               [get list watch create update patch delete]
secrets                               []                  []               [get list watch create update patch delete]
deployments.apps                      []                  []               [get list watch create update patch delete]
...
limitranges                           []                  []               [get list watch]
resourcequotas                        []                  []               [get list watch]
networkpolicies.networking.k8s.io     []                  []               [get list watch]
```

## Clean up

Keep everything, including both `web` deployments. Step 02 builds on it.
