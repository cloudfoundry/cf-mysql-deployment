# Cloud Foundry MySQL Bosh Deployment

This repo contains a BOSH 2 manifest that defines tested topologies of cf-mysql-release.

It serves as the reference for the compatible release and stemcell versions.

This repo takes advantage of new features such as:

- [cloud config](https://bosh.io/docs/cloud-config.html)
- [job links](https://bosh.io/docs/links.html)
- [new CLI](https://github.com/cloudfoundry/bosh-cli)
  - The new BOSH CLI must be installed according to the instructions [here](https://bosh.io/docs/cli-v2.html).

Please refer to BOSH documentation for more details. If you're having troubles
with the pre-requisites, please contact the BOSH team for help
(perhaps on [slack](https://slack.cloudfoundry.org/)).

## Usage

### New deployments

New deployments will work "out of the box" with little additional configuration.
There are two mechanisms for providing credentials to the deployment:

- Credentials can be provided with `-l <path-to-vars-file>` (see below for more
information on variable files).
- variables store file should be provided with
`--vars-store <path-to-vars-store-file>` to let the CLI generate secure passwords
and write them to the provided vars store file.

By default the deployment manifest will not deploy brokers, nor try to register
routes for the proxies with a Cloud Foundry router. To enable integration with
Cloud Foundry, operations files are provided to
[add brokers](https://github.com/cloudfoundry/cf-mysql-deployment/operations/tree/master/add-broker.yml)
and
[register proxy routes](https://github.com/cloudfoundry/cf-mysql-deployment/tree/master/operations/register-proxy-route.yml).

If you require static IPs for the proxy instance groups, these IPs should be
added to the `networks` section of the cloud-config as well as to an operations file
which will use these IPs for the proxy instance groups. See below for more
information on operations files.

```sh
bosh \
  -e my-director \
  -d cf-mysql \
  deploy \
  ~/workspace/cf-msyql-deployment/cf-mysql-deployment.yml \
  -o <path-to-operations
```

### Upgrading from previous deployment topologies

If you are upgrading an existing deployment of cf-mysql-release with a manifest
that does not take advantage of these new features, for example if the manifest
was generated via the spiff templates and stubs provided in the cf-mysql-release
repository, then be aware:

1. The base manifest refers to AZs called `z1`, `z2`, and `z3`. If your
cloud-config doesn't have those AZs, it will result in an error.
1. The base manifest will not deploy brokers, nor try to register
routes for the proxies with a Cloud Foundry router. If you wish to preserve this
behavior you will need to include the
[add brokers](https://github.com/cloudfoundry/cf-mysql-deployment/tree/master/operations/add-broker.yml)
and
[register proxy routes](https://github.com/cloudfoundry/cf-mysql-deployment/tree/master/operations/register-proxy-route.yml) operations files.
1. Create custom operations files to map any non-default configuration (e.g.
the number of maximum connections).
1. Create a variables file to contain the credentials of the existing deployment.
 - Using `--vars-store` is not recommended as it will result in credentials being rotated which can cause issues.
1. Run the following command:

```sh
bosh \
  -e my-director \
  -d my-deployment \
  deploy \
  ~/workspace/cf-msyql-deployment/cf-mysql-deployment.yml \
  -o <path-to-deployment-name-operations> \
  [-o <path-to-additional-operations>] \
  -l <path-to-vars-file> \
  [-l <path-to-additional-vars-files>]
```

## Operations files

Supported operations files (e.g. adding a broker for
Cloud Foundry integration) can be found in the
[operations](https://github.com/cloudfoundry/cf-mysql-deployment/tree/master/operations)
directory.

The [manifest template](https://github.com/cloudfoundry/cf-mysql-deployment/tree/master/cf-mysql-deployment.yml)
is not intended to be modified; any changes you need to make should be added to
environment-specific operations files, e.g. `cf-mysql-operations.yml`

Operations files are optional files for modifying the deployment manifest.
They are intended for structural and non-secret changes, e.g. modifying the
`cf_mysql.mysql.max_connections` property. Secret values should be placed in
variables files (see below for more information on variables files).

The syntax for operations files is detailed
[here](https://github.com/cppforlife/go-patch/tree/master/docs/examples.md),
and example operations files can be found
[here](https://github.com/cloudfoundry/cf-mysql-deployment/tree/master/operations).

Operations files can be provided at deploy-time as follows:

```sh
bosh \
  deploy \
  -o <path-to-operations-file>
```

For instance, our template assumes your cloud-config has a persistent-disk type
named "large". If your cloud-config specifies a type "small", you would add the
following to `cf-mysql-operations.yml`:

```yml
---
- type: replace
  path: /instance_groups/name=mysql/persistent_disk_type
  value: small
```

## Variables files

Variables files are a flat-format key-value yaml file which contains sensitive
information such as password, ssl keys/certs etc.

They can be provided at deploy-time as follows:

```sh
bosh \
  deploy \
  -l <path-to-vars-file>
```

We provide a default set of variables intended for a local bosh-lite environment
[here](https://github.com/cloudfoundry/cf-mysql-deployment/tree/master/bosh-lite/default-vars.yml).

Use this as an example for your environment-specific var-file, e.g. `cf-mysql-vars.yml`

## Cross-deployment links

By default, this deployment assumes that some variables (e.g. nats) are provided
by cross-deployment links from a deployment named `cf`.
This will be true if Cloud Foundry was deployed via
[cf-deployment](https://github.com/cloudfoundry/cf-deployment).

If you wish to disable cross-deployment links, use the
`disable-cross-deployment-links.yml` operations file.

Disabling cross-deployment links will require these values to be provided
manually (e.g. by passing `-v nats={...}` to the `bosh deploy` command).
