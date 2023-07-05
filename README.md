# Openshift Toolkit for Multi-cluster Deployment with GitOps

The aim of this project is to define the technical tools and content (Platform, Manifests & presentations) needed to deliver the "OPP" (Red Hat OpenShift Platform Plus) offering in GitOps mode.

## Install Management Hub Cluster via Red Hat Demo Platform
* https://demo.redhat.com/catalog?item=babylon-catalog-prod/sandboxes-gpte.sandbox-ocp.prod

## Copy your Master Key to encrypt Secret via SealedSecret

```shell
export BASTION=lab-user@bastion.{uid}.sandbox{uid}.opentlc.com
scp $HOME/.ssh/sealed-secrets-from-gitops.* $BASTION:/home/lab-user
```
... sealed-secrets-from-gitops.key

... sealed-secrets-from-gitops.crt


## Connect to Bastion and generate SealedSecret for Cloud provider AWS

```shell
ssh $BASTION
```

## On Git, update cluster name folder
* Search and Repplace sandbox{uid-old} by sandbox{uid-new}

## On Git, update conf.yaml and provision.yaml
* Search and Repplace sandbox{uid-old} by sandbox{uid-new}

## On Git, update repoURL with your own repo for all application and applicationset.
* /clusters/acm-hub.redhat.com/applications/app-argocd.yaml
* /clusters/acm-hub.redhat.com/applications/cluster-config.yaml
* /clusters/acm-hub.redhat.com/applications/cluster-config-overlays.yaml
* /clusters/acm-hub.redhat.com/applications/cluster-provisioning.yaml

## On Git update sealedsecret-*.yaml
* Generate SealedSecret for aws credential

```shell

## Install tools and Cli

```shell
wget https://github.com/mikefarah/yq/releases/download/v4.34.1/yq_linux_amd64
mv yq_linux_amd64 yq
chmod +x yq
sudo mv yq /usr/bin/yq

wget https://mirror.openshift.com/pub/openshift-v4/clients/helm/latest/helm-linux-amd64
mv helm-linux-amd64 helm
chmod +x helm
sudo mv helm /usr/bin/helm
helm version

version=0.19.1
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v${version}/kubeseal-${version}-linux-amd64.tar.gz
tar xfz kubeseal-${version}-linux-amd64.tar.gz
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
kubeseal --version
```

* Encrypt with sealed-Secret your aws-creds, Pull-secret and private-key

```shell
SECRET=aws-creds-secret

cat << EOF >xxx-aws-creds-secret.yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: xxx-aws-creds
  namespace: xxx
stringData:
  aws_access_key_id: xxxx
  aws_secret_access_key: xxxx
EOF

kubeseal --cert "./${PUBLICKEY}" --scope cluster-wide < xxx-aws-creds-secret.yaml >xxx-aws-creds-sealed-secret.json
yq  xxx-aws-creds-sealed-secret.json -oy > xxx-aws-creds-sealed-secret.yaml
cat xxx-aws-creds-sealed-secret.yaml

SECRET=pull-secret

cat << EOF >${SECRET}.yaml
apiVersion: v1
kind: Secret
metadata:
  name: ${SECRET}
  annotations:
    helm.sh/hook-weight: "355"
  namespace: xxx
stringData:
  .dockerconfigjson: xxx
type: kubernetes.io/dockerconfigjson
EOF

kubeseal --cert "./${PUBLICKEY}" --scope cluster-wide < ${SECRET}.yaml >${SECRET}.json
yq  ${SECRET}.json -oy > ${SECRET}.yaml
cat ${SECRET}.yaml

cat << EOF >xxx-ssh-private-key.yaml
apiVersion: v1
kind: Secret
metadata:
  name: xxx-ssh-private-key
  namespace: xxx
stringData:
  ssh-privatekey: |-
    -----BEGIN OPENSSH PRIVATE KEY-----
    xxx
    -----END OPENSSH PRIVATE KEY-----
type: Opaque
EOF

kubeseal --cert "./${PUBLICKEY}" --scope cluster-wide < xxx-ssh-private-key.yaml >xxx-ssh-private-key-sealed-secret.json
yq  xxx-ssh-private-key-sealed-secret.json -oy > xxx-ssh-private-key-sealed-secret.yaml
cat xxx-ssh-private-key-sealed-secret.yaml
```

* in folder /base/provision/openshift-provisionning/templates
 * Update spec encryptedData in sealedsecret-aws.yaml
 * Update spec encryptedData in sealedsecret-pull-secret.yaml
 * Update spec encryptedData in sealedsecret-ssh-private-key.yaml

```shell
  encryptedData:
    aws_access_key_id: xxx
    aws_secret_access_key: xxx
```

## Update sealed-secrets operator primary key

```shell
export NAMESPACE="sealed-secrets"
export PRIVATEKEY="sealed-secrets-from-gitops.key"
export PUBLICKEY="sealed-secrets-from-gitops.crt"
export SECRETNAME="sealed-secrets-from-gitops"

oc new-project "$NAMESPACE"

kubectl -n "$NAMESPACE" create secret tls "$SECRETNAME" --cert="$PUBLICKEY" --key="$PRIVATEKEY"
kubectl -n "$NAMESPACE" label secret "$SECRETNAME" sealedsecrets.bitnami.com/sealed-secrets-key=active
```

## Bootstrap RH ACM Cluster

```shell
until oc apply -k https://gitlab.consulting.redhat.com/lcolagio/openshift-toolkit-multicluster/bootstrap/overlays/default/; do sleep 5; done
```

## To test the deployment of a managed cluster, copy a template from: /tmp/{env}/ to /clusters/{env}/:
* move /tmp/{env}/ocp{n}.sandbox{uid}.opentlc.com to /clusters/{env}
