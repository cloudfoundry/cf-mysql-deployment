# Operations

This directory contains a list of commonly-used, tested, operations files.

Unless otherwise stated, they can be combined in any permutation and in any order.

### [add-broker.yml](https://github.com/cloudfoundry/cf-mysql-deployment/tree/master/operations/add-broker.yml)

Adds multiple instances of the
[cf-mysql-broker](https://github.com/cloudfoundry/cf-mysql-broker)
to enable integration of the MySQL database with Cloud Foundry via the
[Service Broker API](https://docs.cloudfoundry.org/services/api.html).

Example usage:

```
-o add-broker.yml \

-v cf_admin_username=cf-admin-username \
-v cf_admin_password=cf-admin-password
-v cf_api_url=api.my-cf.com \
-v app_domains="[e.g. myapps.my-cf.com]" \
-v skip_ssl_validation=false
```

### [disable-cross-deployment-links.yml](https://github.com/cloudfoundry/cf-mysql-deployment/tree/master/operations/disable-cross-deployment-links.yml)

Used when the dependent deployments (i.e. Cloud Foundry) do not expose properties
via cross-deployment links.

For example, many configurations of
[cf-release](https://github.com/cloudfoundry/cf-release)
(including the
[provided spiff manifest generation](https://github.com/cloudfoundry/cf-release/blob/master/scripts/generate_deployment_manifest))
do not support cross-deployment links without manual modifications to the manifest,
whereas deploying Cloud Foundry via
[cf-deployment](https://github.com/cloudfoundry/cf-deployment)
exposes properties like NATS config by default.

Using this operations file will require you to provide your own values for these
properties which would otherwise be provided via links, e.g. NATS.

Example usage:

```
-o disable-cross-deployment-links.yml \

-v nats="{password: some-nats-password, user: nats, port: 4222, machines: [10.0.31.191]}"
```

### [mysql-host.yml](https://github.com/cloudfoundry/cf-mysql-deployment/tree/master/operations/mysql-host.yml)

Provides a value for the property `cf_mysql.host` property, which is host the
broker provides to applications via service instance bindings.

Typically this is a FQDN pointing to a load balancer or some other mechanism to
achieve HA (e.g. DNS, floating virtual IPs etc).

Example usage:

```
-o mysql-host.yml \

-v cf_mysql_host=my-load-balancer-url
```
