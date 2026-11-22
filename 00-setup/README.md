# 00 · Setup: tenants and identities

## Concept

Kubernetes has **no user or group objects**. You cannot `kubectl create user`. The API server
trusts an identity that something else has already proven:

- a **client certificate** signed by the cluster CA: the certificate's `CN` is the username and
  each `O` is a group;
- a **token** from an identity provider (OIDC): claims in the token are mapped to username and
  groups. This is what you use in production (SSO, offboarding in one place).

Either way, a group is just a string the API server receives with each request. RBAC in the next
step grants permissions to those strings. That is why we design tenants around **groups**, not
individual users: people join and leave, the group binding stays.

This lab uses client certificates because they work on any cluster without changing the API
server's configuration. The isolation steps that follow are identical with OIDC.

```
 alice ──cert: CN=alice, O=team-a-devs──► API server ──► "user alice, groups [team-a-devs]" ──► RBAC (step 01)
```

## What it does not protect

Nothing yet. Authentication answers *who are you*, not *what may you do*. Right after this step
Alice can log in and is forbidden everything.

Client certificates also cannot be revoked one by one. Removing someone means waiting for the
certificate to expire, which is why the CSR below asks for 24 hours only. OIDC with short token
lifetimes is the production answer.

## Apply

Create the two tenant namespaces ([namespaces.yaml](namespaces.yaml)). The `tenant` label is how
later layers recognise which tenant owns a namespace.

```bash
kubectl apply -f 00-setup/namespaces.yaml
```

Create a user. Set three variables, then run the block. Do it once for Alice, then again for Bob
(`U=bob G=team-b-devs NS=team-b`).

```bash
U=alice G=team-a-devs NS=team-a
mkdir -p users/$U

# 1. The user makes a private key and a signing request. The key never leaves their machine.
openssl genrsa -out users/$U/$U.key 2048
openssl req -new -key users/$U/$U.key -subj "/CN=$U/O=$G" -out users/$U/$U.csr

# 2. The admin submits it to the cluster CA and approves it.
sed -e "s/__USER__/$U/" -e "s/__CSR__/$(base64 < users/$U/$U.csr | tr -d '\n')/" \
  00-setup/csr-template.yaml | kubectl apply -f -
kubectl certificate approve $U
kubectl get csr $U -o jsonpath='{.status.certificate}' | base64 -d > users/$U/$U.crt

# 3. Build a kubeconfig for the user: same cluster, their own certificate.
kubectl config view --minify --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' \
  | base64 -d > users/$U/ca.crt
SERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
KC=users/$U.kubeconfig
kubectl --kubeconfig $KC config set-cluster lab --server=$SERVER --certificate-authority=users/$U/ca.crt --embed-certs
kubectl --kubeconfig $KC config set-credentials $U --client-certificate=users/$U/$U.crt --client-key=users/$U/$U.key --embed-certs
kubectl --kubeconfig $KC config set-context $U --cluster=lab --user=$U --namespace=$NS
kubectl --kubeconfig $KC config use-context $U
```

The CSR object is in [csr-template.yaml](csr-template.yaml). Read its comments: the signer name says
"client certificate for the API server", and `expirationSeconds` keeps it short-lived.

Two short aliases make the rest of the lab easier to read:

```bash
alias alice='kubectl --kubeconfig users/alice.kubeconfig'
alias bob='kubectl --kubeconfig users/bob.kubeconfig'
```

## Try it

Check that the admin issued both certificates:

```console
$ kubectl get csr alice bob
NAME    AGE   SIGNERNAME                            REQUESTOR          REQUESTEDDURATION   CONDITION
alice   10s   kubernetes.io/kube-apiserver-client   kubernetes-admin   24h                 Approved,Issued
bob     6s    kubernetes.io/kube-apiserver-client   kubernetes-admin   24h                 Approved,Issued
```

Ask the API server who Alice is. The group came from the certificate's `O` field:

```console
$ alice auth whoami
ATTRIBUTE                                           VALUE
Username                                            alice
Groups                                              [team-a-devs system:authenticated]
Extra: authentication.kubernetes.io/credential-id   [X509SHA256=5ee9f7…]
```

Alice is authenticated, but authorised for nothing:

```console
$ alice get pods -n team-a
Error from server (Forbidden): pods is forbidden: User "alice" cannot list resource "pods" in API group "" in the namespace "team-a"
```

**Forbidden** (403) means "I know who you are, and the answer is no". **Unauthorized** (401) would
mean the certificate was not accepted at all.

## Clean up

Keep everything. Step 01 builds on it. The full clean-up is in [06-beyond](../06-beyond/#clean-up).
