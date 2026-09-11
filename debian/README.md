# Debian and Ubuntu script

`invicti-platform.sh` — Invicti Platform on-premises on Ubuntu or Debian, onto k3s.

```bash
sudo ./invicti-platform.sh install \
  --email you@company.com --license XXXX-XXXX-XXXX --host invicti.company.com
```

Production, with mail and a real certificate:

```bash
sudo ./invicti-platform.sh install --yes \
  --email you@company.com --license XXXX --host invicti.company.com \
  --smtp-host smtp.company.com --smtp-port 587 \
  --smtp-user apikey --smtp-pass 'secret' --smtp-from no-reply@company.com \
  --tls-cert /etc/ssl/certs/invicti.pem --tls-key /etc/ssl/private/invicti.key
```

This script is `apt`-based and assumes SELinux and firewalld are not in play. On Red Hat
use [`rhel/invicti-platform-rhel.sh`](../rhel/) instead.

## Uninstall levels

```bash
./invicti-platform.sh uninstall               # release only, data survives
./invicti-platform.sh uninstall --purge-data   # + PVCs and namespace  (DESTROYS DATA)
./invicti-platform.sh uninstall --purge        # + cluster-scoped leftovers  <-- before a clean reinstall
./invicti-platform.sh uninstall --purge-all    # + k3s itself
```

`--purge` also clears stuck namespaces, PVC finalizers and released PersistentVolumes so
storage can rebind.

## Backup and restore

```bash
./invicti-platform.sh backup --output /mnt/backups/invicti-$(date +%F).tar.gz
./invicti-platform.sh backup --no-pvc                                  # config only, stays online
./invicti-platform.sh restore --from /mnt/backups/invicti-2026-08-18.tar.gz
```

StatefulSets are scaled to zero before PVCs are read, so the copy is consistent. The
Helm release secret (`sh.helm.release.v1.*`) is excluded, because restoring it breaks a
fresh install on a new cluster. Restore works into a different cluster.

The archive is mode `0600` and holds the license key plus any SMTP and database
passwords. Treat it as a secret.

## Notes

- **k3s is installed with `--disable=traefik`.** k3s bundles Traefik, which claims host
  ports 80 and 443 through its own `svclb` DaemonSet; the chart's `nginx-service` is also
  a LoadBalancer on 443, so its svclb pod would sit `Pending` forever. An existing
  Traefik is detected and can be retired.
- **Ubuntu's guided LVM installer leaves most of the disk unallocated.** That caps node
  ephemeral-storage and makes pods unschedulable while `df` still shows free space. The
  script offers to reclaim it.
- **Helm 3 is pinned.** Helm releases after 4.1.1 apply server-side and fight KEDA for
  ownership of ScaledJob fields. A snap-installed Helm 4 is removed, because it wins
  `$PATH`.

## Non-interactive use

```bash
INVICTI_EMAIL=... INVICTI_LICENSE_KEY=... PLATFORM_HOST=... \
  ./invicti-platform.sh install --yes
```

Every flag has a matching environment variable. If you are not root and cannot be
prompted, run the whole script under `sudo`, export `SUDO_ASKPASS`, or grant the user
passwordless sudo.

## Files it creates

| Path | Contents |
|---|---|
| `~/invicti-onprem/values.yaml` | Rendered chart values, mode 0600 |
| `~/invicti-onprem/.invicti-state` | Profile, chart and scanner settings that worked |
| `~/invicti-onprem/onpremises/` | Extracted chart |
| `~/invicti-backups/` | Backup archives |
| `~/invicti-logs/` | Run logs and support bundles |
