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

Copy the Sibboleth files to the local filesystem.

``` shell
docker run -v /tmp/keystone:/mnt ghcr.io/cloudnull/genestack/keystone-shib:2024.1-ubuntu_jammy cp -R /etc/shibboleth /mnt/
```

the resultant should look something like this

| File | Shibboleth file |
| ---- | --------------- |
| attrChecker.html | Use for XXX |
| attribute-map.xml | Use for XXX |
| attribute-policy.xml | Use for XXX |
| bindingTemplate.html | Use for XXX |
| console.logger | Use for XXX |
| discoveryTemplate.html | Use for XXX |
| example-metadata.xml | Use for XXX |
| example-shibboleth2.xml | Use for XXX |
| globalLogout.html | Use for XXX |
| localLogout.html | Use for XXX |
| metadataError.html | Use for XXX |
| native.logger | Use for XXX |
| partialLogout.html | Use for XXX |
| postTemplate.html | Use for XXX |
| protocols.xml | Use for XXX |
| security-policy.xml | Use for XXX |
| sessionError.html | Use for XXX |
| shibboleth2.xml | Use for XXX |
| shibd.logger | Use for XXX |
| sslError.html | Use for XXX |

The SAML identity provider is a little different than the global auth provider. The SAML provider is used to authenticate users against the Rackspace Identity Provider.

Generate the SAML Keys, for N numbers of years. The keys are used to sign the SAML requests and responses.

``` shell
docker run -v /tmp/keystone/shibboleth:/etc/shibboleth ghcr.io/cloudnull/genestack/keystone-shib:2024.1-ubuntu_jammy shib-keygen -y 5
```

First step is to create the identity provider and connect it to a domain.

``` shell
openstack --os-cloud default identity provider create --remote-id rackspace_saml --domain rackspace_cloud_domain rackspace_saml
```

| File | Shibboleth file |
| ---- | --------------- |
| sp-key.pem | Use for XXX |
| sp-cert.pem | Use for XXX |

### Generate the SAML metadata and configure the SAML identity provider

The files will be created and stored in a kubernetes secret named `keystone-shib-etc` and mounted to `/etc/shibboleth` in the `keystone` pod. The files are:

!!! note "Entity ID"

    The `EXAMPLEENTITYID_THAT_SHOULD_BE_REPLACED` is a placeholder and should be replaced with the actual entity ID of your Keystone service.

``` shell
cat << EOF > /tmp/shibboleth2.xml
<SPConfig xmlns="urn:mace:shibboleth:3.0:native:sp:config"
          xmlns:conf="urn:mace:shibboleth:3.0:native:sp:config"
          clockSkew="180">
    <OutOfProcess tranLogFormat="%u|%s|%IDP|%i|%ac|%t|%attr|%n|%b|%E|%S|%SS|%L|%UA|%a" />
    <ApplicationDefaults entityID="EXAMPLEENTITYID_THAT_SHOULD_BE_REPLACED"
                         REMOTE_USER="eppn subject-id pairwise-id persistent-id"
                         cipherSuites="DEFAULT:!EXP:!LOW:!aNULL:!eNULL:!DES:!IDEA:!SEED:!RC4:!3DES:!kRSA:!SSLv2:!SSLv3:!TLSv1:!TLSv1.1">
       <Sessions lifetime="28800"
                 timeout="3600"
                 relayState="ss:mem"
                 checkAddress="false"
                 handlerSSL="true"
                 cookieProps="https"
                 redirectLimit="exact">
            <SSO entityID="EXAMPLEENTITYID_THAT_SHOULD_BE_REPLACED">
              SAML2
            </SSO>
            <Logout>SAML2 Local</Logout>
            <LogoutInitiator type="Admin" Location="/Logout/Admin" acl="127.0.0.1 ::1" />
            <Handler type="MetadataGenerator" Location="/Metadata" signing="false"/>
            <Handler type="Status" Location="/Status" acl="127.0.0.1 ::1"/>
            <Handler type="Session" Location="/Session" showAttributeValues="false"/>
            <Handler type="DiscoveryFeed" Location="/DiscoFeed"/>
        </Sessions>
        <Errors supportContact="root@localhost"
                helpLocation="/about.html"
                styleSheet="/shibboleth-sp/main.css"/>
        <MetadataProvider type="XML" file="keystone-metadata.xml" />
        <AttributeExtractor type="XML" validate="true" reloadChanges="false" path="attribute-map.xml"/>
        <AttributeFilter type="XML" validate="true" path="attribute-policy.xml"/>
        <CredentialResolver type="File" use="signing"
                            key="sp-signing-key.pem"
                            certificate="sp-signing-cert.pem"/>
        <CredentialResolver type="File" use="encryption"
                            key="sp-encrypt-key.pem"
                            certificate="sp-encrypt-cert.pem"/>
    </ApplicationDefaults>
    <SecurityPolicyProvider type="XML" validate="true" path="security-policy.xml"/>
    <ProtocolProvider type="XML" validate="true" reloadChanges="false" path="protocols.xml"/>
</SPConfig>
EOF
```

Store the SAML metadata and configure the SAML identity provider at `/tmp/keystone/shibboleth/keystone-metadata.xml`.

Ingest all files into the kubernetes secret as a secrete, which will be mounted to the keystone pod.

``` shell
kubectl -n openstack create secret generic keystone-shib-etc \
        --from-file=/tmp/keystone/shibboleth/attrChecker.html \
        --from-file=/tmp/keystone/shibboleth/attribute-map.xml \
        --from-file=/tmp/keystone/shibboleth/attribute-policy.xml \
        --from-file=/tmp/keystone/shibboleth/bindingTemplate.html \
        --from-file=/tmp/keystone/shibboleth/console.logger \
        --from-file=/tmp/keystone/shibboleth/discoveryTemplate.html \
        --from-file=/tmp/keystone/shibboleth/example-metadata.xml \
        --from-file=/tmp/keystone/shibboleth/example-shibboleth2.xml \
        --from-file=/tmp/keystone/shibboleth/globalLogout.html \
        --from-file=/tmp/keystone/shibboleth/localLogout.html \
        --from-file=/tmp/keystone/shibboleth/metadataError.html \
        --from-file=/tmp/keystone/shibboleth/native.logger \
        --from-file=/tmp/keystone/shibboleth/partialLogout.html \
        --from-file=/tmp/keystone/shibboleth/postTemplate.html \
        --from-file=/tmp/keystone/shibboleth/protocols.xml \
        --from-file=/tmp/keystone/shibboleth/security-policy.xml \
        --from-file=/tmp/keystone/shibboleth/sessionError.html \
        --from-file=/tmp/keystone/shibboleth/shibboleth2.xml \
        --from-file=/tmp/keystone/shibboleth/shibd.logger \
        --from-file=/tmp/keystone/shibboleth/sp-cert.pem \
        --from-file=/tmp/keystone/shibboleth/sp-key.pem \
        --from-file=/tmp/keystone/shibboleth/sslError.html
```
