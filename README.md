# tap-postgres-cluster

Repo to demonstrate setting up two postgres instances on a kubernetes cluster that is managed using gitops for usage with Tanzu Application Platform (TAP).

# Pre-Requisites

## fork repo

Need to fork or download this repo and move it into your git provider

## install cluster essentials
https://docs.vmware.com/en/Cluster-Essentials-for-VMware-Tanzu/1.7/cluster-essentials/deploy.html

Note: TKG clusters should have this installed already

You can verify by running the following

```shell
# secret gen controller
kubectl get crd | grep secretgen.carvel.dev
```

it should have output similar to
```
secretexports.secretgen.carvel.dev                       2023-10-26T19:01:44Z
secretimports.secretgen.carvel.dev                       2023-10-26T19:01:45Z
secrettemplates.secretgen.carvel.dev                     2023-10-26T19:01:44Z
```

```shell
# kapp controller
kubectl get po -A | grep kapp 
```

it should have output similar to
```shell
kapp-controller            kapp-controller-777d598f96-72n9c                    2/2     Running   0             44h
```

Note: the namespace might be different, such as `tkg-system`, which is okay.

## Relocate package repositories

### Tanzu Data Services
https://docs.vmware.com/en/VMware-SQL-with-Postgres-for-Kubernetes/2.2/vmware-postgres-k8s/GUID-install-operator.html#relocate-images-to-a-private-registry

### TAP
https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.6/tap/install-online-profile.html#relocate-images-to-a-registry-0

## Create auth secrets
Note: this readme is in the .gitignore to help you not accidentally commit passwords. Do not commit passwords to git.

To manage these secrets, please look at utilizing https://external-secrets.io/latest/ to allow you to commit secret details to git without exposing the actual values.

### create namespace for secrets

```shell
kubectl create ns postgres-install
```

### Create Git repository Secret
Need to create a secret to hold auth details which enable the kapp apps to access the resources located in git.

Docs for format: https://carvel.dev/kapp-controller/docs/v0.48.x/app-overview/#git-authentication

SSH or basic are supported, here is an example for basic auth
```shell
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: postgres-install-creds
  namespace: postgres-install
type: Opaque
stringData:
  username: postgres-install
  password: MY_ACCESS_TOKEN
EOF
```

### Create Image Repository secret
```shell
DOCKER_CONFIG_JSON=$(echo '{"auths":{"harbor.company.com":{"username":"robot$tap","password":"1234dakdjfaf"}}}' | base64)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: registry-credentials
  namespace: postgres-install
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: $DOCKER_CONFIG_JSON
EOF
```

## replace host values in yaml files

### image repository host

We need to update the host values in the following files to match your environment

* [kapp-apps/postgres-deps/tap-package-repository/tanzu-tap-repository.yaml](https://github.com/rabeyta/tap-postgres-clusters/blob/main/kapp-apps/postgres-deps/tap-package-repository/tanzu-tap-repository.yaml#L12)
* [kapp-apps/postgres-deps/tds-package-repository/tanzu-tds-repository.yaml](https://github.com/rabeyta/tap-postgres-clusters/blob/main/kapp-apps/postgres-deps/tds-package-repository/tanzu-tds-repository.yaml#L13)

### git repository host

We need to update the platform automation repo url in the following files (where this repo is stored)

* [kapp-apps/postgres-cluster-bootstrap/postgres-dependency-installer-app.yaml](https://github.com/rabeyta/tap-postgres-clusters/blob/main/kapp-apps/postgres-cluster-bootstrap/postgres-dependency-installer-app.yaml#L16)
* [kapp-apps/postgres-deps/postgres-install-app.yaml](https://github.com/rabeyta/tap-postgres-clusters/blob/main/kapp-apps/postgres-deps/postgres-install-app.yaml#L15)

## Self Signed Certs

If your image registry, such as harbor, is using self-signed certificates, you will need to provide that certificate to `kapp-controller` to enable it to talk to it for pulling images.

You can find the official documentation [here](https://carvel.dev/kapp-controller/docs/v0.48.x/controller-config/).

Here is a snippet of the file you need to create and commit to the git repo.

```shell
mkdir -p ./clusters/postgres/install/kapp-config
cat <<EOF > ./clusters/postgres/install/kapp-config/kapp-controller-config-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  # Name must be `kapp-controller-config` for kapp controller to pick it up
  name: kapp-controller-config

  # Namespace must match the namespace kapp-controller is deployed to. 
  namespace: kapp-controller

stringData:
  # A cert chain of trusted ca certs. These will be added to the system-wide
  # cert pool of trusted ca's (optional)
  caCerts: |
    -----BEGIN CERTIFICATE-----
    MIIEXTCCAsWgA...
    -----END CERTIFICATE-----    
EOF
```

You need to do the following to the snippet:
1. edit the namespace to be where `kapp-controller` is deployed. You can find this by running `kubectl get po -A | grep kapp`
2. replace the example certificate placeholder with your harbor cert
3. commit the file to this git repo under `./clusters/postgres/install/kapp-config/kapp-controller-config-secret.yaml`
4. apply the file manually for now to your cluster (will be managed through gitops for future changes)
   1. `kubectl apply -f ./clusters/postgres/install/kapp-config/kapp-controller-config-secret.yaml`
5. restart `kapp-controller` to have the configuration take effect
   1. `kubectl -n kapp-controller rollout restart deployment kapp-controller`

## commit and push repository changes

Commit and push all the changes performed to this repository to your remote git repository

# Bootstrap cluster
bootstrap gitops on the cluster

```shell
kubectl apply -f kapp-apps/postgres-cluster-bootstrap
```

The manual bootstrap step installs the following on your cluster.
* a namespace to hold our automation (`postgres-install`)
* service account with necessary rbac to perform installs
* a kapp app `postgres-dependency-installer`, located at `kapp-apps/postgres-cluster-bootstrap/postgres-dependency-installer-app.yaml`, that will monitor and reconcile everything under the `kapp-apps/postgres-deps` folder to your cluster this was applied to.

the installed kapp app, `postgres-dependency-installer`, will reconcile the following on your cluster based on what is in git
* tap package repository (location we will install cert-manager from)
* cert-manager (required by vmware postgres operator)
* tds package repository (location we will install postgres operator from)
* vmware postgres operator
* a kapp app `postgres-install`, located at `kapp-apps/postgres-deps/postgres-install-app.yaml`, that will monitor and reconcile everything under the `clusters/postgres/install` folder path to your cluster this was applied to

the installed kapp app, `postgres-install`, will reconcile the following on your cluster based on what is in git
* postgres instances for tap-gui located within the namespace `postgres-tap-gui`
* postgres instances for metadata store within the namespace `postgres-metadata-store`

# monitor app installation

```shell
kubectl -n postgres-install get apps
```

if there are any errors, you can see based on the reconciliation status

a successful install will look like this

```shell
kubectl -n postgres-install get apps

NAME                            DESCRIPTION           SINCE-DEPLOY   AGE
cert-manager                    Reconcile succeeded   4m48s          4m49s
postgres-dependency-installer   Reconcile succeeded   21s            4m59s
postgres-install                Reconcile succeeded   22s            4m4s
```

you will also be able to see your postgres instances in a running state
```shell
k get postgres -A

NAMESPACE                 NAME                STATUS    DB VERSION   BACKUP LOCATION   AGE
postgres-metadata-store   pg-metadata-store   Running   15.4                           10m21s
postgres-tap-gui          pg-tap-gui          Running   15.4                           10m21s
```

To dig into the individual database, go into the namespaces and view the events.

```shell
kubectl -n postgres-metadata-store get events --sort-by=".lastTimestamp"

kubectl -n postgres-tap-gui get events --sort-by=".lastTimestamp"
```

# create dns records

## metadata-store lb

Obtain the load balancer ip
```shell
kubectl -n postgres-metadata-store get svc pg-metadata-store -oyaml | yq -r '.status.loadBalancer.ingress[0].ip'
```

create a dns record pointing to your load balancer to enable a friendly hostname

example.

`metadata-store.pg.tap.example.com`

## tap-gui lb

Obtain the load balancer ip
```shell
kubectl -n postgres-tap-gui get svc pg-tap-gui -oyaml | yq -r '.status.loadBalancer.ingress[0].ip'
```

create a dns record pointing to your load balancer to enable a friendly hostname
`tap-gui.pg.tap.example.com`

# access the db

https://docs.vmware.com/en/VMware-SQL-with-Postgres-for-Kubernetes/2.2/vmware-postgres-k8s/GUID-accessing.html

the username and password details are located in a secret within each namespace.

```shell
kubectl -n postgres-metadata-store get secret pg-metadata-store-db-secret  -oyaml

kubectl -n postgres-tap-gui get secret pg-tap-gui-db-secret  -oyaml
```
