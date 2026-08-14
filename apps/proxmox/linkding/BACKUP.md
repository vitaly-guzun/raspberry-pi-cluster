# Linkding backups on Synology

The `linkding-backup` CronJob creates a full Linkding backup every day at
03:15 in the `Europe/Amsterdam` time zone. Backups are transaction-safe ZIP
archives containing the SQLite database, bookmark assets, favicons, and preview
images.

The archives are stored on Synology at:

```text
/volume1/backups/linkding/linkding-YYYY-MM-DDTHH-MM-SSZ.zip
```

The CronJob validates each ZIP before copying it to Synology and only exposes
the final filename after the copy completes. It does not delete old backups;
configure retention or snapshots on Synology separately.

## Synology preparation

1. In DSM, enable NFS under **Control Panel -> File Services -> NFS**.
2. Create the `backups` shared folder on `volume1`.
3. Add an NFS permission for the Kubernetes node `192.168.1.225`
   (`k3s-proxmox-01`):
   - privilege: `Read/Write`;
   - squash: `Map all users to admin`;
   - security: `sys`.
4. Keep the rule restricted to that exact node address. If worker nodes are
   added later, grant access to their exact addresses as well. Do not expose
   the NFS service outside the LAN.
5. Make sure the NFS client package is installed on every worker node. On
   Debian/Ubuntu this package is `nfs-common`.

The manifest expects the NAS at `192.168.1.59` and the NFS export at
`/volume1/backups`. Update `backup-cronjob.yaml` if either value differs.

## First run

After Flux applies the manifest, start one backup manually:

```shell
job="linkding-backup-manual-$(date +%s)"
kubectl -n linkding create job --from=cronjob/linkding-backup "${job}"
kubectl -n linkding wait --for=condition=complete --timeout=1h "job/${job}"
kubectl -n linkding logs "job/${job}"
```

Then verify that a non-empty `.zip` file exists in
`/volume1/backups/linkding` on Synology and test it there with:

```shell
unzip -t /volume1/backups/linkding/linkding-YYYY-MM-DDTHH-MM-SSZ.zip
```

## Restore

Stop Linkding before restoring. Extract the selected archive, replace the
contents of `/etc/linkding/data` on the Linkding PVC with the extracted files,
and start Linkding again. Keep the current data until the restored application
has been verified.
