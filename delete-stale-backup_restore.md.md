# OADP Test Resource Cleanup Guide

## Step 1: Check what exists

```bash
oc get backup.velero.io -n openshift-adp
oc get restore.velero.io -n openshift-adp
oc get dataupload.velero.io -n openshift-adp
oc get datadownload.velero.io -n openshift-adp
oc get backuprepository -n openshift-adp
```

## Step 2: Check if Velero pod is running

```bash
oc get pods -n openshift-adp | grep velero
```

## Step 3A: If Velero IS running (DPA exists)

Use `DeleteBackupRequest` to properly remove backup data from S3:

```bash
for backup in $(oc get backup.velero.io -n openshift-adp -o name); do
  NAME=$(echo $backup | cut -d/ -f2)
  cat <<EOF | oc apply -f -
apiVersion: velero.io/v1
kind: DeleteBackupRequest
metadata:
  name: delete-${NAME:0:50}
  namespace: openshift-adp
spec:
  backupName: $NAME
EOF
done

# Delete restores directly
oc delete restore.velero.io --all -n openshift-adp
```

## Step 3B: If Velero is NOT running (DPA already deleted)

Resources get stuck due to finalizers. Patch them out first:

```bash
# Remove finalizers from restores
for restore in $(oc get restore.velero.io -n openshift-adp -o jsonpath='{.items[*].metadata.name}'); do
  oc patch restore.velero.io "$restore" -n openshift-adp --type merge -p '{"metadata":{"finalizers":null}}'
done

# Remove finalizers from backups
for backup in $(oc get backup.velero.io -n openshift-adp -o jsonpath='{.items[*].metadata.name}'); do
  oc patch backup.velero.io "$backup" -n openshift-adp --type merge -p '{"metadata":{"finalizers":null}}'
done

# Delete everything
oc delete backup.velero.io --all -n openshift-adp
oc delete restore.velero.io --all -n openshift-adp
oc delete dataupload.velero.io --all -n openshift-adp
oc delete datadownload.velero.io --all -n openshift-adp
oc delete backuprepository --all -n openshift-adp
oc delete deletebackuprequest.velero.io --all -n openshift-adp
oc delete jobs --all -n openshift-adp
```

## Step 4: Verify clean

```bash
oc get backup.velero.io,restore.velero.io,dataupload.velero.io,datadownload.velero.io,backuprepository -n openshift-adp
```

## One-Liner (removes all Velero resources regardless of state)

```bash
for type in backup restore dataupload datadownload backuprepository; do
  for item in $(oc get ${type}.velero.io -n openshift-adp -o jsonpath='{.items[*].metadata.name}' 2>/dev/null); do
    oc patch ${type}.velero.io "$item" -n openshift-adp --type merge -p '{"metadata":{"finalizers":null}}' 2>/dev/null
  done
  oc delete ${type}.velero.io --all -n openshift-adp 2>/dev/null
done
oc delete deletebackuprequest.velero.io --all -n openshift-adp 2>/dev/null
oc delete jobs --all -n openshift-adp 2>/dev/null
```

**IMPORTANT:** Always delete `backuprepository` CRs! If left behind from a previous run, they cause
"repository not initialized" errors on the next test because the new DPA uses a different S3 prefix.

## Deleting BackupRepository — Official References

Deleting `BackupRepository` CRs is officially documented and safe in the following scenarios:

**When it's safe to delete:**
- All backups referencing it have been deleted
- The BSL (BackupStorageLocation) no longer exists
- Between test runs where each run creates a new BSL prefix
- Velero auto-creates a fresh BackupRepository when the next backup starts

**Official procedure (from Red Hat OADP docs):**

```bash
# List backup repositories
oc get backuprepositories.velero.io -n openshift-adp

# Delete a specific backup repository
oc delete backuprepository <backup_repository_name> -n openshift-adp

# Or delete all (safe between test runs)
oc delete backuprepository --all -n openshift-adp
```

**Key quote from OADP TROUBLESHOOTING.md:**
> "Once a backup and related artifacts are deleted and no active backups related to the
> related namespace it is recommended to delete the backupRepository object."

**Key quote from Broadcom KB (Velero/TMC):**
> "Velero will automatically create a backuprepository if it does not exist once the
> corresponding backup is started."

**Note on production:** In production environments, wait up to 72 hours after backup deletion
before removing BackupRepository CRs — this allows Kopia maintenance cycles to garbage collect
artifacts from S3. For test environments, immediate deletion is safe since you don't need
the backup data.

## Why do resources get stuck?

Velero adds `restores.velero.io/external-resources-finalizer` to restores (and similar finalizers
to backups). These finalizers tell Kubernetes: "don't delete this until Velero processes it."
If the Velero pod isn't running (because the DPA was deleted by test cleanup), no controller
exists to remove the finalizer, so the resource stays in `Terminating` state forever.
Patching `finalizers` to `null` tells Kubernetes to skip that check.

## Common errors from incomplete cleanup

| Error | Cause | Fix |
|-------|-------|-----|
| `repository not initialized in the provided storage` | Stale `BackupRepository` CR from previous run | Delete all `backuprepository` CRs |
| Resources stuck in `Terminating` | Finalizers without a running Velero server | Patch finalizers to null |
| `AlreadyExists` on backup/restore creation | Old CRs not deleted | Delete all backup/restore CRs |

**Sources:**
- Red Hat OADP Docs: https://docs.okd.io/4.21/backup_and_restore/application_backup_and_restore/backing_up_and_restoring/oadp-deleting-backups.html
- OADP Operator Troubleshooting: https://github.com/openshift/oadp-operator/blob/master/docs/TROUBLESHOOTING.md
- Upstream Velero Discussion: https://github.com/velero-io/velero/discussions/9417
- Kopia Troubleshooting: https://github.com/openshift/oadp-operator/blob/master/docs/kopia_troubleshooting.md
