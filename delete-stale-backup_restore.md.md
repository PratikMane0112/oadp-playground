# OADP Test Resource Cleanup Guide

## Step 1: Check what exists

```bash
oc get backup.velero.io -n openshift-adp
oc get restore.velero.io -n openshift-adp
oc get dataupload.velero.io -n openshift-adp
oc get datadownload.velero.io -n openshift-adp
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
oc delete deletebackuprequest.velero.io --all -n openshift-adp
```

## Step 4: Verify clean

```bash
oc get backup.velero.io,restore.velero.io,dataupload.velero.io,datadownload.velero.io -n openshift-adp
```

## One-Liner (removes all Velero resources regardless of state)

```bash
for type in backup restore dataupload datadownload; do
  for item in $(oc get ${type}.velero.io -n openshift-adp -o jsonpath='{.items[*].metadata.name}' 2>/dev/null); do
    oc patch ${type}.velero.io "$item" -n openshift-adp --type merge -p '{"metadata":{"finalizers":null}}' 2>/dev/null
  done
  oc delete ${type}.velero.io --all -n openshift-adp 2>/dev/null
done
oc delete deletebackuprequest.velero.io --all -n openshift-adp 2>/dev/null
```

## Why do resources get stuck?

Velero adds `restores.velero.io/external-resources-finalizer` to restores (and similar finalizers to backups). 
These finalizers tell Kubernetes: "don't delete this until Velero processes it." If the Velero pod isn't running 
(because the DPA was deleted by test cleanup), no controller exists to remove the finalizer, so the resource stays 
in `Terminating` state forever. Patching `finalizers` to `null` tells Kubernetes to skip that check.
