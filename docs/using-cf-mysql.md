# Application Developer's Guide to Using cf-mysql

## Connecting to cf-mysql

### Binding an App

You can connect apps that are deployed to Cloud Foundry, via `cf push` using the standard instructions on [Delivering Service Credentials to an Application](https://docs.cloudfoundry.org/devguide/services/application-binding.html).

cf-mysql does not offer any arbitrary parameters.

### Service Keys

To connect to cf-mysql from an app which has not been deployed to Cloud Foundry, you can follow the instructions for [creating a service key](https://docs.cloudfoundry.org/devguide/services/service-keys.html).

### Encryption

When using a service key to establish a connection using TLS (aka SSL) you will need the `ca_certificate` from the service key output. See the [service key documentation](https://docs.cloudfoundry.org/devguide/services/service-keys.html#detail) to view the service key JSON.

Save the CA certificate to a file in PEM format. You'll need to replace the "\n" with newlines.

To test, you can use any MySQL client. Connect to the hostname given in the service key output with the given credentials. When using the `mysql` CLI, also specify `--ssl-verify-server-cert --ssl-ca=PATH_TO_CA_CERTIFICATE`.
