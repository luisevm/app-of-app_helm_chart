# Gitea (Validated Patterns chart) via Argo CD app-of-apps

## Clone repo
git clone https://github.com/luisevm/app-of-app_helm_chart.git

```
root-app.yaml      # applied once, manually
gitea.yaml         # child Application, synced by root
nonroot-scc.yaml   # SCC grant for the Gitea default ServiceAccount
```

## Prerequisites (out of Git — contains credentials)
```bash
oc create namespace vp-gitea
oc -n vp-gitea create secret generic gitea-admin-secret \
  --from-literal=username=gitea_admin \
  --from-literal=password='Admin123!'
```
(Keys `username`/`password` are what templates/gitea/deployment.yaml reads. Gitea rejects the name `admin` as reserved — the chart default is `gitea_admin`.)
Use Sealed Secrets / External Secrets instead if you want this in Git.

The `configure-gitea` init container runs as UID `1000`. The default `restricted-v2` SCC only allows the namespace UID range, so the pod stays forbidden. `nonroot-scc.yaml` binds the `default` ServiceAccount in `vp-gitea` to the `nonroot` SCC (`anyuid` also works, but it allows root). `gitea.yaml` sets `podSecurityContext.fsGroup: 1000` so that ServiceAccount can write the data volume:

```bash
oc apply -f nonroot-scc.yaml
```

## Fill placeholders
```bash
CLUSTER_DOMAIN=$(oc get dns.config.openshift.io/cluster -o jsonpath='{.spec.baseDomain}')
echo $CLUSTER_DOMAIN

sed -i -E "s#(https://gitea-route-vp-gitea\.apps\.)[^[:space:]]+#\1${CLUSTER_DOMAIN}#g" \
  /mnt/vm/Var/my_git-clone/acm/1.acm_procedures/acm_cluster_compare/app-of-app_helm_chart/gitea.yaml
echo "cluster domain: ${CLUSTER_DOMAIN}"
```

## Update repo
git add *
git commit -m "c"
git push

## Deploy
```bash
oc apply -f root-app.yaml
oc -n openshift-gitops get applications
oc -n vp-gitea get route gitea-route
```
