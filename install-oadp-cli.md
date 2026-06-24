## Install OADP CLI from cluster


### Get the route dynamically and download the binary

```bash
ROUTE=$(oc get route oadp-cli-server-route -n openshift-adp -o jsonpath='{.spec.host}') && \
curl -k "https://${ROUTE}/download/kubectl-oadp_darwin_arm64" -o /tmp/kubectl-oadp && \
chmod +x /tmp/kubectl-oadp && \
mkdir -p ~/bin && cp /tmp/kubectl-oadp ~/bin/kubectl-oadp && \
export PATH="$HOME/bin:$PATH" && \
oc oadp version
```
