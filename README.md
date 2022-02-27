# Prow on Tanzu Community Edition

This is a WIP recipe to deploy Prow on tanzu-community-edition. This experiment has been carried out with tanzu community edition v0.10.0-rc.1.

## Prerequisites

### DNS

Make sure you have a Domain Name where you can add a CNAME for prow.yourdomain.com

### Github App

Follow the instructions to setup Github App for Prow. 

https://github.com/kubernetes/test-infra/blob/master/prow/getting_started_deploy.md#github-app

## Cluster config

### Bootstrap up Tanzu Community Edition Cluster

Follow the steps mentioned in https://tanzucommunityedition.io/docs/latest/getting-started/ to create a Management Cluster and a Workload Cluster for Prow Deployment on AWS.

### Install cert-manager and contour

Contour is the ingress controller and cert-manager is used to provision certificate for tls. Run the following commands to install the packages.

```
tanzu package repository add tce-repo --url projects.registry.vmware.com/tce/main:0.9.1

tanzu package install cert-manager --package-name cert-manager.community.tanzu.vmware.com --version 1.5.3

tanzu package install contour --package-name contour.community.tanzu.vmware.com --version 1.18.1 -f contour-values.yaml
```

### Add cert-manager cluster issuer

Edit cluster-issuer.yaml to include valid email address.

```
kubectl apply -f cluster-issuer.yaml
```

### Cluster Role Bindings

Follow https://github.com/kubernetes/test-infra/blob/master/prow/getting_started_deploy.md#create-cluster-role-bindings to create cluster role bindings.

### Setup before prow deployment

Follow https://github.com/kubernetes/test-infra/blob/master/prow/getting_started_deploy.md#create-the-github-secrets to create Github Secrets.

```
kubectl create ns prow

kubectl create ns test-pods

kubectl create secret -n prow generic hmac-token --from-file=hmac=/path/to/hmac/secret

kubectl create secret -n prow generic github-token --from-file=cert=/path/to/github-app-private-key --from-literal=appid="github-app-id"
```

Follow https://github.com/kubernetes/test-infra/blob/master/prow/cmd/deck/github_oauth_setup.md to setup Github OAuth app needed to merge PRs.

```
kubectl create secret generic github-oauth-config --from-file=secret=/path/to/github-oauth-config -n prow

kubectl create secret -n prow generic cookie --from-file=secret=/path/to/cookie.txt
```

Follow https://github.com/kubernetes/test-infra/blob/master/prow/getting_started_deploy.md#configure-a-gcs-bucket to setup GCS bucket.

### Config maps

Edit config.yaml to modify config of prow components.

Edit plugins.yaml to modify config of prow plugins. 

```
kubectl create configmap plugins --from-file=plugins.yaml=plugins.yaml --dry-run=client -oyaml | kubectl apply -f - -n prow

kubectl create configmap config --from-file=config.yaml=config.yaml --dry-run=client -oyaml | kubectl apply -f - -n prow

kubectl create configmap job-config --from-file=/path/to/prow/job --dry-run=client -oyaml | kubectl apply -f - -n prow
```

## Prow Deployment

```
kubectl apply --server-side=true -f https://raw.githubusercontent.com/kubernetes/test-infra/master/config/prow/cluster/prowjob-crd/prowjob_customresourcedefinition.yaml
```

Edit secrets.yaml to include github App private key, id, hmac token and your domain name.

```
kubectl apply -f secrets.yaml
kubectl apply -f components.yaml
```

Prow should now be accessible on https://prow.yourdomain.com

## Job configs

Refer to https://github.com/vagator/test-infra for experimental prow configs and https://github.com/vagator/test-prow for prow experiments!

## Run test-pods on Build Cluster

Switch kube-context to Management Cluster and create a Workload Cluster.

Run the following command to get the kubeconfig of Workload Cluster

```
tanzu cluster kubeconfig get `CLUSTER_NAME` --admin
```

Switch kube-context to Workload Cluster

Run the following commands to install prowjob CRDs and secrets to GCS bucket

```
kubectl create namespace test-pods
kubectl apply --server-side=true -f https://raw.githubusercontent.com/kubernetes/test-infra/master/config/prow/cluster/prowjob-crd/prowjob_customresourcedefinition.yaml
kubectl -n test-pods create secret generic gcs-credentials --from-file=/path/to/service-account.json
```

Switch kube-context to Prow Service Cluster

Use gencred to create the kubeconfig file (and credentials) for accessing the build and service cluster(s). More details in - https://github.com/kubernetes/test-infra/blob/master/prow/getting_started_deploy.md#run-test-pods-in-different-clusters

```
cd path/to/clone/of/test-infra
go run ./gencred --context="WORKLOAD_CLUSTER_KUBE_CONTEXT" --name=CLUSTER_NAME --output=/path/to/kubeconfig.yaml
```

This should generate kubeconfig.yaml similar to the following yaml from https://github.com/kubernetes/test-infra/tree/master/gencred

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: fake-ca-data
    server: https://1.2.3.4
  name: oldcluster
- cluster:
    certificate-authority-data: fake-ca-data
    server: https://1.2.3.4
  name: newcluster
contexts:
- context:
    cluster: oldcluster
    user: oldcluster
  name: oldcluster
- context:
    cluster: newcluster
    user: newcluster
  name: newcluster
users:
- name: oldcluster
  user:
    client-certificate-data: fake-cert-data
    client-key-data: fake-key-data
- name: newcluster
  user:
    client-certificate-data: fake-cert-data
    client-key-data: fake-key-data
```

Delete existing kubeconfig secret in Prow Service Cluster

```
kubectl delete secret kubeconfig -n prow
```

Create kubeconfig secret in Prow Service Cluster

```
kubectl create secret generic kubeconfig --from-file=config=/path/to/kubeconfig -n prow
```

This should restart `prow-controller-manager`, `crier`, `deck` and `sinker` pods in Prow Service. If they do not restart for some reason, delete these pods to get kubeconfig secret in effect.

Use `cluster: <cluster-alias>` in Prow Job configs to schedule jobs on Build Cluster.

## References

https://github.com/kubernetes/test-infra/blob/master/prow/getting_started_deploy.md

https://github.com/kubernetes/test-infra/blob/master/prow/cmd/deck/github_oauth_setup.md

https://github.com/kubernetes/test-infra/blob/master/config/prow/config.yaml

https://github.com/kubernetes/test-infra/blob/master/config/prow/plugins.yaml

https://github.com/kubernetes/test-infra/blob/master/config/prow/cluster/starter/starter-gcs.yaml
