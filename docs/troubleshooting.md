# Troubleshooting

Failure modes seen on real hosts. Each one points somewhere other than its cause, which
is why they are written down.

## All PVCs Pending, ~20 pods CrashLoopBackOff (RKE2)

```
mkdir: can't create directory '/opt/local-path-provisioner/pvc-...': Permission denied
```

Nothing in that failure mentions SELinux, but SELinux is the cause. RKE2 ships no storage
provisioner, so the script installs Rancher's local-path, which writes volumes to
`/opt/local-path-provisioner` from inside a helper pod.

`container-selinux` ships a file-context rule mapping that path to `container_file_t`,
but the rule alone is not enough: the directory is created at runtime, inherits `usr_t`
from `/opt`, and nothing ever runs `restorecon` on it. No PVC binds, PostgreSQL never
gets a volume, and everything behind it crash-loops.

The script creates and relabels the directory before the first volume is requested. k3s
is unaffected — its bundled provisioner uses `/var/lib/rancher/k3s/storage`, which
`k3s-selinux` labels at install time.

## Pods report `no route to host` after replacing the Kubernetes distribution

Both k3s and RKE2 leave netfilter tables and virtual interfaces behind after their own
uninstallers finish. Install on top of that residue and every component checks out
individually:

- pods cannot reach the service VIP (`10.43.0.1`),
- the host reaches that same address perfectly (`HTTP 401`),
- kube-proxy's `KUBE-SERVICES` rules are present and correct,
- firewalld is configured correctly, and SELinux logs zero denials.

Only a reboot clears it. **Reboot between Kubernetes distributions.** The script refuses
to install over the residue unless you pass `--force`, and `uninstall --purge-all` tells
you to reboot before installing again.

It is also not self-healing. While the network is broken the chart's
`invicti-certificate-generator` and `invicti-secret-generator` Jobs exhaust their backoff
limit, and a Kubernetes Job that has Failed is never retried. So
`invicti-generated-secrets` never exists and ~40 pods sit in `CreateContainerConfigError`
even after connectivity is restored. Recovery is `uninstall --purge` and a fresh install.

## Reinstall fails on ownership metadata conflicts

KEDA's CRDs, ClusterRoles and webhooks are cluster-scoped, and Helm never deletes
anything from a chart's `crds/` directory. The CRDs also carry the
`customresourcecleanup.apiextensions.k8s.io` finalizer, so once the KEDA operator is gone
nothing can finish cleaning up the custom resources and the CRD sits in `Terminating`
forever:

```
$ kubectl get crd | grep keda
scaledjobs.keda.sh              2026-08-18T08:59:58Z
triggerauthentications.keda.sh  2026-08-18T08:59:58Z
$ kubectl get crd scaledjobs.keda.sh -o jsonpath='{.metadata.finalizers}'
["customresourcecleanup.apiextensions.k8s.io"]
```

Always use `uninstall --purge` before a reinstall. It strips the finalizers, retries the
delete, then re-checks and reports what survived rather than claiming success.

## Scanner pods stuck Pending forever

The chart keeps `minReplicaCount: 4` warm DAST scanners, each requesting 2 CPU and 6 Gi
plus a 1 Gi sidecar — roughly 28 Gi before a single scan runs. On a smaller node the
surplus never schedules and looks like a broken install.

Both scripts size this to the node automatically (`--scanner-min-replicas`, default
auto) and prune the stale scanner Jobs KEDA leaves behind when the count is lowered.

## Most pods Pending after install

`values-resources-recommended.yaml` is high-availability multi-node sizing. On one node
most pods stay Pending. The scripts pick a profile that fits the node with an automatic
fallback, and record the profile that worked in `~/invicti-onprem/.invicti-state` so
`upgrade` and `reconfigure` reuse it instead of re-deriving a heavier one.

## Rolling upgrade never converges on a single node

The default Deployment strategy (`maxSurge=1, maxUnavailable=0`) needs the new pod Ready
before the old one is removed, so both have to fit at once. On a tight node the rollout
deadlocks and Helm times out. Affected Deployments are switched to stop-then-start.

## Collecting a support bundle

```bash
./invicti-platform.sh logs
```

`license_key` is redacted; the account email remains, because Invicti Support needs it to
identify the account. Pod logs are not scrubbed — review the bundle before sending it
outside the company.
