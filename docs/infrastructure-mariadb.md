# Deploy the MariaDB Operator and a Galera Cluster

## Create secret

``` shell
# MariaDB
kubectl --namespace openstack \
        create secret generic mariadb \
        --type Opaque \
        --from-literal=root-password="$(< /dev/urandom tr -dc _A-Za-z0-9 | head -c${1:-32};echo;)" \
        --from-literal=password="$(< /dev/urandom tr -dc _A-Za-z0-9 | head -c${1:-32};echo;)"
```

## Deploy the mariadb operator

``` shell
cluster_name=`kubectl config view --minify -o jsonpath='{.clusters[0].name}'`
sed -i -e "s/cluster\.local/$cluster_name/" /opt/genestack/base-kustomize/mariadb-operator/kustomization.yaml

test -n "$cluster_name" && kubectl kustomize --enable-helm /opt/genestack/base-kustomize/kustomize/mariadb-operator | \
  kubectl --namespace mariadb-system apply --server-side --force-conflicts -f -
```

!!! info

    The operator may take a minute to get ready, before deploying the Galera cluster, wait until the webhook is online.

``` shell
kubectl --namespace mariadb-system get pods -w
```

## Deploy the MariaDB Cluster

``` shell
kubectl --namespace openstack apply -k /opt/genestack/base-kustomize/mariadb-cluster/base
```

!!! note

    MariaDB has a base configuration which is HA and production ready. If you're deploying on a small cluster the `aio` configuration may better suit the needs of the environment.

## Verify readiness with the following command

``` shell
kubectl --namespace openstack get mariadbs -w
```
