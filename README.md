# Cloud Foundry MySQL Bosh Deployment

# cf-mysql-deployment

This repo contains a BOSH 2 manifest that defines tested topologies of cf-mysql-release.

## Usage

## Getting Started

To follow the instructions below you will need:

* [bosh go CLI](https://github.com/cloudfoundry/bosh-cli): `brew install chendrix/tap/gobosh`
* [cf-mysql-deployment](https://github.com/cloudfoundry/cf-mysql-deployment) repository
* Prepare a [cloud-config](http://bosh.io/docs/cloud-config.html) and update your environment to use it

## Upload Release

```bash
gobosh \
  -e <environment> \
  upload-release https://bosh.io/d/github.com/cloudfoundry/cf-mysql-release
```

## Prepare Variables File

We provide a default set of variables intended for a local bosh-lite environment
[here](https://github.com/cloudfoundry/cf-mysql-deployment/blob/master/bosh-lite/default-vars.yml).

Use this as an example for your environment-specific var-file, e.g. `cf-mysql-vars.yml`

## Prepare Operations File

The [manifest template](https://github.com/cloudfoundry/cf-mysql-deployment/blob/master/cf-mysql-deployment.yml)
is not intended to be modified; any changes you need to make should be added to
environment-specific ops-file, e.g. `cf-mysql-operations.yml`

For instance, our template assumes your cloud-config has a persistent-disk type
named "large". If your cloud-config specifies a type "small", you would add the
following to `cf-mysql-operations.yml`:

```yml
---
- type: replace
  path: /instance_groups/name=mysql/persistent_disk_type
  value: small
```

## Deploy

```bash
gobosh \
  -e <environment> \
  deploy \
  "${CF_MYSQL_RELEASE_DIR}/manifest-generation/cf-mysql-template-v2.yml" \
  -l "cf-mysql-vars.yml" \
  -o "cf-mysql-operations.yml"
```

### Validated operations

In addition to the deployment topology described in the base manifest, other
deployment toplogies are suppored via provided overrides files, listed below.

#### Providing database as a service to Cloud Foundry

To integrate this with Cloud Foundry via the
[Service Broker API](https://docs.cloudfoundry.org/services/api.html),
include the `add-broker.yml` and `register-proxy-route.yml`.

```
$ bosh \
  -e <environment> \
  -o operations/add-broker.yml \
  -o operations/register-proxy-route.yml \
  -l path-to-custom-vars-file.yml
```

