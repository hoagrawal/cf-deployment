# cf-deployment

**This repo is still a work in progress for certain use cases.
Take a look at <a href='#readiness'>this table</a>
to see if it's recommended that you use it.**


### Table of Contents
* <a href='#purpose'>Purpose</a>
* <a href='#readiness'>Is `cf-deployment` ready to use?</a>
* <a href='#deploying-cf'>Deploying CF</a>
* <a href='#contributing'>Contributing</a>
* <a href='#setup'>Setup and Prerequisites</a>
* <a href='#deploying'>Deploying `cf-deployment`</a>
* <a href='#ops-files'>Ops Files</a>
* <a href='#ci'>CI</a>

## <a name='purpose'></a>Purpose
This repo contains a canonical manifest
for deploying Cloud Foundry without the use of `cf-release`,
relying instead on individual component releases.
It will replace the [manifest generation scripts in cf-release][cf-release-url]
when `cf-release` is deprecated.
It uses several newer features
of the BOSH director and CLI.
Older directors may need to be upgraded
and have their configurations extended
in order to support `cf-deployment`.

`cf-deployment` embodies several opinions
about Cloud Foundry deployment.
It:
- prioritizes readability and meaning to a human operator.
  For instance, only necessary configuration is included.
- emphasizes security and production-readiness by default.
  - bosh's `--vars-store` feature is used
  to generate strong passwords, certs, and keys.
  There are no default credentials, even in bosh-lite.
  - TLS/SSL features are enabled on every job which supports TLS.
- uses three AZs, of which two are used to provide redundancy for most instance groups.
The third is used only for instance groups
that should not have even instance counts,
such as etcd and consul.
- uses Diego natively,
does not support DEAs,
and enables diego-specific features
such as ssh access to apps by default.
- deploys jobs to handle platform data persistence
using the cf-mysql release for databases
and the CAPI release's WebDAV job for blob storage.
- assumes load-balancing will be handled by the IaaS
or an external deployment.
- assumes GCP as the default deployment environment.
For use with other IaaSs, see the **Ops Files** section below.

### <a name='readiness'></a> Is `cf-deployment` ready to use?

| Use Case | Is cf-deployment ready? | Blocked On |
| -------- | ----------------------- | ---------- |
| Test and development | Yes | |
| New production deployments | No | Downtime testing |
| Existing production deployments using cf-release | No | Migration tools |

We've been testing cf-deployment for some time,
and many of the development teams in the Cloud Foundry organization
are using it for development and testing.
If that describes your use case,
you can use cf-deployment as your manifest.

If you're hoping to use cf-deployment for a new _production_ deployment,
we still wouldn't suggest using cf-deployment.
We still need to be able to make some guarantees
about app availability during rolling deploys.
When we think cf-deployment is ready,
we'll update this section and make announcements on the cf-dev mailing list.

**Migrating from cf-release:**
The Release Integration team is still working on developing
a migration path from cf-release.
This use case is not sufficiently tested yet,
and we don't advise anybody to attempt it
until we develop the necessary tooling and guide.

## <a name='deploying-cf'></a>Deploying CF

#### Step 1: Get a BOSH Director
To deploy a BOSH Director to AWS or GCP,
use [`bbl`](https://github.com/cloudfoundry/bosh-bootloader)
(the Bosh BootLoader).
For a full guide to getting set up on GCP, look at [this guide](gcp-deployment-guide.md).

If you're deploying against a local bosh-lite,
you'll need to take the following steps before deploying:
```
export BOSH_CA_CERT=$PWD/bosh-lite/ca/certs/ca.crt
bosh -e 192.168.50.4 update-cloud-config bosh-lite/cloud-config.yml
```

##### Step 1.5: Get load balancers
For IaaSes like AWS and GCP,
you'll need to use `bbl` to create load balancers as well
by running `bbl create-lbs`.

#### Step 2: Deploy CF
To deploy to a configured BOSH director using the new `bosh` CLI:

```
export SYSTEM_DOMAIN=some-domain.that.you.have
bosh -e my-env -d cf deploy cf-deployment/cf-deployment.yml \
  --vars-store env-repo/deployment-vars.yml \
  -v system_domain=$SYSTEM_DOMAIN \
  [ -o operations/CUSTOMIZATION1 ] \
  [ -o operations/CUSTOMIZATION2 (etc.) ]
```

The CF Admin credentials will be stored in the file passed to the `--vars-store` flag
(`env-repo/deployment.yml` in the example).
You can find them by searching for `uaa_scim_users_admin_password`.

If you're using a local bosh-lite,
remember to add the `operations/bosh-lite.yml` ops-file
to your deploy command:
  ```

  bosh -e 192.168.50.4 -d cf deploy cf-deployment.yml \
    -o operations/bosh-lite.yml \
    --vars-store deployment-vars.yml \
    -v system_domain=bosh-lite.com
  ```

See the rest of this document for more on the new CLI, deployment vars, and configuring your BOSH director.

## <a name='contributing'></a>Contributing
Although the default branch for the repository is `master`,
we ask that all pull requests be made against
the `develop` branch.
Please also take a look at the ["style guide"](texts/style-guide.md),
which lays out some guidelines for adding properties or jobs
to the deployment manifest.

We ask that pull requests and other changes be successfully deployed,
and tested with the latest sha of CATs.

## <a name='setup'></a>Setup and Prerequisites
`cf-deployment` relies on newer BOSH features,
and requires a bosh director with a valid cloud-config that has been configured with a certificate authority.
It also requires the new alpha `bosh` CLI,
which it relies on to generate and fill-in needed variables.

### BOSH CLI
`cf-deployment` requires the new [BOSH CLI](https://github.com/cloudfoundry/bosh-cli).
It's in alpha, but has features necessary to use `cf-deployment`.

### BOSH `cloud-config`
`cf-deployment` assumes that
you've uploaded a compatible [cloud-config](http://bosh.io/docs/cloud-config.html) to the BOSH director.
The cloud-config produced by `bbl` is compatible by default,
which covers GCP and AWS.
For bosh-lite, you can use the cloud-config in the `bosh-lite` directory of this repo.
We have not yet tested cf-deployment against other IaaSes,
so you may need to do some engineering work to figure out the right cloud config (and possibly ops files)
to get it working for `cf-deployment`.

### Deployment variables and the var-store
`cf-deployment.yml` requires additional information
to provide environment-specific or sensitive configuration
such as the system domain and various credentials.
To do this we use the `--vars-store` flag in the new BOSH CLI.
This flag takes the name of a `yml` file that it will read and write to.
Where necessary credential values are not present,
it will generate new values based on the type information stored in `cf-deployment.yml`.
Variables passed in with `-v` or `-l` will override those already in the var store,
but will also be stored there for future use.
The `-v` flag is also the recommended mechanism for providing the system domain,
which `bosh` is not equipped to generate.

## <a name='ops-files'></a>Ops Files
The configuration of CF represented by `cf-deployment.yml` is intended to be a workable, secure, fully-featured default.
When the need arises to make different configuration choices,
we accomplish this with the `-o`/`--ops-file` flags.
These flags read a single `.yml` file that details operations to be performed on the manifest
before variables are generated and filled.
We've supplied some common manifest modifications in the `operations` directory.

Here's an (alphabetical) summary:
- `operations/aws.yml` and `operations/change-logging-port-for-aws-elb.yml` -
  this file overrides the vm_extensions for load balancers and overrides the loggregator ports to 4443,
  since it is required under AWS to have a separate port from the standard HTTPS port (443) for loggregator traffic
  in order to use the AWS load balancer.
- `operations/disable-router-tls-termination.yml` -
  this file eliminates keys related to performing tls/ssl termination within the gorouter job.
  It's useful for deployments where tls termination is performed prior to the gorouter -
  for instance, on AWS, such termination is commonly done at the ELB.
  This also eliminates the need to specify `((router_ssl.certificate))` and `((router_ssl.private_key))` in the var files.
- `operations/gcp.yml` -
  this file was intentionally left blank and left for backwards compatibility. It previously overrode the static IP addresses assigned to some instance groups,
  as GCP networking features allow them to all co-exist on the same subnet
  despite being spread across multiple AZs.
- `operations/scale-to-one-az.yml` -
  Scales cf-deployment down to a single instance per instance group,
  placing them all into a single AZ.
  Effectively halves the deployment's footprint.
  Should be applied before other ops files.
- `operations/tcp-routing-gcp.yml` - this ops file adds TCP router and routing api for GCP.
  Not directly compatible with the `use-postgres` ops file;
  see that ops file's entry
  for details.
- `operations/use-postgres.yml` -
  replaces the MySQL instance group
  with a postgres instance group.
  **Warning**: this will lead to total data loss
  if applied to an existing deployment with MySQL
  or removed from an existing deployment with postgres.
  Requires an additional ops file to work with
  the `tcp-routing-gcp` ops file.
  See the next entry for details.
- `use-postgres-tcp-routing` - builds on
  `tcp-routing-gcp` and `use-postgres`
  to use the postgres database for TCP routing.
  Must come after the other two.

## <a name='ci'></a>CI
The [ci](https://release-integration.ci.cf-app.com/teams/main/pipelines/cf-deployment) for `cf-deployment`
automatically bumps to the latest versions of its component releases on the `develop` branch.
These bumps, along with any other changes made to `develop`, are deployed to a single long-running environment
and tested with CATs before being merged to master if CATs goes green.
There is not presently any versioning scheme,
or way to correlate which version of CATs is associated with which sha of cf-deployment,
other than the history in CI.
As `cf-deployment` matures, we'll address versioning.
The configuration for our pipeline can be found [here](https://github.com/cloudfoundry/runtime-ci/pipelines/cf-deployment.yml).

[cf-deployment-concourse-url]: https://release-integration.ci.cf-app.com/teams/main/pipelines/cf-deployment
[cf-release-url]: https://github.com/cloudfoundry/cf-release/tree/master/templates
