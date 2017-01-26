# Cloud Foundry MySQL Bosh Deployment

# cf-mysql-deployment

This repo contains a BOSH 2 manifest that defines tested topologies of cf-mysql-release.

## Usage

### Requirements
To follow the instructions listed below, you will need:
- [BOSH CLI v2](https://bosh.io/docs/cli-v2.html)

### Deploy to bosh-lite

```
$ bosh \
  -e <environment> \
  -d cf-mysql deploy cf-mysql-deployment.yml \
  -l bosh-lite/default-vars.yml
```

### Optional Configuration

#### Providing database as a service to Cloud Foundry

To integrate this with Cloud Foundry via the
[Service Broker API](https://docs.cloudfoundry.org/services/api.html),
include the `add-broker.yml` and `register-proxy-route.yml`.

```
$ bosh \
  -e <environment> \
  -o operations/add-broker.yml \
  -o operations/register-proxy-route.yml \
  -l bosh-lite/default-vars.yml
```

