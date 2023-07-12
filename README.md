# Using Crossplane to deploy ArgoCD on vcluster

Run your own [k3d](https://k3d.io/) cluster with [Crossplane](https://www.crossplane.io/) and deploy [ArgoCD](https://argoproj.github.io/cd/) instances onto [vclusters](https://www.vcluster.com/)
![Architecture](architecture.png "Architecture")

---

## Requirements to run this demo

1. k3d
2. Helm
3. no VPN

---

## What's in this repository?

1. [Initialization script](scripts/init-k3d-demo-env.sh) - deploys k3d cluster and installs ArgoCD onto it. It also applies the [argocd-applications](argocd-applications) to this cluster.
2. [crossplane-resources](crossplane-resources) - Contains the providers required to deploy our virtualargocd composite resource, along with the [definition](crossplane-resources/xvirtualargocd/definition.yaml) and the [composition](crossplane-resources/xvirtualargocd/composition.yaml)
3. [virtualargocds](virtualargocds) - contains the composite resource claims. Here we will define all instances of the composition to be created.
4. [argocd-applications](argocd-applications) - contains ArgoCD applications so that ArgoCD syncs it all to the cluster. One of the ArgoCD appplications deploys crossplane itself to the cluster.

---

## How to run the demo?

1. Execute initialization script.

**Notes**

```bash
export KUBECONFIG=/tmp/k3s-crossplane.config
# check claim
k get virtualargocds.demo.codefresh.io -n crossplane-system
# get XR name from claim
XR_NAME=$(k get virtualargocds.demo.codefresh.io -n crossplane-system customer1 -o yaml -o jsonpath='{.spec.resourceRef.name}')
echo "XR_NAME=$XR_NAME"
# check XR
k get xvirtualargocds.demo.codefresh.io $XR_NAME
# get argocd svc
ARGO_SVC=$(k get svc -n argocd -l app.kubernetes.io/name=argocd-server -o name)
# get argocd admin password
ARGO_PASSWORD=$(k get secret -n argocd argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)
# Access vcluster Argocd
k port-forward -n argocd $VC_ARGO_SVC 8080:80
# open http://localhost:8081 and login with admin/$VC_ARGO_PASSWORD

## ---------- In another terminal ----------

# check vcluster
vcluster list
VC_NAME="${XR_NAME}-vcluster"
# port-forward vcluster
VC_SVC=$(k get svc -l app=vcluster -o name)
k port-forward -n $XR_NAME $VC_SVC 4443:443
[...]

## ---------- In another terminal ----------

# get vcluster kubeconfig
vcluster connect ${VC_NAME} -n ${XR_NAME} --server localhost:4443 --update-current=false --kube-config-context-name="vc-${XR_NAME}" --kube-config="/tmp/vc-${XR_NAME}-kubeconfig.conf"
# connect to vcluster
export KUBECONFIG="/tmp/vc-${XR_NAME}-kubeconfig.conf"
k get po -A
# get argocd svc
VC_ARGO_SVC=$(k get svc -n argocd -l app.kubernetes.io/name=argocd-server -o name)
# get argocd admin password
VC_ARGO_PASSWORD=$(k get secret -n argocd argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)
# Access vcluster Argocd
k port-forward -n argocd $VC_ARGO_SVC 8081:80
# open http://localhost:8081 and login with admin/$VC_ARGO_PASSWORD
```

2. Once all ArgoCD applications are synced, uncomment [customer2.yaml](virtualargocds/customer2.yaml), see that a namespace is created, with vcluster and ArgoCD deployed onto it.
