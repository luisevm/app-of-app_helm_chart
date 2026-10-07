# Gitea (Validated Patterns chart) via Argo CD app-of-apps

## Clone repo
git clone https://github.com/luisevm/app-of-app_helm_chart.git

```
root-app.yaml      # applied once, manually
gitea.yaml         # child Application, synced by root
```

## What gitea.yaml installs besides the chart
`root` syncs only `gitea.yaml`. These are Helm `extraDeploy` objects in that file:

- Role `gitea-argocd-secrets` (sync wave `-95`). The chart binds its admin Role at wave `-90` but creates that Role only at wave `0`, so the first sync cannot create Secrets and then waits on the Deployment forever.
- Secret `gitea-admin-secret` (sync wave `-1`): username `gitea_admin`, password `Admin123!`. Gitea rejects the name `admin`.
- RoleBinding `gitea-default-nonroot` (sync wave `-1`), so the `default` ServiceAccount can use the `nonroot` SCC. `configure-gitea` runs as UID `1000`, which `restricted-v2` rejects. `podSecurityContext.fsGroup: 1000` lets that user write the data volume.

The chart creates namespace `vp-gitea`.

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
oc -n openshift-gitops get applications.argoproj.io
oc -n vp-gitea get route gitea-route
```

## Delete
Delete `root` first. It owns `gitea-in-cluster`, and that Application owns the chart objects, including the cluster-scoped ConsoleLink `gitea-link`. Deleting the namespace while either Application still exists lets self-heal recreate it.

The chart marks PVC `gitea-shared-storage` with `helm.sh/resource-policy: keep`, so Argo leaves it. Deleting namespace `vp-gitea` removes that PVC, Secret `gitea-admin-secret`, and RoleBinding `gitea-default-nonroot`. The storage class reclaim policy is `Delete`, so the bound volume is removed with the claim.

```bash
oc -n openshift-gitops delete applications.argoproj.io root --wait=true
oc delete namespace vp-gitea --ignore-not-found --wait=true
oc delete consolelink gitea-link --ignore-not-found
```
