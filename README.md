# Gitea (Validated Patterns chart) via Argo CD app-of-apps

```
bootstrap/root-app.yaml   # applied once, manually
apps/gitea.yaml           # child Application, synced by root
```

## Prerequisites (out of Git — contains credentials)
```bash
oc create namespace vp-gitea
oc -n vp-gitea create secret generic gitea-admin-secret \
  --from-literal=username=admin \
  --from-literal=password='Admin123!'
```
(Keys `username`/`password` are what templates/gitea/deployment.yaml reads.)
Use Sealed Secrets / External Secrets instead if you want this in Git.

## Fill placeholders
```bash
CLUSTER_DOMAIN=$(oc get dns.config.openshift.io/cluster -o jsonpath='{.spec.baseDomain}')
echo $CLUSTER_DOMAIN

sed -i -E "s#(https://gitea-route-vp-gitea\.apps\.)[^[:space:]]+#\1${CLUSTER_DOMAIN}#g" \
  /mnt/vm/Var/my_git-clone/acm/1.acm_procedures/acm_cluster_compare/app-of-app_helm_chart/gitea.yaml
echo "cluster domain: ${CLUSTER_DOMAIN}"
```

Set `repoURL` in `root-app.yaml`, then push.

## Deploy
```bash
oc apply -f bootstrap/root-app.yaml
oc -n openshift-gitops get applications
oc -n vp-gitea get route gitea-route
```
