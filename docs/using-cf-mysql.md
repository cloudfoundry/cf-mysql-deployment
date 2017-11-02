# Application Developer's Guide to Using cf-mysql

## Connecting to cf-mysql

### Binding an App

You can connect apps that are deployed to Cloud Foundry, via `cf push` using the standard instructions on [Delivering Service Credentials to an Application](https://docs.cloudfoundry.org/devguide/services/application-binding.html).

cf-mysql does not offer any arbitrary parameters.

### Service Keys

To connect to cf-mysql from an app which has not been deployed to Cloud Foundry, you can follow the instructions for [creating a service key](https://docs.cloudfoundry.org/devguide/services/service-keys.html).

### Encryption

#### Applications Running on Cloud Foundry

Most applications, save Java and Spring (see below), can be modified to discover the information necessary to connect to cf-mysql using TLS. When inspecting `VCAP_SERVICES` for username and password, if the additional property, `ca_certificate` is available, your application can connect to cf-mysql using TLS.

Here's a Node.js example:

```node
ca_cert = vcap_services["p-mysql"][0]["credentials"]["ca_certificate"] ;
dbClient = mysql.createConnection( {
    host : host,
    user : user,
    password : password,
    port : port,
    database : database,
    ssl : {
        ca : ca_cert
    },
} ) ;
```
Some languages automatically check the operating system's [trust store](https://docs.cloudfoundry.org/devguide/deploy-apps/trusted-system-certificates.html). In those cases, it is not necessary to parse `VCAP_SERVICES` for the CA certificate.

#### Java and Spring Applications

To enable apps using the [Java buildpack](https://docs.cloudfoundry.org/buildpacks/java/), you'll need to delete the existing binding and create a new one. This will update the `jdbcUrl` to specify an encrypted connection.

**Note:** Should your deployment of cf-mysql turn off encryption in the future, you'll need to **re-bind** all Java applications that use cf-mysql. That will remove the encryption requirement, allowing apps to connect to an instance that does not offer encryption.

#### Service Keys

When using a service key to establish a connection using TLS (aka SSL) you will need the `ca_certificate` from the service key output. See the [service key documentation](https://docs.cloudfoundry.org/devguide/services/service-keys.html#detail) to view the service key JSON.

Save the CA certificate to a file in PEM format. You'll need to replace the "\n" with newlines.

To test, you can use any MySQL client. Connect to the hostname given in the service key output with the given credentials. When using the `mysql` CLI, also specify `--ssl-verify-server-cert --ssl-ca=PATH_TO_CA_CERTIFICATE`.
