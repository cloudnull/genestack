# Deploy Cyborg

OpenStack Cyborg is the accelerator offload controller for OpenStack. It enables
the deployment and management of hardware accelerators and offload functions in
an OpenStack cloud, supporting use cases like DPU acceleration, SmartNIC
offloading, and specialized hardware resource management. This document outlines
the deployment of OpenStack Cyborg using Genestack.

## Create secrets

!!! note "Information about the secrets used"

    Manual secret generation is only required if you haven't run the
    `create-secrets.sh` script located in `/opt/genestack/bin`.

    ??? example "Example secret generation"

        ``` shell
        kubectl --namespace openstack \
                create secret generic cyborg-rabbitmq-password \
                --type Opaque \
                --from-literal=username="cyborg" \
                --from-literal=password="$(< /dev/urandom tr -dc _A-Za-z0-9 | head -c${1:-64};echo;)"
        kubectl --namespace openstack \
                create secret generic cyborg-db-password \
                --type Opaque \
                --from-literal=password="$(< /dev/urandom tr -dc _A-Za-z0-9 | head -c${1:-32};echo;)"
        kubectl --namespace openstack \
                create secret generic cyborg-admin \
                --type Opaque \
                --from-literal=password="$(< /dev/urandom tr -dc _A-Za-z0-9 | head -c${1:-32};echo;)"
        ```

## Define policy configuration

!!! note "Information about the default policy rules used"

    The default RabbitMQ policy sets quorum queues target group size to 3 for
    the `cyborg` vhost. This can be changed in `base-kustomize/cyborg/base/policies.yaml`.

    ??? example "Default RabbitMQ policy"

        ``` yaml
        apiVersion: rabbitmq.com/v1beta1
        kind: Policy
        metadata:
          name: cyborg-quorum-three-replicas
          namespace: openstack
        spec:
          name: cyborg-quorum-three-replicas
          vhost: "cyborg"
          pattern: ".*"
          applyTo: queues
          definition:
            target-group-size: 3
          priority: 0
          rabbitmqClusterReference:
            name: rabbitmq
        ```

## Run the package deployment

!!! example "Run the Cyborg deployment Script `/opt/genestack/bin/install-cyborg.sh`"

    ``` shell
    --8<-- "bin/install-cyborg.sh"
    ```

!!! tip

    You may need to provide custom values to configure your OpenStack services.
    For a simple single region or lab deployment you can supply an additional
    overrides flag using the example found at
    `base-helm-configs/aio-example-openstack-overrides.yaml`.

## Validate functionality

``` shell
kubectl --namespace openstack exec -ti openstack-admin-client -- cyborg version
```
