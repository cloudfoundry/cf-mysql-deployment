# Cloud Foundry MySQL Bosh Deployment

## Table of contents

[Usage](#usage)

[Deploying](#deploying)

[Security Groups](#security-groups)

[Registering the Service Broker](#registering-broker)

[Smoke Tests](#smoke-tests)

[Deregistering the Service Broker](#deregistering-broker)


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

<a name="usage"></a>
## Usage

### Prerequisites

- A deployment of [BOSH](https://github.com/cloudfoundry/bosh)
- A deployment of [Cloud Foundry](https://github.com/cloudfoundry/cf-release), [final release 193](https://github.com/cloudfoundry/cf-release/tree/v193) or greater
- Instructions for installing BOSH and Cloud Foundry can be found at http://docs.cloudfoundry.org/.
- [Routing release](https://github.com/cloudfoundry-incubator/routing-release)
v0.145.0 or later is required to register the proxy and broker routes with
Cloud Foundry:

    ```bash
    bosh upload release https://bosh.io/d/github.com/cloudfoundry-incubator/cf-routing-release?v=0.145.0
    ```

    Standalone deployments (i.e. deployments that do not interact with Cloud Foundry)
    do not require the routing release.

<a name="upload_stemcell"></a>
### Upload Stemcell

The latest final release expects the Ubuntu Trusty (14.04) go_agent stemcell version [2859](https://github.com/cloudfoundry/bosh/blob/master/CHANGELOG.md#2859) by default. Older stemcells are not recommended. Stemcells can be downloaded from http://bosh.io/stemcells; choose the appropriate stemcell for your infrastructure ([vsphere esxi](https://d26ekeud912fhb.cloudfront.net/bosh-stemcell/vsphere/bosh-stemcell-2859-vsphere-esxi-ubuntu-trusty-go_agent.tgz),  [aws hvm](https://d26ekeud912fhb.cloudfront.net/bosh-stemcell/aws/light-bosh-stemcell-2859-aws-xen-hvm-ubuntu-trusty-go_agent.tgz), or [openstack kvm](https://d26ekeud912fhb.cloudfront.net/bosh-stemcell/openstack/bosh-stemcell-2859-openstack-kvm-ubuntu-trusty-go_agent.tgz)).

<a name="upload_release"></a>
### Upload Release

You can use a pre-built final release or build a dev release from any of the branches described in <a href="https://github.com/cloudfoundry/cf-mysql-release/blob/develop/CONTRIBUTING.md#branches">Getting the Code</a>.

Final releases are stable releases created periodically for completed features. They also contain pre-compiled packages, which makes deployment much faster. To deploy the latest final release, simply check out the **master** branch. This will contain the latest final release and accompanying materials to generate a manifest. If you would like to deploy an earlier final release, use `git checkout <tag>` to obtain both the release and corresponding manifest generation materials. It's important that the manifest generation materials are consistent with the release.

If you'd like to deploy the latest code, build a release yourself from the **develop** branch.

#### Create and upload a BOSH Release:

1. Build the development release.

  ```
  $ cd ~/workspace/cf-mysql-release
  $ git checkout release-candidate
  $ ./scripts/update
  $ bosh2 create-release
  ```

1. Upload the release to your bosh environment:

  ```
  $ bosh2 -e YOUR_ENV upload-release
  ```

<a name="create_infrastructure"></a>
### Create Infrastructure

#### Define subnets

Prior to deployment, the operator should define three subnets via their infrastructure provider.
The MySQL release is designed to be deployed across three subnets to ensure availability in the event of a subnet failure.

#### Create load balancer

In order to route requests to both proxies, the operator should create a load balancer.
Manifest changes required to configure a load balancer can be found in the
[proxy](https://github.com/cloudfoundry/cf-mysql-release/blob/master/docs/proxy.md#configuring-load-balancer) documentation.
Once a load balancer is configured, the brokers will hand out the address of the load balancer rather than the IP of the first proxy.

- **Note:** When using an Elastic Load Balancer (ELB) on Amazon, make sure to create the ELB in the same VPC as your cf-mysql deployment
- **Note:** For all load balancers, take special care to configure health checks to use the health_port of the proxies (default 1936). Do not configure the load balancer health check to use port 3306.

There are two ways to configure a load balancer, either automatically through your IaaS or by supplying static IPs for the proxies

##### For IaaS native load balancers (AWS elb, GCP target_pool, etc)

In order for the MySQL deployment to attach the proxy instances to your configured load balancer, you need to use the [proxy-elb.yml](https://github.com/cloudfoundry/cf-mysql-deployment/blob/develop/operations/proxy-elb.yml) opsfile. This opsfile requires a [vm_extension](https://bosh.io/docs/cloud-config.html#vm-extensions) in your [cloud-config](https://bosh.io/docs/cloud-config.html) which references your load balancer and also defines the specific requirements for your IaaS. You'll need to consult your IaaS documentation as well as your BOSH CPI documentation for the specifics of the `cloud_properties` definitions for use in your `vm_extension`. You can read more specifics about configuration of the proxies [here](https://github.com/cloudfoundry/cf-mysql-release/blob/develop/docs/proxy.md).

##### For custom load balancers (haproxy, f5, etc)

If you would like to use a custom load balancer, you can manually configure your proxies to use static IP addresses which your load balancer can point to. To do that, create an operations file that looks like the following, with static IPs that make sense for your network:
```yaml
- type: replace
  path: /instance_groups/name=proxy/networks
  value:
    - name: default
      static_ips:
      - 10.10.0.1
      - 10.10.0.2
```

<a name="deployment"></a>
## Deploying
### Deployment Components

#### Database nodes

The number of mysql nodes should always be odd, with a minimum count of three, to avoid [split-brain](http://en.wikipedia.org/wiki/Split-brain\_\(computing\)).
When the failed node comes back online, it will automatically rejoin the cluster and sync data from one of the healthy nodes.

The MariaDB cluster nodes are configured by default with 10GB of persistent disk. This can be configured using an operations file to change `instance_groups/name=mysql/persistent_disk` and `properties/cf_mysql/mysql/persistent_disk`, however your deployment will fail if this is less than 3GB.

#### Proxy nodes

There are two proxy instances. The second proxy is intended to be used in a failover capacity.
In the event the first proxy fails, the second proxy will still be able to route requests to the mysql nodes.

#### Broker nodes

There are also two broker instances.
The brokers each register a route with the router, which load balances requests across the brokers.

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
1. The base manifest will not deploy brokers, nor try to register routes for the proxies with a Cloud Foundry router. If you wish to preserve this behavior you will need to include the [add brokers](https://github.com/cloudfoundry/cf-mysql-deployment/tree/master/operations/add-broker.yml) and [register proxy routes](https://github.com/cloudfoundry/cf-mysql-deployment/tree/master/operations/register-proxy-route.yml) operations files.
1. Create custom operations files to map any non-default configuration (e.g. the number of maximum connections).
1. Create a custom operation file to migrate your BOSH 1 `jobs` and static IPs to their new BOSH 2 `instance_groups`. See the section below for [more information](#operations-file-for-migrating-from-bosh-1-style-manifest-to-a-bosh-2-style-manifest).
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

#### Operations file for migrating from BOSH 1 style manifest to a BOSH 2 style manifest
Refer to [these docs](https://bosh.io/docs/migrated-from.html) on migrating from a BOSH 1 style manifest, then create an ops file to mix in those migrations into the base deployment manifest. See below for an example:

```yaml

---
- type: replace
  path: /instance_groups/name=mysql/migrated_from?
  value:
  - name: mysql_z1
    az: z1
  - name: mysql_z2
    az: z2
  - name: mysql_z3
    az: z3

- type: replace
  path: /instance_groups/name=mysql/networks
  value:
  - name: default
    static_ips:
    - 10.10.0.1
    - 10.10.0.2
    - 10.10.0.3
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
[here](http://bosh.io/docs/cli-ops-files.html),
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

### Variables files

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

### Cross-deployment links

By default, this deployment assumes that some variables (e.g. nats) are provided
by cross-deployment links from a deployment named `cf`.
This will be true if Cloud Foundry was deployed via
[cf-deployment](https://github.com/cloudfoundry/cf-deployment).

If you wish to disable cross-deployment links, use the
`disable-cross-deployment-links.yml` operations file.

Disabling cross-deployment links will require these values to be provided
manually (e.g. by passing `-v nats={...}` to the `bosh deploy` command).

<a name="security-groups"></a>
## Security Groups

By default, applications cannot to connect to IP addresses on the private network,
preventing applications from connecting to the MySQL service.
To enable access to the service, create a new security group for the IP
configured in your manifest for the property `jobs.cf-mysql-broker.mysql_node.host`.

Note: This is not required for CF running on bosh-lite, as these application
groups are pre-configured.

1. Add the rule to a file in the following json format; multiple rules are supported.

  ```
  [
   {
     "destination": "10.10.163.1-10.10.163.255",
     "protocol": "all"
    },
   {
     "destination": "10.10.164.1-10.10.164.255",
     "protocol": "all"
    },
   {
     "destination": "10.10.165.1-10.10.165.255",
     "protocol": "all"
    }
 ]
  ```

- Create a security group from the rule file.

  ```shell
  $ cf create-security-group p-mysql rule.json
  ```

- Enable the rule for all apps

  ```shell
  $ cf bind-running-security-group p-mysql
  ```

Security group changes are only applied to new application containers;
existing apps must be restarted.

<a name="registering-broker"></a>
## Registering the Service Broker

After registering the service broker, the MySQL service will be visible in the Services Marketplace; using the [CLI](https://github.com/cloudfoundry/cli), run `cf marketplace`.

### BOSH errand

```
$ bosh2 -e YOUR_ENV -d cf-mysql run-errand broker-registrar
```

### Manually

1. First register the broker using the `cf` CLI.  You must be logged in as an admin.

    ```
    $ cf create-service-broker p-mysql BROKER_USERNAME BROKER_PASSWORD URL
    ```

    `BROKER_USERNAME` and `BROKER_PASSWORD` are the credentials Cloud Foundry will use to authenticate when making API calls to the service broker. Use the values for manifest properties `jobs.cf-mysql-broker.properties.auth_username` and `jobs.cf-mysql-broker.properties.auth_password`.

    `URL` specifies where the Cloud Controller will access the MySQL broker. Use the value of the manifest property `jobs.cf-mysql-broker.properties.external_host`. By default, this value is set to `p-mysql.<properties.domain>` (in spiff: `"p-mysql." .properties.domain`).

    For more information, see [Managing Service Brokers](http://docs.cloudfoundry.org/services/managing-service-brokers.html).

2. Then [make the service plan public](http://docs.cloudfoundry.org/services/managing-service-brokers.html#make-plans-public).


<a name="smoke-tests"></a>
## Smoke Tests

The smoke tests are useful for verifying a deployment.
The MySQL Release contains an "smoke-tests" job which is deployed as a BOSH errand.

### Running Smoke Tests via BOSH errand

Run the smoke tests via bosh errand as follows:

```
$ bosh2 -e YOUR_ENV -d cf-mysql run-errand smoke-tests
```

<a name="deregistering-broker"></a>
## De-registering the Service Broker

The following commands are destructive and are intended to be run in conjuction with deleting your BOSH deployment.
```
$ bosh2 -e YOUR_ENV -d cf-mysql run-errand deregister-and-purge-instances
```

### Manually

Run the following:

```
$ cf purge-service-offering p-mysql
$ cf delete-service-broker p-mysql
```
