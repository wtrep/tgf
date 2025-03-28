# TGF

<!-- markdownlint-disable MD033 -->

[![Build Status](https://travis-ci.org/coveooss/tgf.svg?branch=master)](https://travis-ci.org/coveooss/tgf)
[![Go Report Card](https://goreportcard.com/badge/github.com/coveooss/tgf)](https://goreportcard.com/report/github.com/coveooss/tgf)
[![Coverage Status](https://coveralls.io/repos/github/coveooss/tgf/badge.svg?branch=master)](https://coveralls.io/github/coveooss/tgf?branch=master)

A **T**erra**g**runt **f**rontend that allows execution of Terragrunt/Terraform through Docker.

Table of content:

* [Description](#description)
* [Configuration](#configuration)
* [Invocation arguments](#tgf-invocation)
* [Images](#default-docker-images)
* [Usage](#usage)
* [Development](#development)

## Description

`TGF` is a small utility used to launch a Docker image and automatically map the current folder, your HOME folder and your current environment
variables to the underlying container.

By default, TGF is used as a frontend for [our fork of terragrunt](https://github.com/coveooss/terragrunt), but it could also be used to run different endpoints.

### Why use TGF

Using `TGF` ensures all your users are using the same set of tools to run infrastructure configuration even if they are working on different environments (`linux`, `Microsoft Windows`, `Mac OSX`, etc).

`Terraform` is very sensitive to the version used and if one user updates to a newer version, the state files will be marked with the latest version and
all other users will have to update their `Terraform` version to the latest used one.

Also, tools such as `AWS CLI` are updated on a regular basis and people don't tend to update their version regularly, resulting in many versions
among your users. If someone makes a script calling a new feature of the `AWS` api, that script may break when executed by another user that has an
outdated version.

## Installation

On `mac` and `linux`:

You can run the `get-latest-tgf.sh` script to check if you have the latest version of tgf installed and install it as needed:

```bash
curl https://raw.githubusercontent.com/coveooss/tgf/master/get-latest-tgf.sh | bash
```

On `Windows`, run `get-latest-tgf.ps1` with Powershell:

```PowerShell
(Invoke-WebRequest https://raw.githubusercontent.com/coveooss/tgf/master/get-latest-tgf.ps1).Content | Invoke-Expression
```

> This will install tgf in your current directory. Make sure to add the executable to your PATH.

## Configuration

TGF has multiple levels of configuration. It first looks through the [AWS parameter store](https://aws.amazon.com/ec2/systems-manager/parameter-store/)
under `/default/tgf` using your current [AWS CLI configuration](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html) if any. There, it tries to find parameters called `config-location` (example: bucket.s3.amazonaws.com/foo) and `config-paths` (example: my-file.json:my-second-file.json, default: TGFConfig). If it finds `config-location`, it fetches its config from that path using the [go-getter library](https://github.com/hashicorp/go-getter). Otherwise, it looks directly in SSM for configuration keys (ex: `/default/tgf/logging-level`).

**Note**: The SSM configuration will only be read if AWS environment variables are set, the AWS CLI is installed or the ~/.aws folder exists. If you wish to force TGF to read the SSM config and these conditions are not met, you can set the `TGF_USE_AWS_CONFIG=true` environment variable

TGF then looks for a file named .tgf.config or tgf.user.config in the current working folder (and recursively in any parent folders) to get its parameters. These configuration files overwrite the remote configurations.
Your configuration file could be expressed in  [YAML](http://www.yaml.org/start.html) or [JSON](http://www.json.org/).

Example of a YAML configuration file:

```yaml
docker-refresh: 1h
logging-level: notice
```

Example of a JSON configuration file:

```json
{
  "docker-refresh": "1h",
  "logging-level": "notice"
}
```

### Configuration keys

Key | Description | Default value
--- | --- | ---
| docker-image | Identify the docker image  to use | coveo/tgf
| docker-image-version | Identify the image version |
| docker-image-tag | Identify the image tag (could specify specialized version such as k8s, full) | latest
| docker-image-build | List of Dockerfile instructions to customize the specified docker image) |
| docker-image-build-folder | Folder where the docker build command should be executed |
| docker-refresh | Delay before checking if a newer version of the docker image is available | 1h (1 hour)
| docker-options | Additional options to supply to the Docker command |
| logging-level | Terragrunt logging level (only applies to Terragrunt entry point).<br>*Critical (0), Error (1), Warning (2), Notice (3), Info (4), Debug (5), Full (6)* | Notice
| entry-point | The program that will be automatically launched when the docker container starts | terragrunt
| tgf-recommended-version | The minimal tgf version recommended in your context  (should not be placed in `.tgf.config file`) | *no default*
| recommended-image | The tgf image recommended in your context (should not be placed in `.tgf.config file`) | *no default*
| environment | Allows temporary addition of environment variables | *no default*
| run-before | Script that is executed before the actual command | *no default*
| run-after | Script that is executed after the actual command | *no default*
| alias | Allows to set short aliases for long commands<br>`my_command: "--ri --with-docker-mount --image=my-image --image-version=my-tag -E my-script.py"` | *no default*
| auto-update | Toggles the auto update check. Will only perform the update after the delay | true
| auto-update-delay | Delay before running auto-update again  | 2h (2 hours)
| update-version | The version to update to when running auto update | Latest fetched from Github's API

Note: *The key names are not case-sensitive*

### Configuration section

It is possible to specify configuration elements that only apply on a specific os.

Example of an HCL configuration file:

```text
docker-refresh: 1h
logging-level: notice
windows:
  logging-level: debug
linux:
  docker-refresh: 2h
```

section | Description
--- | ---
| windows | Configuration that is applied only on Windows systems
| linux | Configuration that is applied only on Linux systems
| darwin | Configuration that is applied only on OSX systems
| ix | Configuration that is applied only on Linux or OSX systems

## TGF Invocation

```text
> tgf -H
usage: tgf [<flags>]

DESCRIPTION:
TGF (terragrunt frontend) is a Docker frontend for terragrunt/terraform. It automatically maps your current folder,
your HOME folder, your TEMP folder as well of most environment variables to the docker process. You can add -D to
your command to get the exact docker command that is generated.

It then looks in your current folder and all its parents to find a file named '.tgf.config' to retrieve the
default configuration. If not all configurable values are satisfied and you have an AWS configuration, it will
then try to retrieve the missing elements from the AWS Parameter Store under the key '/default/tgf'.

Configurable values are:
  - docker-image
  - docker-image-version
  - docker-image-tag
  - docker-image-build
  - docker-image-build-folder
  - docker-image-build-tag
  - logging-level
  - entry-point
  - docker-refresh
  - docker-options
  - recommended-image-version
  - required-image-version
  - tgf-recommended-version
  - environment
  - run-before
  - run-after
  - alias
  - update-version
  - auto-update-delay
  - auto-update

Full documentation can be found at https://github.com/coveooss/tgf/blob/master/README.md
Check for new version at https://github.com/coveooss/tgf/releases/latest.

Any docker image could be used, but TGF specialized images could be found at: https://hub.docker.com/r/coveo/tgf/tags.

Terragrunt documentation could be found at https://github.com/coveo/terragrunt/blob/master/README.md (Coveo fork).

Terraform documentation could be found at https://www.terraform.io/docs/index.html.

ENVIRONMENT VARIABLES:
Most of the arguments can be set through environment variables using the format TGF_ARG_NAME.

Ex:
   TGF_LOCAL_IMAGE=1      ==> --local-image
   TGF_IMAGE_VERSION=2.0  ==> --image-version=2.0

SHORTCUTS:
You can also use shortcuts instead of using the long argument names (first letter of each word).

Ex:
   --li     ==> --local-image
   --iv=2.0 ==> --image-version=2.0

IMPORTANT:
Most of the tgf command line arguments are in uppercase to avoid potential conflict with the underlying command.
If any of the tgf arguments conflicts with an argument of the desired entry point, you must place that argument
after -- to ensure that they are not interpreted by tgf and are passed to the entry point. Any non conflicting
argument will be passed to the entry point wherever it is located on the invocation arguments.

  tgf ls -- -D   # Avoid -D to be interpreted by tgf as --debug

It is also possible to specify additional arguments through environment variable TGF_ARGS.

VERSION: 1.23.1

AUTHOR: Coveo

Flags:
  -H, --help-tgf                Show context-sensitive help (also try --help-man)
      --image=coveo/tgf         Use the specified image instead of the default one
      --image-version=version   Use a different version of docker image instead of the default one
  -T, --tag=latest              Use a different tag of docker image instead of the default one
      --local-image             If set, TGF will not pull the image when refreshing
      --get-image-name          Just return the resulting image name
      --refresh-image           Force a refresh of the docker image
  -E, --entrypoint=terragrunt   Override the entry point for docker
      --current-version         Get current version information
      --all-versions            Get versions of TGF & all others underlying utilities
  -L, --logging-level=<level>   Set the logging level (panic=0, fatal=1, error=2, warning=3, info=4, debug=5, trace=6, full=7)
  -D, --debug                   Print debug messages and docker commands issued
  -F, --flush-cache             Invoke terragrunt with --terragrunt-update-source to flush the cache
      --interactive             ON by default: Launch Docker in interactive mode, use --no-interactive to disable
      --docker-build            ON by default: Enable docker build instructions configured in the config files, use
                                --no-docker-build to disable
      --home                    ON by default: Enable mapping of the home directory, use --no-home to disable
      --temp                    ON by default: Map the temp folder to a local folder (Deprecated: Use --temp-location host and
                                --temp-location none), use --no-temp to disable
      --temp-location=folder    Determine where the temporary work folder 'tgf' inside the docker image is mounted:
                                  volume: Mounts the work folder in the docker volume named “tgf”. The volume is created if it doesn't exist.
                                  host: Mounts the work folder in a directory on the host.
                                  none: The work folder is not mounted and is private to the docker container.
      --mount-point=<folder>    Specify a mount point for the current folder
      --prune                   Remove all previous versions of the targeted image
      --docker-arg=<opt> ...    Supply extra argument to Docker
      --with-current-user       Runs the docker command with the current user, using the --user arg
      --with-docker-mount       Mounts the docker socket to the image so the host's docker api is usable
      --ignore-user-config      Ignore all tgf.user.config files
      --aws                     ON by default: Use AWS Parameter store to get configuration, use --no-aws to disable
  -P, --profile=<AWS profile>   Set the AWS profile configuration to use
      --ssm-path=<path>         Parameter Store path used to find AWS common configuration shared by a team
      --config-files=<files>    Set the files to look for (default: TGFConfig)
      --config-location=<path>  Set the configuration location
      --update                  Run auto update script
```

Example:

```bash
> tgf --current-version
tgf v1.22.0
```

Returns the current version of the tgf tool

```bash
> tgf -- --version
terragrunt version v1.2.0
```

Returns the version of the default entry point (i.e. `Terragrunt`), the --version located after the -- instructs tgf to pass this argument
to the desired entry point

```bash
> tgf -E terraform -- --version
Terraform v0.11.8
```

Returns the version of `Terraform` since we specified the entry point to be terraform.

## Default Docker images

### Base image: coveo/tgf.base (based on Alpine)

* [Terraform](https://www.terraform.io/)
* [Terragrunt](https://github.com/coveo/terragrunt)
* [Go Template](https://github.com/coveooss/gotemplate)
* Shells & tools
  * `sh`
  * `openssl`

### Default image: coveo/tgf (based on Alpine)

All tools included in `coveo/tgf:base` plus:

* [Python](https://www.python.org/) (2 and 3)
* [Ruby](https://www.ruby-lang.org/en/)
* [AWS CLI](https://aws.amazon.com/cli/)
* [jq](https://stedolan.github.io/jq/)
* [Terraforming](https://github.com/dtan4/terraforming)
* [Tflint](https://github.com/wata727/tflint)
* [Terraform-docs](https://github.com/segmentio/terraform-docs)
* [Terraform Quantum Provider](https://github.com/coveo/terraform-provider-quantum)
* Shells
  * `bash`
  * `zsh`
  * `fish`
* Tools & editors
  * `vim`
  * `nano`
  * `zip`
  * `git`
  * `mercurial`

### AWS provider specialized image: coveo/tgf:aws (based on Alpine)

All tools included in `coveo/tgf` plus:

* [terraform-provider-aws (fork)](https://github.com/coveo/terraform-provider-aws)

### Kubernetes tools (based on Alpine)

All tools included in `coveo/tgf:aws` plus:

* `kubectl`
* `helm`

### Full image: coveo/tgf:full (based on Ubuntu)

All tools included in the other images plus:

* [AWS Tools for Powershell](https://aws.amazon.com/powershell)
* [Oh My ZSH](http://ohmyz.sh/)
* Shells
  * `powershell`

## Usage

### As Terragrunt front-end

```bash
> tgf plan
```

Invoke `terragrunt plan` (which will invoke `terraform plan`) after doing the `terragrunt` relative configurations.

```bash
> tgf apply -var env=dev
```

Invoke `terragrunt apply` (which will invoke `terraform apply`) after doing the `terragrunt` relative configurations. You can pass any arguments
that are supported by `terraform`.

```bash
> tgf plan-all
```

Invoke `terragrunt plan-all` (which will invoke `terraform plan` on the current folder and all sub folders). `Terragrunt` allows xxx-all operations to be
executed according to dependencies that are defined by the [dependencies statements](https://github.com/coveo/terragrunt#dependencies-between-modules).

### Other usages

```bash
> tgf -E aws s3 ls
```

Invoke `AWS CLI` as entry point and list all s3 buckets

```bash
> tgf -E fish
```

Start a shell `fish` in the current folder

```bash
> tgf -E my_command -i my_image:latest
```

Invokes `my_command` in your own docker image. As you can see, you can do whatever you need to with `tgf`. It is not restricted to only the pre-packaged
Docker images, you can use it to run any program in any Docker images. Your imagination is your limit.
