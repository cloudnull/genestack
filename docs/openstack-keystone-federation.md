# Setup the Keystone Federation Plugins

## Create the Rackspace Domain

``` shell
openstack --os-cloud default domain create rackspace_cloud_domain
```

## Global Auth Identity Provider

First step is to create the identity provider and connect it to a domain.

``` shell
openstack --os-cloud default identity provider create --remote-id rackspace --domain rackspace_cloud_domain rackspace
```

### Create the Global Auth mapping for our identity provider

You're also welcome to generate your own mapping to suit your needs; however, if you want to use the example mapping (which is suitable for production) you can.

??? abstract "Example keystone `mapping.json` file"

    ``` json
    --8<-- "etc/keystone/mapping.json"
    ```

The example mapping **JSON** file can be found within the genestack repository at `/opt/genestack/etc/keystone/mapping.json`.

!!! tip "Creating the `creator` role"

    The creator role does not exist by default, but is included in the example
    mapping. One must create the creator role in order to prevent authentication
    errors if using the mapping "as is".

    ``` shell
    openstack --os-cloud default role create creator
    ```

### Register the Global Auth mapping within Keystone

``` shell
openstack --os-cloud default mapping create --rules /tmp/mapping.json --schema-version 2.0 rackspace_mapping
```

### Create the Global Auth federation protocol

``` shell
openstack --os-cloud default federation protocol create rackspace --mapping rackspace_mapping --identity-provider rackspace
```

### Rackspace Global Auth Configuration Options

The `[rackspace]` section can also be used in your `keystone.conf` to allow you to configure how to anchor on
roles.

| key | value | default |
| --- | ----- | ------- |
| `role_attribute` | A string option used as an anchor to discover roles attributed to a given user | **os_flex** |
| `role_attribute_enforcement` | When set `true` will limit a users project to only the discovered GUID for the defined `role_attribute` | **false** |

## SAML Identity Provider

First step is to create the identity provider and connect it to a domain.

``` shell
openstack --os-cloud default identity provider create --remote-id rackspace_saml --domain rackspace_cloud_domain rackspace_saml
```

### Generate the SAML metadata and configure the SAML identity provider

The files will be created and stored in a kubernetes secret named `keystone-shib-etc` and mounted to `/etc/shibboleth` in the `keystone` pod. The files are:

* `sp-metadata.xml`
* `shibboleth2.xml`
* `attribute-map.xml`

``` shell
cat << EOF >> /tmp/sp-metadata.xml
...
EOF
```

``` shell
cat << EOF > /tmp/shibboleth2.xml
<ApplicationDefaults entityID="https://login.rackspace.com/saml/idp" REMOTE_USER="eppn persistent-id targeted-id">
<SSO entityID="https://login.rackspace.com/saml/idp">
<MetadataProvider type="XML" file="/etc/shibboleth/sp-metadata.xml" />
EOF
```

``` shell
cat << EOF > /tmp/attribute-map.xml
<Attribute name="urn:oid:0.9.2342.19200300.100.1.1" id="uid" />
<ApplicationDefaults entityID="https://login.rackspace.com/saml/idp" REMOTE_USER="uid">
EOF
```

Now ingest all files into the kubernetes secret as a secrete, which will be mounted to the keystone pod.

``` shell
kubectl -n openstack create secret generic keystone-shib-etc \
  --from-file=/tmp/sp-metadata.xml \
  --from-file=/tmp/shibboleth2.xml \
  --from-file=/tmp/attribute-map.xml
```
