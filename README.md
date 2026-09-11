# Invicti Platform On-Premises Tools

Single-file lifecycle managers for the Invicti Platform on-premises edition, deployed by
Helm onto a single-node Kubernetes cluster or an existing one.

Each script installs, checks, upgrades, reconfigures, backs up, restores and removes the
platform. Pick the one that matches the host distribution:

| Host | Script | Kubernetes |
|---|---|---|
| RHEL 9, Rocky, AlmaLinux, CentOS Stream, Oracle Linux | [`rhel/invicti-platform-rhel.sh`](rhel/) | k3s or RKE2, SELinux Enforcing, firewalld running |
| Ubuntu, Debian | [`debian/invicti-platform.sh`](debian/) | k3s |

They are not interchangeable. The RHEL script is `dnf`-based and configures SELinux and
firewalld; the Debian script is `apt`-based and assumes neither.

## Quick start

```bash
sudo ./rhel/invicti-platform-rhel.sh install \
  --email you@company.com --license XXXX-XXXX-XXXX --host invicti.company.com
```

Run `check` first on an unfamiliar host — it runs every preflight test and changes
nothing. `--help` lists every flag; each flag has a matching environment variable for
non-interactive use.

## Commands

| Command | What it does |
|---|---|
| `install` | Preflight, install Kubernetes and Helm 3 if needed, deploy the chart |
| `check` | Every preflight test, changes nothing |
| `status` | Cluster, release, pods, storage, networking, events, reachability |
| `upgrade` | `helm upgrade` to the latest or a pinned chart version |
| `reconfigure` | Re-render `values.yaml` from flags and apply it |
| `backup` | Quiesced backup: values, secrets, PVC data, chart version |
| `restore` | Restore an archive produced by `backup` |
| `uninstall` | Release only, or `--purge-data` / `--purge` / `--purge-all` |
| `logs` | Redacted support bundle for Invicti Support |
| `version` | Script, chart, Helm, Kubernetes and OS versions |

## Sizing

The documented minimum is 6 CPU / 12 GB RAM / 50 GB disk per worker node, but that
assumes a multi-node cluster. For a usable single-node install budget **8+ CPU and 32 GB
RAM**. Less than that works, but the scripts will scale warm DAST scanners down and
select a lighter resource profile to fit. Production storage guidance is 1.2–1.5 TB,
because SeaweedFS holds scan artefacts.

Both scripts are single-node oriented: sizing heuristics read local `/proc/meminfo`, so
managed clusters (EKS, AKS, GKE, OpenShift) are out of scope.

## Before you uninstall and reinstall

Use `uninstall --purge`, not plain `uninstall`. The chart ships KEDA's CRDs, Helm never
deletes anything from a chart's `crds/` directory, and those CRDs carry a finalizer that
cannot complete once the KEDA operator is gone. Leave them behind and the next install
fails on ownership conflicts. `--purge` strips the finalizers and reports honestly if
anything survives.

See [docs/troubleshooting.md](docs/troubleshooting.md) for the rest.

## Security

- `values.yaml` is written mode `0600` and contains the license key. It is gitignored.
- Backup archives contain secrets, the license key and any SMTP or database passwords.
  Store them accordingly.
- The `logs` support bundle redacts `license_key`. Pod logs are **not** scrubbed —
  review a bundle before sending it outside the company.

## Reference

- [Invicti Platform on-premises documentation](https://docs.invicti.com/ip/category/invicti-platform-on-premises)
- [Helm prerequisites](https://docs.invicti.com/ip/helm-prerequisites)
- [Kubernetes requirements](https://docs.invicti.com/ip/helm-k8s-requirements)
- [On-premises trustlist](https://docs.invicti.com/ip/trustlist-on-premises)

Requires a valid Invicti Platform license.
