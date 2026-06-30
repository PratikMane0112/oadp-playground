# FS Backup on GCP 

```bash
# Confirm oc access
oc whoami
oc get csv -n openshift-adp 

# Confirm appm is available
source /Users/prmane/OADP/oadp-apps-deployer/.venv/bin/activate
appm list

#login
oc login -u kubeadmin -p <PASSWORD> https://api.<CLUSTER>:6443 --insecure-skip-tls-verify

#Create cloud credentials secret
oc create secret generic cloud-credentials -n openshift-adp --from-file=cloud=/Users/prmane/OADP/oadp-e2e-qe/oadp-pipeline-artifacts/credentials

#verify
oc get secret cloud-credentials -n openshift-adp

# DPA
cat <<'EOF' | oc apply -f -
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: ts-dpa
  namespace: openshift-adp
spec:
  configuration:
    velero:
      defaultPlugins:
        - openshift
        - gcp
    nodeAgent:
      enable: true
      uploaderType: kopia
  backupLocations:
    - velero:
        provider: gcp
        default: true
        objectStorage:
          bucket: <BUCKET_NAME>
          prefix: velero
        credential:
          name: cloud-credentials
          key: cloud
  snapshotLocations:
    - velero:
        provider: gcp
        config:
          project: <GCP_PROJECT_ID>
        credential:
          name: cloud-credentials
          key: cloud
EOF

# verify
oc get dpa -n openshift-adp

oc get pods -n openshift-adp -w

oc get bsl -n openshift-adp


# deploy app
source /Users/prmane/OADP/oadp-apps-deployer/.venv/bin/activate
appm list
appm deploy ocp-mysql

# Common choices for FSB testing: `ocp-mysql`, `ocp-mongodb`, `ocp-django`, `ocp-nginx`.

# Verify deployment

oc get all,pvc -n ocp-mysql

# Backup 
cat <<'EOF' | oc apply -f -
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: test-backup
  namespace: openshift-adp
spec:
  includedNamespaces:
    - ocp-mysql
  defaultVolumesToFsBackup: true
  storageLocation: <BSL_NAME>
EOF

# Check phase (wait until "Completed")
oc get backup test-backup -n openshift-adp -o jsonpath='{.status.phase}'

# Check PodVolumeBackups
oc get podvolumebackups -n openshift-adp -w

# Check backup progress
oc get backup test-backup -n openshift-adp -o jsonpath='{.status.progress}'


## Delete the application 
source /Users/prmane/OADP/oadp-apps-deployer/.venv/bin/activate
appm list
appm remove ocp-mysql

#verify
oc get ns ocp-mysql

# Restore
```bash
cat <<'EOF' | oc apply -f -
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: test-restore
  namespace: openshift-adp
spec:
  backupName: test-backup
  includedNamespaces:
    - ocp-mysql
  restorePVs: true
EOF

# Check phase (wait until "Completed")
oc get restore test-restore -n openshift-adp -o jsonpath='{.status.phase}'

# Check PodVolumeRestores
oc get podvolumerestores -n openshift-adp -w

# Validate restored app
oc get all,pvc -n ocp-mysql

source /path/to/oadp-apps-deployer/.venv/bin/activate
appm validate ocp-mysql

# cleanup
oc delete restore <RESTORE_NAME> -n openshift-adp

# oc delete backup` only removes the CR, not the data from object storage.
cat <<'EOF' | oc apply -f -
apiVersion: velero.io/v1
kind: DeleteBackupRequest
metadata:
  name: delete-test-backup
  namespace: openshift-adp
spec:
  backupName: test-backup
EOF

# verify
oc get backup -n openshift-adp

# remove app
source /path/to/oadp-apps-deployer/.venv/bin/activate
appm remove ocp-mysql

# dpa & secret
oc delete dpa ts-dpa -n openshift-adp
oc delete secret cloud-credentials -n openshift-adp
```


```text
# some debugging ......

| Resource             | Command 
|----------------------|-----------------------------------------
| DPA                  | `oc get dpa -n openshift-adp -o yaml` 
| BSL                  | `oc get bsl -n openshift-adp` 
| Backup               | `oc get backup -n openshift-adp` 
| Restore              | `oc get restore -n openshift-adp` 
| PodVolumeBackups     | `oc get podvolumebackups -n openshift-adp` 
| PodVolumeRestores    | `oc get podvolumerestores -n openshift-adp` 
| Node-agent pods      | `oc get pods -n openshift-adp -l name=node-agent` 
| Velero logs          | `oc logs deployment/velero -n openshift-adp` 
| Node-agent logs      | `oc logs -n openshift-adp -l name=node-agent --tail=50` 
| Node-agent ConfigMap | `oc get configmap node-agent-<dpa-name> -n openshift-adp -o yaml` 
| DeleteBackupRequest  | `oc get deletebackuprequest -n openshift-adp` 
```
