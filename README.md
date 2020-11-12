# Hawtio Online

An Hawtio console that eases the discovery and management of _hawtio-enabled_ <sup>[1](#f1)</sup> applications deployed on OpenShift.

<p align="center">
  <img align="center" src="docs/overview.gif">
</p>

## Table of Contents

  * [Deployment](#deployment)
  * [OpenShift 4](#openshift-4)
     * [Manual steps](#manual-steps)
  * [RBAC](#rbac)
     * [Configuration](#configuration)
     * [Roles](#roles)
     * [ACL](#acl)
     * [Authorization](#authorization)
  * [Development](#development)
     * [Tools](#tools)
     * [Build](#build)
     * [Install](#install)
        * [Cluster mode](#cluster-mode)
        * [Namespace mode](#namespace-mode)
     * [Run](#run)
        * [Cluster mode](#cluster-mode-1)
        * [Namespace mode](#namespace-mode-1)
        * [Disable Jolokia authentication for deployments (dev only)](#disable-jolokia-authentication-for-deployments-dev-only)

## Deployment

You can run the following instructions to deploy the Hawtio Online console on your OpenShift cluster.
You may want to read how to [get started with the CLI](https://docs.openshift.com/container-platform/latest/cli_reference/openshift_cli/getting-started-cli.html) for more information about the `oc` client tool.

There exist different OpenShift templates to choose from, depending on the following characteristics:

| Template | Descripton |
| -------- | ---------- |
| [deployment-cluster.yml](https://raw.githubusercontent.com/hawtio/hawtio-online/master/deployment-cluster.yml) | Use an OAuth client that requires the `cluster-admin` role to be created. The Hawtio Online console can discover and connect to _hawtio-enabled_ <sup>[1](#f1)</sup> applications deployed across multiple namespaces / projects. |
| [deployment-cluster-os4.yml](https://raw.githubusercontent.com/hawtio/hawtio-online/master/deployment-cluster-os4.yml) | Same as `deployment-cluster.yml`, to be used for OpenShift 4. By default, this requires the generation of a client certificate, signed with the [service signing certificate][service-signing-certificate] authority, prior to the deployment. See the [OpenShift 4](#openshift-4) section for more information. |
| [deployment-namespace.yml](https://raw.githubusercontent.com/hawtio/hawtio-online/master/deployment-namespace.yml) | Use a service account as OAuth client, which only requires `admin` role in a project to be created. This restricts the Hawtio Online console access to this single project, and as such acts as a single tenant deployment. |
| [deployment-namespace-os4.yml](https://raw.githubusercontent.com/hawtio/hawtio-online/master/deployment-namespace-os4.yml) | Same as `deployment-namespace.yml`, to be used for OpenShift 4. By default, this requires the generation of a client certificate, signed with the [service signing certificate][service-signing-certificate] authority, prior to the deployment. See the [OpenShift 4](#openshift-4) section for more information. |
| [deployment-namespace-rbac.yml](https://raw.githubusercontent.com/hawtio/hawtio-online/master/deployment-namespace-rbac.yml) | Same as `deployment-namespace-os4.yml`, with configurable RBAC for Jolokia requests authorization. See the [RBAC](#rbac) section for more information. |
| [deployment-cluster-rbac.yml](https://raw.githubusercontent.com/hawtio/hawtio-online/master/deployment-cluster-rbac.yml) | Same as `deployment-cluster-os4.yml`, with configurable RBAC for Jolokia requests authorization. See the [RBAC](#rbac) section for more information. |

[service-signing-certificate]: https://docs.openshift.com/container-platform/latest/security/certificates/service-serving-certificate.html

To deploy the Hawtio Online console, execute the following command:

```sh
$ oc new-app -f https://raw.githubusercontent.com/hawtio/hawtio-online/master/deployment-namespace.yml \
  -p ROUTE_HOSTNAME=<HOST>
```

Note that the `ROUTE_HOSTNAME` parameter can be omitted when using the `deployment-namespace` template.
In that case, OpenShift automatically generates one for you.

You can obtain more information about the template parameters, by executing the following command:

```sh
$ oc process --parameters -f https://raw.githubusercontent.com/hawtio/hawtio-online/master/deployment-namespace.yml
NAME                DESCRIPTION                                                                   GENERATOR           VALUE
ROUTE_HOSTNAME      The externally-reachable host name that routes to the Hawtio Online service
```

You can obtain the status of your deployment, by running:

```sh
$ oc status
In project hawtio on server https://192.168.64.12:8443

https://hawtio-online-hawtio.192.168.64.12.nip.io (redirects) (svc/hawtio-online)
  dc/hawtio-online deploys istag/hawtio-online:latest 
    deployment #1 deployed 2 minutes ago - 1 pod
```

Open the route URL displayed above from your Web browser to access the Hawtio Online console.

## OpenShift 4

To secure the communication between Hawtio Online and the Jolokia agents, a client certificate must be generated and mounted into the Hawtio Online pod with a secret, to be used for TLS client authentication. This client certificate must be signed using the [service signing certificate][service-signing-certificate] authority private key.

Prior to the deployment, run the following script to generate and set up a client certificate for Hawtio Online:

```sh
$ ./scripts/generate-certificate.sh
```

Or if you have Yarn intalled, this will also do the same thing:

```sh
$ yarn generate-certificate
```

### Manual steps

Instead of running the script, you can choose to perform everything manually. Here are the steps to be performed:

1. First, retrieve the service signing certificate authority keys, by executing the following commmands as a _cluster-admin_ user:
    ```sh
    # The CA certificate
    $ oc get secrets/signing-key -n openshift-service-ca -o "jsonpath={.data['tls\.crt']}" | base64 --decode > ca.crt
    # The CA private key
    $ oc get secrets/signing-key -n openshift-service-ca -o "jsonpath={.data['tls\.key']}" | base64 --decode > ca.key
    ```

2. Then, generate the client certificate, as documented in [Kubernetes certificates administration](https://kubernetes.io/docs/concepts/cluster-administration/certificates/), using either `easyrsa`, `openssl`, or `cfssl`, e.g., using `openssl`:
    ```sh
    # Generate the private key
    $ openssl genrsa -out server.key 2048
    # Write the CSR config file
    $ cat <<EOT >> csr.conf
    [ req ]
    default_bits = 2048
    prompt = no
    default_md = sha256
    distinguished_name = dn
    
    [ dn ]
    CN = hawtio-online.hawtio.svc
    
    [ v3_ext ]
    authorityKeyIdentifier=keyid,issuer:always
    keyUsage=keyEncipherment,dataEncipherment,digitalSignature
    extendedKeyUsage=serverAuth,clientAuth
    EOT
    # Generate the CSR
    $ openssl req -new -key server.key -out server.csr -config csr.conf
    # Issue the signed certificate
    $ openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 10000 -extensions v3_ext -extfile csr.conf
    ```

3. Finally, you can create the secret to be mounted in Hawtio Online, from the generated certificate:
   ```sh
   $ oc create secret tls hawtio-online-tls-proxying --cert server.crt --key server.key
   ```

Note that `CN=hawtio-online.hawtio.svc` must be trusted by the Jolokia agents, for which client certification authentication is enabled. See the `clientPrincipal` parameter from the [Jolokia agent configuration options](https://jolokia.org/reference/html/agents.html#agent-jvm-config).

You can then proceed with the [deployment](#deployment).

## RBAC

### Configuration

The `deployment-cluster-rbac.yml` and `deployment-namespace-rbac.yml` templates create a _ConfigMap_, that contains the configuration file used to define the roles allowed for MBean operations.
This _ConfigMap_ is mounted into the Hawtio Online container, and the `HAWTIO_ONLINE_RBAC_ACL` environment variable is used to pass the configuration file path to the server.
If that environment variable is not set, RBAC support is disabled, and only users granted the `update` verb on the pod resources are authorized to call MBeans operations.

### Roles

For the time being, only the `viewer` and `admin` roles are supported.
Once the current invocation is authenticated, these roles are inferred from the permissions the user impersonating the request is granted for the pod hosting the operation being invoked.

A user that's granted the `update` verb on the pod resource is bound to the `admin` role, i.e.:

```sh
$ oc auth can-i update pods/<pod> --as <user>
yes
```

Else, a user granted the `get` verb on the pod resource is bound the `viewer` role, i.e.:

```sh
$ oc auth can-i get pods/<pod> --as <user>
yes
```

Otherwise the user is not bound any roles, i.e.:

```sh
$ oc auth can-i get pods/<pod> --as <user>
no
```

### ACL

The ACL definition for JMX operations works as follows:

Based on the _ObjectName_ of the JMX MBean, a key composed with the _ObjectName_ domain, optionally followed by the `type` attribute, can be declared, using the convention `<domain>.<type>`.
For example, the `java.lang.Threading` key for the MBean with the _ObjectName_ `java.lang:type=Threading` can be declared.
A more generic key with the domain only can be declared (e.g. `java.lang`).
A `default` top-level key can also be declared.
A key can either be an unordered or ordered map, whose keys can either be string or regexp, and whose values can either be string or array of strings, that represent roles that are allowed to invoke the MBean member.

The default ACL definition can be found in the `hawtio-rbac` _ConfigMap_ from the `deployment-cluster-rbac.yml` and `deployment-namespace-rbac.yml` templates.

### Authorization

The system looks for allowed roles using the following process:

The most specific key is tried first. E.g. for the above example, the `java.lang.Threading` key is looked up first.
If the most specific key does not exist, the domain-only key is looked up, otherwise, the `default` key is looked up.
Using the matching key, the system looks up its map value for:

1. An exact match for the operation invocation, using the operation signature, and the invocation arguments, e.g.:

   `uninstall(java.lang.String)[0]: [] # no roles can perform this operation`

2. A regexp match for the operation invocation, using the operation signature, and the invocation arguments, e.g.:

   `/update\(java\.lang\.String,java\.lang\.String\)\[[1-4]?[0-9],.*\]/: admin`

   Note that, if the value is an ordered map, the iteration order is guaranteed, and the first matching regexp key is selected;

3. An exact match for the operation invocation, using the operation signature, without the invocation arguments, e.g.:

   `delete(java.lang.String): admin`

4. An exact match for the operation invocation, using the operation name, e.g.:

   `dumpStatsAsXml: admin, viewer`

If the key matches the operation invocation, it is used and the process will not look for any other keys. So the most specific key always takes precedence.
Its value is used to match the role that impersonates the request, against the roles that are allowed to invoke the operation.
If the current key does not match, the less specific key is looked up and matched following the steps 1 to 4 above, up until the `default` key.
Otherwise, the operation invocation is denied.

## Development

### Tools

You must have the following tools installed:

* [Node.js](http://nodejs.org)
* [Yarn](https://yarnpkg.com) (version `1.5.1` or higher)
* [gulp](http://gulpjs.com/) (version `4.x`)

### Build

```
$ yarn install
```

### Install

In order to authenticate and obtain OAuth access tokens for the Hawtio console be authorized to watch for _hawtio-enabled_ <sup>[1](#f1)</sup> applications deployed in your cluster, you have to create an OAuth client that matches localhost development URLs.

##### Cluster mode

```sh
$ oc create -f oauthclient.yml
```

See [OAuth Clients](https://docs.openshift.com/container-platform/4.6/authentication/configuring-oauth-clients.html#oauth-default-clients_configuring-oauth-clients) for more information.

##### Namespace mode

```sh
$ oc create -f serviceaccount.yml
```

See [Service Accounts as OAuth Clients](https://docs.openshift.com/container-platform/latest/authentication/using-service-accounts-as-oauth-client.html) for more information.

### Run

##### Cluster mode

```
$ yarn start --master=`oc whoami --show-server` --mode=cluster
```

##### Namespace mode

```
$ yarn start --master=`oc whoami --show-server` --mode=namespace --namespace=`oc project -q`
```

You can access the console at <http://localhost:2772/>.

### Disable Jolokia authentication for deployments (dev only)

In order for a local hawtio-online to detect the hawtio-enabled applications, each application container needs to be configured with the following environment variables:

```
AB_JOLOKIA_AUTH_OPENSHIFT=false
AB_JOLOKIA_PASSWORD_RANDOM=false
AB_JOLOKIA_OPTS=useSslClientAuthentication=false,protocol=https
```

The following script lets you apply the above environment variables to all the deployments with a label `provider=fabric8` in a batch:

```sh
$ ./scripts/disable-jolokia-auth.sh
```

---

<a name="f1">1</a>. Containers with a configured port named `jolokia` and that exposes the [Jolokia](https://jolokia.org) API.
