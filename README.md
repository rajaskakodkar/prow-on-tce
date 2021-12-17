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

Edit starter.yaml to include github App private key, id, hmac token and your domain name.

```
kubectl apply -f starter.yaml
```

Prow should now be accessible on https://prow.yourdomain.com

## Job configs

Refer to https://github.com/vagator/test-infra for experimental prow configs and https://github.com/vagator/test-prow for prow experiments!

## References

https://github.com/kubernetes/test-infra/blob/master/prow/getting_started_deploy.md

https://github.com/kubernetes/test-infra/blob/master/prow/cmd/deck/github_oauth_setup.md

https://github.com/kubernetes/test-infra/blob/master/config/prow/config.yaml

https://github.com/kubernetes/test-infra/blob/master/config/prow/plugins.yaml

https://github.com/kubernetes/test-infra/blob/master/config/prow/cluster/starter/starter-gcs.yaml
