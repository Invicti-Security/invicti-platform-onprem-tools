# RHEL script

`invicti-platform-rhel.sh` — Invicti Platform on-premises on Red Hat Enterprise Linux 9
and its rebuilds, onto k3s (default) or RKE2.

```bash
sudo ./invicti-platform-rhel.sh install \
  --email you@company.com --license XXXX-XXXX-XXXX --host invicti.company.com
```

The point of this build is that it works on a host configured the way a Red Hat estate
normally is: **SELinux stays Enforcing** and **firewalld keeps running**. Neither is
disabled, and a full install completes with zero AVC denials.

| | |
|---|---|
| Distributions | RHEL 9, Rocky Linux 9, AlmaLinux 9, CentOS Stream 9, Oracle Linux 9 |
| Kubernetes | k3s (default) or RKE2, via `--k8s` |
| Architecture | x86_64 |

RHEL 8 and 10 are accepted with a warning. Anything outside the Red Hat family is
refused up front with a pointer to the Debian script.

## Flags worth knowing

```
--k8s <k3s|rke2>                      which Kubernetes to install (default k3s)
--selinux <enforcing|permissive>      default enforcing; permissive is runtime-only
--firewalld <configure|disable|skip>  default configure, keeps the firewall on
--scanner-min-replicas <n|default>    warm DAST scanners; auto-sized by default
--purge-data / --purge / --purge-all  uninstall scope, least to most destructive
```

`--selinux permissive` is a runtime `setenforce 0` only. `/etc/selinux/config` is never
edited, so a reboot returns the host to Enforcing.

## How it handles SELinux and firewalld

`container-selinux` is installed before the cluster, then k3s is started with
`--selinux` (or RKE2 with `selinux: true`), so the distribution's own policy module is
loaded rather than the host being dropped to permissive.

firewalld is left running. The two cluster CIDRs are trusted and only what is needed is
opened:

```
--zone=trusted --add-source=10.42.0.0/16   pod network
--zone=trusted --add-source=10.43.0.0/16   service network
--add-port=6443/tcp                        kube-apiserver
--add-port=80/tcp --add-port=443/tcp       platform ingress
--add-port=9345/tcp                        RKE2 supervisor   (RKE2 only)
--add-port=30000-32767/tcp                 NodePort range    (RKE2 only)
```

`uninstall --purge-all` reverts every one of them, and removes only the SELinux
file-context rule the script added.

## RKE2 specifics

RKE2 ships no storage provisioner, so the script installs Rancher's local-path and marks
it default, relabelling `/opt/local-path-provisioner` before the first volume is
requested — see [docs/troubleshooting.md](../docs/troubleshooting.md).

RKE2's Canal CNI creates flannel and calico interfaces that NetworkManager will reclaim,
killing cluster networking at the next reload. The drop-in at
`/etc/NetworkManager/conf.d/rke2-canal.conf` is written before RKE2 first starts, because
applying it afterwards needs a reboot. `nm-cloud-setup` is disabled if present.

## Notes

- **Unentitled RHEL has no repositories at all.** Preflight checks entitlement and
  enabled repos and says the one thing that needs doing, instead of producing a wall of
  `dnf` metadata errors. Rebuilds carry no `subscription-manager` and are skipped.
- **EPEL is not required.** Everything needed
  (`curl tar ca-certificates iscsi-initiator-utils lvm2 jq python3`) is in BaseOS or
  AppStream. `curl-minimal` is accepted rather than swapped for full `curl`.
- **Replacing the Kubernetes distribution needs a reboot.** The script detects leftover
  netfilter state and refuses to install over it; it cannot clear it for you.
