## Crossplane + vcluster with provider-argocd (GitOps)

This repo contains a crossplane composition to spin up [virtual Kubernetes clusters](https://www.vcluster.com/) which are automatically registered at [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) (via [provider-argocd](https://github.com/crossplane-contrib/provider-argocd)):

![](argocd.png)

###  Prerequisites

- [Docker](https://docs.docker.com/engine/install/ubuntu/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
- [helm](https://helm.sh/docs/intro/install/)
- [k3d](https://k3d.io/#installation)
- [vcluster](https://www.vcluster.com/docs/getting-started/setup)
- [kubectl-crossplane](https://docs.crossplane.io/v1.10/getting-started/install-configure/)
```
### Install vlcuster CLI (example):
curl -L -o vcluster "https://github.com/loft-sh/vcluster/releases/latest/download/vcluster-linux-amd64" && sudo install -c -m 0755 vcluster /usr/local/bin && rm -f vcluster
### Install kubectl-crossplane CLI
curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh
```

### Setup environment

```
cd scripts/ && ./init-k3d-demo-env.sh

Example Output:

$ ./init-k3d-demo-env.sh 
....
....
....
ArcgoCD admin password:
V7JBqouePrCmtzwj
```

add a user for [provider-argocd](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/):
```bash

kubectl apply -f user-provider-argcod.yaml

kubectl patch configmap/argocd-cm \
  -n argocd \
  --type merge \
  -p '{"data":{"accounts.provider-argocd":"apiKey, login"}}'

kubectl patch configmap/argocd-rbac-cm \
  -n argocd \
  --type merge \
  -p '{"data":{"policy.csv":"g, provider-argocd, role:admin"}}'
```
create JWTs and a corresponding Kubernetes Secret, so that `provider-argocd` is able to connect to ArgoCD:
```bash
ARGOCD_ADMIN_SECRET=V7JBqouePrCmtzwj
ARGOCD_ADMIN_TOKEN=$(curl -s -X POST -k -H "Content-Type: application/json" --data '{"username":"admin","password":"'$ARGOCD_ADMIN_SECRET'"}' http://argocd.192.168.1.99.nip.io:8080/api/v1/session | jq -r .token)
ARGOCD_TOKEN=$(curl -s -X POST -k -H "Authorization: Bearer $ARGOCD_ADMIN_TOKEN" -H "Content-Type: application/json" http://argocd.192.168.1.99.nip.io:8080/api/v1/account/provider-argocd/token | jq -r .token)
kubectl create ns crossplane-system
kubectl create secret generic argocd-credentials -n crossplane-system --from-literal=authToken="$ARGOCD_TOKEN"
```

## Install crossplane
```bash
kubectl create ns crossplane-system
helm install crossplane --namespace crossplane-system crossplane-stable/crossplane
```

### provider-helm
install [provider-helm](https://github.com/crossplane-contrib/provider-helm) to later install the `vcluster` chart via compositions
```bash
./kubectl-crossplane install provider crossplane/provider-helm:v0.9.0

# in-cluster rbac, so that provider-helm is allowed to install helm charts into the host cluster
SA=$(kubectl -n crossplane-system get sa -o name | grep provider-helm | sed -e 's|serviceaccount\/|crossplane-system:|g')
kubectl create clusterrolebinding provider-helm-admin-binding --clusterrole cluster-admin --serviceaccount="${SA}"

kubectl apply -f provider-helm/providerconfig.yaml
```
### provider-argocd
Install [provider-argocd](https://github.com/crossplane-contrib/provider-argocd) to register the `vcluster` at ArgoCD:
```bash
./kubectl-crossplane install provider crossplane/provider-argocd:v0.1.0
kubectl apply -f provider-argocd/providerconfig.yaml
```

### provider-kubernetes
Install [provider-kubernetes](https://github.com/crossplane-contrib/provider-kubernetes) to create an ArgoCD `application`:
```bash
./kubectl-crossplane install provider crossplane/provider-kubernetes:main

SA=$(kubectl -n crossplane-system get sa -o name | grep provider-kubernetes | sed -e 's|serviceaccount\/|crossplane-system:|g')
kubectl create clusterrolebinding provider-kubernetes-admin-binding --clusterrole cluster-admin --serviceaccount="${SA}"
kubectl apply -f provider-kubernetes/providerconfig.yaml 
```

## Kubernetes in Kubernetes

NOTE: To get the CIDR range, we can execute:

```bash
$ kubectl create service clusterip test --clusterip 1.1.1.1 --tcp=80:80

error: failed to create ClusterIP service: Service "test" is invalid: spec.clusterIPs: Invalid value: []string{"1.1.1.1"}: failed to allocate IP 1.1.1.1: the provided IP (1.1.1.1) is not in the valid range. The range of valid IPs is 10.43.0.0/16

```

The valid IP range will be displayed in the command output. Here it is `10.43.0.0/16`

Edit file crossplane/composition.yaml:
```
...
              vcluster:
                # image: rancher/k3s:v1.19.5-k3s2 
                extraArgs:
                  - --service-cidr=10.43.0.0/16
...
```


Now that we have everything in place we can get started and schedule some Kubernetes clusters in our host cluster.

First create the composition and the corresponding composite resource definition:
```bash
kubectl apply -f crossplane/composition.yaml
kubectl apply -f crossplane/xrd.yaml
```
And now the actual clusters:
```bash
kubectl apply -f crossplane/xrc.yaml
```
They will show up in the `default` namespace:
```bash
d$ vcluster list

 NAME              NAMESPACE         STATUS    CONNECTED   CREATED                          AGE     CONTEXT            
 cluster-2-gqcxj   cluster-2-gqcxj   Running               2023-07-03 10:05:15 +0300 EEST   5m16s   k3d-argo-vcluster  
 cluster-1-5h6xt   cluster-1-5h6xt   Running               2023-07-03 10:05:15 +0300 EEST   5m16s   k3d-argo-vcluster  

```
For each a respective cluster-scoped resource is created:
```bash
$ kubectl get xcluster
NAME              SYNCED   READY   COMPOSITION                AGE
cluster-1-5h6xt   True     False   xcluster.cnp.example.org   5m57s
cluster-2-gqcxj   True     False   xcluster.cnp.example.org   5m57s

```
And finally there should be new `namespaces` in the host cluster, with pods running `vcluster` and the ArgoCD guestbook application.
At ArgoCD, there should be two Kubernetes clusters and to each a corresponding guestbook application (see screenshot at the top).
```
$ kubectl get all -n cluster-1-5h6xt
NAME                                                          READY   STATUS    RESTARTS   AGE
pod/cluster-1-5h6xt-0                                         2/2     Running   0          6m44s
pod/coredns-85cb69466-x452l-x-kube-system-x-cluster-1-5h6xt   1/1     Running   0          5m39s

NAME                                                      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                  AGE
service/cluster-1-5h6xt-headless                          ClusterIP   None           <none>        443/TCP                  6m44s
service/cluster-1-5h6xt                                   ClusterIP   10.43.92.231   <none>        443/TCP                  6m44s
service/cluster-1-5h6xt-node-k3d-argo-vcluster-server-0   ClusterIP   10.43.37.204   <none>        10250/TCP                5m39s
service/kube-dns-x-kube-system-x-cluster-1-5h6xt          ClusterIP   10.43.108.87   <none>        53/UDP,53/TCP,9153/TCP   5m40s

NAME                               READY   AGE
statefulset.apps/cluster-1-5h6xt   1/1     6m45s

```
Note: ssl passthrough has to be enabled for argocd grpc to work. The configuration provided for split ingress in argocd documentation doesn't work. UI login is successfull. However cli login doesn't work -> Argo CLI : `helm upgrade ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace --set controller.publishService.enabled=true --set controller.extraArgs.enable-ssl-passthrough=true` # or just make port forwarding (kubectl port-forward svc/argocd-server -n argocd 8080:443), then: $ argocd login localhost:8080 --username admin --password


