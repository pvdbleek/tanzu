## Pre-reqs
This guide assumes the following:

* MacOS with 16Gb of RAM or more (and plenty of CPU).
* Homebrew installed
* Downloaded Tanzu CLI package available from [https://network.tanzu.vmware.com/products/tanzu-application-platform/](https://docs.vmware.com/en/Tanzu-Application-Platform/1.0/tap/GUID-install-general.html)
* Downloaded Tanzu Cluster Essentials package available from [https://network.tanzu.vmware.com/products/tanzu-cluster-essentials/](https://docs.vmware.com/en/Tanzu-Application-Platform/1.0/tap/GUID-install-general.html)
* Accepted all EULAs as described here [https://docs.vmware.com/en/Tanzu-Application-Platform/1.0/tap/GUID-install-general.html](https://docs.vmware.com/en/Tanzu-Application-Platform/1.0/tap/GUID-install-general.html)


## minikube setup

```
brew install minikube kubectl
minikube config set driver hyperkit
minikube config set memory 8192M
minikube config set cpus 8
minikube start
```

## prep minikube cluster

```
mkdir $HOME/tanzu-cluster-essentials
tar -xvf tanzu-cluster-essentials-darwin-amd64-1.0.0.tgz -C $HOME/tanzu-cluster-essentials

export INSTALL_BUNDLE=registry.tanzu.vmware.com/tanzu-cluster-essentials/cluster-essentials-bundle@sha256:82dfaf70656b54dcba0d4def85ccae1578ff27054e7533d08320244af7fb0343
export INSTALL_REGISTRY_HOSTNAME=registry.tanzu.vmware.com
export INSTALL_REGISTRY_USERNAME=TANZU-NET-USER
export INSTALL_REGISTRY_PASSWORD=TANZU-NET-PASSWORD

cd $HOME/tanzu-cluster-essentials
./install.sh

sudo cp $HOME/tanzu-cluster-essentials/kapp /usr/local/bin/kapp
```

## install tanzu cli

```
mkdir $HOME/tanzu
tar -xvf tanzu-framework-darwin-amd64.tar -C $HOME/tanzu

export TANZU_CLI_NO_INIT=true

cd $HOME/tanzu
install cli/core/v0.10.0/tanzu-core-darwin_amd64 /usr/local/bin/tanzu

tanzu plugin install --local cli all
```

## prep cluster for TAP install

```
kubectl create ns tap-install

tanzu secret registry add tap-registry \
  --username ${INSTALL_REGISTRY_USERNAME} \
  --password ${INSTALL_REGISTRY_PASSWORD} \
  --server ${INSTALL_REGISTRY_HOSTNAME} \
  --export-to-all-namespaces \
  --yes \
  --namespace tap-install

tanzu package repository add tanzu-tap-repository \
  --url registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:1.0.0 \
  --namespace tap-install
```

## Find your minikube cluster IP

You'll need it for the `tap-values.yml`:

```
minikube ip
```

## TAP Installation

Adjust `tap-values.yml` to reflect your environment and start the installation:

```
tanzu package install tap  \
     -p tap.tanzu.vmware.com \
     -v 1.0.0 \
     --values-file tap-values.yml 
     -n tap-install  

tanzu secret registry add registry-credentials \
     --server https://index.docker.io/v1/ \
     --username <your_dockerhub_username> \
     --password <your_dockerhub_secret> \
     --namespace default

cat <<EOF | kubectl -n default apply -f -

apiVersion: v1
kind: Secret
metadata:
  name: tap-registry
  annotations:
    secretgen.carvel.dev/image-pull-secret: ""
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: e30K

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
secrets:
  - name: registry-credentials
imagePullSecrets:
  - name: registry-credentials
  - name: tap-registry

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: default
rules:
- apiGroups: [source.toolkit.fluxcd.io]
  resources: [gitrepositories]
  verbs: ['*']
- apiGroups: [source.apps.tanzu.vmware.com]
  resources: [imagerepositories]
  verbs: ['*']
- apiGroups: [carto.run]
  resources: [deliverables, runnables]
  verbs: ['*']
- apiGroups: [kpack.io]
  resources: [images]
  verbs: ['*']
- apiGroups: [conventions.apps.tanzu.vmware.com]
  resources: [podintents]
  verbs: ['*']
- apiGroups: [""]
  resources: ['configmaps']
  verbs: ['*']
- apiGroups: [""]
  resources: ['pods']
  verbs: ['list']
- apiGroups: [tekton.dev]
  resources: [taskruns, pipelineruns]
  verbs: ['*']
- apiGroups: [tekton.dev]
  resources: [pipelines]
  verbs: ['list']
- apiGroups: [kappctrl.k14s.io]
  resources: [apps]
  verbs: ['*']
- apiGroups: [serving.knative.dev]
  resources: ['services']
  verbs: ['*']
- apiGroups: [servicebinding.io]
  resources: ['servicebindings']
  verbs: ['*']
- apiGroups: [services.apps.tanzu.vmware.com]
  resources: ['resourceclaims']
  verbs: ['*']
- apiGroups: [scanning.apps.tanzu.vmware.com]
  resources: ['imagescans', 'sourcescans']
  verbs: ['*']

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: default
subjects:
  - kind: ServiceAccount
    name: default

EOF