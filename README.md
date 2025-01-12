# GCP GitLab Runner

This [Terraform](https://www.terraform.io/) modules creates a [GitLab CI runner](https://docs.gitlab.com/runner/). 

The runners created by the module use preemptible instances by default for running the builds using the `docker+machine` executor.

- Shared cache in GCS with life cycle management to clear objects after x days.
- Runner agents registered automatically.

The runner supports 2 main scenarios:

### GitLab CI docker-machine runner 

In this scenario the runner agent is running on a GCP Compute Instance and runners are created by [docker machine](https://docs.gitlab.com/runner/configuration/autoscale.html) using preemptible instances. Runners will scale automatically based on the configuration. The module creates a GCS cache by default, which is shared across runners (preemptible instances). 

### GitLab CI docker runner

In this scenario _not_ docker machine is used but docker to schedule the builds. Builds will run on the same compute instance as the agent. 

## Autoscaling
Both docker-machine runner and docker runners autoscale using GCP Custom metrics. The runner publishes running jobs metrics to stackdriver which is then used to scale up/down the number of active runners. `var.runners_min_replicas` and `var.runners_max_replicas` defined variables for the minimum and maximum number of runners respectively. It uses Google Managed Instance Group Autoscaler to scale when the average of running jobs exceeds `var.runners_concurrent - 2`. 

> NOTE: If runners are set to use internal IPs, a Cloud NAT must be deployed for runners to be able to reach internet

### GitLab runner token configuration

The runner is registered on initial deployment. Each new runner registers itself with the same description and tag. To register the runner automatically set the variable `gitlab_runner_registration_config["registration_token"]`. This token value can be found in your GitLab project, group, or global settings. For a generic runner you can find the token in the admin section. By default the runner will be locked to the target project, not run untagged. Below is an example of the configuration map.

```hcl
gitlab_runner_registration_config = {
  registration_token = "Required: <registration token>"
  tag_list           = "Optional: <your tags, comma separated>"
  description        = "Optional: <some description>"
  locked_to_project  = "optional: true"
  run_untagged       = "Optional: false"
  maximum_timeout    = "Optional: 3600"
  access_level       = "Optional: <not_protected OR ref_protected, ref_protected runner will only run on pipelines triggered on protected branches. Defaults to not_protected>"
}
```

### GitLab runner cache

By default the module creates a a cache for the runner in Google Cloud Storage. Old objects are automatically removed via a configurable life cycle policy on the bucket.


## Usage


```hcl
module "runner" {
  source  = "DeimosCloud/gitlab-ci-runner/google"

  network = "default"
  region  = "europe-west1"
  project = local.project_id

  runners_name       = "docker-default"
  runners_gitlab_url = "https://gitlab.com"

  gitlab_runner_registration_config = {
    registration_token = "my-token"
    tag_list           = "docker"
  }

}
```

## Doc generation

Code formatting and documentation for variables and outputs is generated using [pre-commit-terraform hooks](https://github.com/antonbabenko/pre-commit-terraform) which uses [terraform-docs](https://github.com/segmentio/terraform-docs).


And install `terraform-docs` with
```bash
go get github.com/segmentio/terraform-docs
```
or
```bash
brew install terraform-docs.
```

## Contributing

Report issues/questions/feature requests on in the issues section.

Full contributing guidelines are covered [here](CONTRIBUTING.md).



<!-- BEGINNING OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
## Requirements

| Name | Version |
|------|---------|
| terraform | >= 0.12 |

## Providers

| Name | Version |
|------|---------|
| google | n/a |
| random | n/a |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| cache\_bucket\_versioning | Boolean used to enable versioning on the cache bucket, false by default. | `bool` | `false` | no |
| cache\_expiration\_days | Number of days before cache objects expires. | `number` | `2` | no |
| cache\_location | The location where to create the cache bucket in. If not specified, it defaults to the region | `any` | `null` | no |
| cache\_shared | Enables cache sharing between runners. | `bool` | `true` | no |
| cache\_storage\_class | The cache storage class | `string` | `"STANDARD"` | no |
| create\_cache\_bucket | Creates a cache cloud storage bucket if true | `bool` | `true` | no |
| docker\_machine\_disk\_size | The disk size for the docker-machine instances. | `number` | `20` | no |
| docker\_machine\_disk\_type | The disk Type for docker-machine instances. | `string` | `"pd-standard"` | no |
| docker\_machine\_download\_url | Full url pointing to a linux x64 distribution of docker machine. | `string` | `"https://gitlab-docker-machine-downloads.s3.amazonaws.com/main/docker-machine-Linux-x86_64"` | no |
| docker\_machine\_image | A GCP custom image to use for spinning up docker-machines | `string` | `""` | no |
| docker\_machine\_machine\_type | The Machine Type for the docker-machine instances. | `string` | `"f1-micro"` | no |
| docker\_machine\_options | List of additional options for the docker machine config. Each element of this list must be a key=value pair. E.g. '["google-zone=a"]' | `list(string)` | `[]` | no |
| docker\_machine\_preemptible | If true, docker-machine instances will be premptible | `bool` | `false` | no |
| docker\_machine\_tags | Additional Network tags to be attached to the docker-machine instances. | `list(string)` | `[]` | no |
| docker\_machine\_use\_internal\_ip | If true, docker-machine instances will have only internal IPs. | `bool` | `false` | no |
| gitlab\_runner\_registration\_config | Configuration used to register the runner. Available at https://docs.gitlab.com/ee/api/runners.html#register-a-new-runner. | `map` | <pre>{<br>  "access_level": "not_protected",<br>  "description": "",<br>  "locked_to_project": "",<br>  "maximum_timeout": "",<br>  "registration_token": "",<br>  "run_untagged": "",<br>  "tag_list": ""<br>}</pre> | no |
| gitlab\_runner\_version | Version of the GitLab runner. Defaults to latest | `string` | `""` | no |
| labels | Map of labels that will be added to created resources | `map(string)` | `{}` | no |
| network | The target VPC for the docker-machine and runner instances. | `string` | `"default"` | no |
| prefix | The prefix to apply to all GCP resource names (e.g. <prefix>-runner, <prefix>-agent-1). | `string` | `"ci"` | no |
| project | The GCP project to deploy the runner into. | `string` | n/a | yes |
| region | The GCP region to deploy the runner into. | `string` | n/a | yes |
| runner\_additional\_service\_account\_roles | Additional roles to pass to the Runner service account | `list(string)` | `[]` | no |
| runners\_additional\_volumes | Additional volumes that will be used in the runner config.toml, e.g Docker socket | `list(any)` | `[]` | no |
| runners\_allow\_ssh\_access | Enables SSH Access to the runner instances. | `bool` | `true` | no |
| runners\_concurrent | Concurrent value for the runners, will be used in the runner config.toml. Limits how many jobs globally can be run concurrently when running docker-machine. | `number` | `10` | no |
| runners\_disable\_cache | Runners will not use local cache, will be used in the runner config.toml | `bool` | `false` | no |
| runners\_disk\_size | The size of the created gitlab runner instances in GB. | `number` | `20` | no |
| runners\_disk\_type | The Disk type of the gitlab runner instances | `string` | `"pd-standard"` | no |
| runners\_docker\_runtime | docker runtime for runners, will be used in the runner config.toml | `string` | `""` | no |
| runners\_enable\_monitoring | Installs Stackdriver monitoring Agent on runner Instances to collect metrics. | `bool` | `true` | no |
| runners\_environment\_vars | Environment variables during build execution, e.g. KEY=Value, see runner-public example. Will be used in the runner config.toml | `list(string)` | `[]` | no |
| runners\_executor | The executor to use. Currently supports `docker+machine` or `docker`. | `string` | `"docker+machine"` | no |
| runners\_gitlab\_url | URL of the GitLab instance to connect to. | `string` | `"https://gitlab.com"` | no |
| runners\_helper\_image | Overrides the default helper image used to clone repos and upload artifacts, will be used in the runner config.toml | `string` | `""` | no |
| runners\_idle\_count | (docker-machine) Idle count of the runners, will be used in the runner config.toml. | `number` | `0` | no |
| runners\_idle\_time | (docker-machine) Idle time of the runners, will be used in the runner config.toml. | `number` | `600` | no |
| runners\_image | Image to run builds, will be used in the runner config.toml | `string` | `"docker:19.03"` | no |
| runners\_install\_docker\_credential\_gcr | Install docker\_credential\_gcr inside `startup_script_pre_install` script | `bool` | `true` | no |
| runners\_limit | Limit for the runners, will be used in the runner config.toml. | `number` | `0` | no |
| runners\_machine\_autoscaling | (docker-machine) Set autoscaling parameters based on periods, see https://docs.gitlab.com/runner/configuration/advanced-configuration.html#the-runnersmachine-section | <pre>list(object({<br>    periods    = list(string)<br>    idle_count = number<br>    idle_time  = number<br>    timezone   = string<br>  }))</pre> | `[]` | no |
| runners\_machine\_type | Instance type used for the GitLab runner. | `string` | `"n1-standard-1"` | no |
| runners\_max\_builds | (docker-machine) Max builds for each runner after which it will be removed, will be used in the runner config.toml. By default set to 0, no maxBuilds will be set in the configuration. | `number` | `0` | no |
| runners\_max\_growth\_rate | (docker-machine) The maximum number of machines that can be added to the runner in parallel. Default is 0 (no limit). | `number` | `0` | no |
| runners\_max\_replicas | The maximum number of runners to spin up.For docker+machine, this is the max number of instances that will run docker-machine. For docker, this is the maximum number of runner instances. | `number` | `1` | no |
| runners\_metadata | (Optional) Metadata key/value pairs to make available from within instances created from this template. | `map` | `{}` | no |
| runners\_min\_replicas | The minimum number of runners to spin up. For docker+machine, this is the min number of instances that will run docker-machine. For docker, this is the minimum number of runner instances | `number` | `1` | no |
| runners\_name | Name of the runner, will be used in the runner config.toml. | `string` | n/a | yes |
| runners\_output\_limit | Sets the maximum build log size in kilobytes, by default set to 4096 (4MB) | `number` | `4096` | no |
| runners\_post\_build\_script | Commands to be executed on the Runner just after executing the build, but before executing after\_script. | `string` | `"\"\""` | no |
| runners\_pre\_build\_script | Script to execute in the pipeline just before the build, will be used in the runner config.toml | `string` | `"\"\""` | no |
| runners\_pre\_clone\_script | Commands to be executed on the Runner before cloning the Git repository. this can be used to adjust the Git client configuration first, for example. | `string` | `"\"\""` | no |
| runners\_preemptible | If true, runner compute instances will be premptible | `bool` | `true` | no |
| runners\_privileged | Runners will run in privileged mode, will be used in the runner config.toml | `bool` | `true` | no |
| runners\_pull\_policy | pull\_policy for the runners, will be used in the runner config.toml | `string` | `"always"` | no |
| runners\_request\_concurrency | Limit number of concurrent requests for new jobs from GitLab (default 1) | `number` | `1` | no |
| runners\_root\_size | Runner instance root size in GB. | `number` | `16` | no |
| runners\_services\_volumes\_tmpfs | n/a | <pre>list(object({<br>    volume  = string<br>    options = string<br>  }))</pre> | `[]` | no |
| runners\_shm\_size | shm\_size for the runners, will be used in the runner config.toml | `number` | `0` | no |
| runners\_ssh\_allowed\_cidr\_blocks | List of CIDR blocks to allow SSH Access to the gitlab runner instance. | `list(string)` | <pre>[<br>  "0.0.0.0/0"<br>]</pre> | no |
| runners\_tags | Additional Network tags to be attached to the Gitlab Runner. | `list(string)` | `[]` | no |
| runners\_target\_autoscale\_cpu\_utilization | The target CPU utilization that the autoscaler should maintain. If runner CPU utilization gets above this, a new runner is created until runners\_max\_replicas is reached | `number` | `0.9` | no |
| runners\_use\_internal\_ip | Restrict runners to the use of a Internal IP address. NOTE: NAT Gateway must be deployed in your network so that Runners can access resources on the internet | `bool` | `false` | no |
| runners\_volumes\_tmpfs | n/a | <pre>list(object({<br>    volume  = string<br>    options = string<br>  }))</pre> | `[]` | no |
| startup\_script\_post\_install | Startup script snippet to insert after GitLab runner install | `string` | `""` | no |
| startup\_script\_pre\_install | Startup script snippet to insert before GitLab runner install | `string` | `""` | no |
| subnetwork | Subnetwork used for hosting the gitlab-runners. | `string` | `""` | no |

## Outputs

No output.

<!-- END OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
