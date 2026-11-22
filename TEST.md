# Test report

Every "Try it" in this lab was run on a real cluster, and the expected output in each step's
README is copied from those runs.

| | |
|---|---|
| Date | 2026-11-22 |
| Cluster | kind v0.24.0, Kubernetes v1.36.1, 1 control-plane + 3 workers, default CNI (kindnet, enforces NetworkPolicy) |
| Method | All steps in order, from a clean cluster to the clean-up in step 06. |
| Result | Steps 00-05 + clean-up: **102 / 102 checks, 3 runs back to back**. With step 06 (tenant-insights) added: **114 / 114 checks**, 1 full run. |

A check passes only if the command gives the expected result: success where the lab expects
success, and the expected error message where the lab expects a denial.

## What was checked

| Step | Checks | Before the guardrail | After the guardrail |
|---|---|---|---|
| 00 setup | 10 | – | Alice and Bob authenticate with CSR-issued certificates; groups come from the certificate (`team-a-devs`, `team-b-devs`); both are **Forbidden** everything until RBAC exists |
| 01 namespace + RBAC | 15 | built-in `edit` role allows deleting NetworkPolicies | Alice cannot read team B's secrets, list namespaces, create a RoleBinding, delete NetworkPolicies or change quotas; she can read her quota; Bob cannot list team A's pods |
| 02 quota + limits | 17 | 10 × 0.5 CPU pods all run | only 4 run (quota 2 CPU); `exceeded quota` in ReplicaSet events; team B unaffected; LimitRange fills in defaults and rejects a 2-CPU container |
| 03 network policy | 14 | team A reaches team B's service (HTTP 200) | default-deny breaks even DNS until allowed; same-tenant traffic works (200); cross-tenant traffic blocked in both directions; Alice cannot delete the policy |
| 04 pod security | 13 | a privileged hostPath pod reads the node's disk, including another tenant's pod logs | server-side dry run warns first; the same pod is rejected by `restricted`; compliant pods and the existing `web` keep working; Alice cannot remove the label |
| 05 dedicated nodes | 24 | the taint repels team A's own pods; Bob's pod with team A's toleration runs on team A's node | NoExecute evicts other tenants' pods; team A lands only on its nodes (nodeSelector added by MutatingAdmissionPolicy); foreign toleration, wildcard toleration and `spec.nodeName` are all denied; team B stays on the shared node |
| 06 observe | 11 | – | tenant-insights installs with `TENANT_LABEL=tenant`; reports show group access and quota left; re-applying `hog` turns team A **Blocked** with the Pod Security reason and a fix hint; back to `OK` after removing it |
| clean-up | 10 | – | tenant-insights uninstalled; no taints, labels or admission policies left; Alice can do nothing |

## Not tested

- Managed clusters (EKS, AKS, GKE) and other network plugins. The concepts are the same, but
  NetworkPolicy enforcement depends on your plugin (step 03 shows how to check).
- Kubernetes versions before 1.36 for step 05. MutatingAdmissionPolicy is v1 from 1.36. Step 05
  explains the alternative for older clusters.
