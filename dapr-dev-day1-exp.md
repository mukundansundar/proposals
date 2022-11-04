# Proposal Dapr User Day 1 Experience

## Introduction

As a Dapr user, I need to be able to quickly setup, run and experiment using Dapr. As a new user looking at Dapr, this proposal aims to improve the experience for the user to quickly onboard onto Dapr.

## Terms used

Dapr User / User : A person who uses Dapr for developing applications. 

Dapr : The Dapr project

## Scope

In local development environments it should be easy to run, experiment, use Dapr. Currently the scope only targets local development using self hosted mode of Dapr. In future, we can also look into improving our k8s and local-cloud experiences.

## Current problem areas

There are a couple problem areas that have been identified by users of `dapr` CLI:
- There is a need to keep track of multiple ports, app-ids, etc
- There is a need to run multiple services with multiple `dapr run` commands to get a quick microservice setup up and running
- There are a too many arguments, environment variables that can be set which do not provide effective defaults. This also adds to the initial friction in onboarding onto Dapr.
- There is a need to define multiple component files, with multiple different metadata per component which adds to the learning curve to start using Dapr.
- There are no tools to easily discover what components are supported, and to validate component YAMLs for correctness. 
- Not all building blocks are easily accessible from the `dapr` CLI
- Ability to install and run against edge/nightly dapr builds?

## Potential solutions for the problems identified above

For some of the areas identified, these are some of the proposed solutions: 
- Component discovery and definition
  - Add command to [lint](https://github.com/dapr/cli/issues/1074) component files and validate them.
  - Add command to [discover and generate](https://github.com/dapr/cli/issues/992) YAML for components with sane defaults 
- Expose all building blocks and features in Dapr through `dapr` CLI
  - Add support for [resiliency defaults](https://github.com/dapr/cli/issues/1027)
  - Add command support for all [custom CRDs](https://github.com/dapr/cli/issues/950) in Dapr
  - [Default secret store](https://github.com/dapr/cli/issues/1066) on init
  - Have quickstarts defined for all building blocks 
  - Improve existing command support for service invocation and publish 
  - Add additional command support for bindings, secrets, configuration, state management etc. 
- Improve `dapr init` experience 
 - Have a CLI [config file](https://github.com/dapr/cli/issues/1058) with sane defaults and additional knobs to imrpove `init` experience
- Run multiple services with Dapr sidecar, and keep track of ports
  - Proposed solution is [dapr compose](https://github.com/dapr/cli/issues/1123)

## Dapr Compose

### Overview

This is a proposal for `dapr compose` command to be included in the dapr CLI. This is so that it is easy to orchestrate multiple services that needs to be run in tandem along with their daprd sidecars in local self-hosted mode. 

### Background

> from dapr/community proposal - https://github.com/dapr/community/issues/207

Currently to run multiple services along with `dapr sidecar` locally, users need to run multiple `dapr run` commands, keep track of all ports opened, the components folders that each service refers to, the config file each service refers to etc. 
There are also multiple other flags that can be used to tweak behavior of `dapr run` command eg: `--unix-domain-socket`, `--dapr-internal-grpc-port`, `--app-health-check-path` etc.

This increases the complexity of using dapr in development, where users want to run multiple services in local mode and be able to partially/fully replicate the production sceanrio. 

In K8s mode this is alleviated through the use of helm/deployment YAML files. There is currently no such capability available for local self hosted mode. 

Asking a user to run multiple different `dapr run` commands each with different flags, increases the learning curve for new users onboarding onto Dapr. 

#### dapr CLI or sanbox project?

From the initial [proposal](https://github.com/dapr/community/issues/207), the solution was proposed as a seprate repo and CLI in itself. But later it was suggested to use dapr CLI itself to have a `compose` command in it instead of a having a separate CLI. 
The main reason for including it in the dapr CLI itself is that, users do not have to download and use a new CLI in addition to the dapr CLI.

This feature is tightly coupled and opinionated on how `dapr` is going to be run locally, and having a separate CLI `dapr-compose` deciding on how `dapr` CLI should be used, is not a good pattern to start with. 

> Note: `daprd` is more generally used and considered a binary and not necessarily a CLI tool. So `dapr` CLI is not making use of another CLI but rather passing on configs for running a binary.

Another option is to have this as a dapr-sandbox project `dapr-compose` with the caveat that it will be moved into dapr CLI as `dapr compose` command when it graduates. This restricts that `dapr-compose` be built in such a way that it can be easily integrated into `dapr` CLI. But the advantage here is that, we can iterate much faster if needed on the sandbox project. (Additional work)
Also we clearly do not affect the `dapr` CLI with _any preview code_, in any way since it will be seperate project/CLI. 
Disadvantage as mentioned above is that it is yet another CLI to download and use. 

### Analysis

### Proposed Structure 

```yaml
version: 1
common:
  env:  # any environment variable shared among apps
    - DEBUG: true
apps:
  - app_id: webapp
    work_dir: ./webapp/
    resources_dir: ./webapp/components # can be default by convention too, ignore if dir is not found.
    config_file: ./webapp/config.yaml # can be default by convention too, ignore if file is not found.
    app_protocol: HTTP
    app_port: 8080
    command: ["python3" "app.py"]
  - app_id: backend
    work_dir: ./backend/
    resources_dir: ./backend/components # can be default by convention too, ignore if dir is not found.
    config_file: ./backend/config.yaml # can be default by convention too, ignore if file is not found.
    app_protocol: GRPC
    app_port: 3000
    env:
      - DEBUG: false
    command: ["./backend"]
```

- Each file conatins a `common` object which contains `env`, `resources_dir` and `config_file` that can be used in common across all the apps defined in this YAML
- There is an `apps` section that lists the different app configs. 
- Each app config has the followin
  - `app_id` mandatory field 
  - `work_dir` root directory of the application 
  - `resources_dir` directory of all dapr resources (components, resiliency policies, subscription crds) (overrides common def)
  - `config_file` the configuration file to be used for this app (overrides common def)
  - `app_protocol` HTTP, gRPC defaults to HTTP
  - `app_port` port the app listens to if any
  - other dapr run parameters that are not generally `daprd` passthroughs
  - `command` ["exec" "arg1" "arg2"] format for application command
  - `env` which overrides or adds to common env var defined or the shell env var passed in when `dapr compose` is called


#### Constraints

- `dapr compose` should be able to access only the **pwd and sub folders** from where it is called from. 
- any parameters not present in the compose yaml, will default to existing values
- command has only one form as seen above, the first is the executable and others are args to the executable
- command if not given will only start the sidecar with the id and other required parameters

#### Execution options
There are two options on how `dapr compose` executes 
- executes and creates an interactive shell which will log any state changes in apps etc. state is managed only in memory and if the command is exited, all the apps and sidecar processes also exit. (`dapr compose run` command)
- executes such that even if the command exits, a fork is created and the rest of the processes continue to run. this will require a separate state managenet solution to manage state of app and sidecar, also require a two seperate commands `dapr compose up` and `dapr compose down` to start and stop the apps defined in the yaml file. 

### Surfacing apps run through `dapr compose`

Any apps started with `dapr compose` should still be allowed to be individually controlled or managed by existing `dapr` commands? 
Any apps started with `dapr compose` should be visible in `dapr dashboard`?

### Open Questions
1. Should dapr CLI have persistent state of each APP, so that if one of the apps fails to start or crashes, then CLI can restart the app?
1. Should dapr CLI run through `compose` run the apps in containers? Inherently there might be issues that crop up due to env isolation not being available and running in different Oses. Ideally runnning containers is the best way, but building containers and running them against host network in itself might tie dapr compose to a specific container runtime(podman might not support what docker supports (yet))
1. Should `dapr compose` expose ability to users to set all the different flags that `daprd` exposes? Its a moving target for `dapr run`. Relavant https://github.com/dapr/cli/issues/1038
1. Should `dapr compose` be extended to other platforms? In which case going with containers is again the solution to look into, since platforms like k8s only supports containers 
1. Try looking into CNAB and see if that can be used for `dapr compose`

### References 
Project Tye - https://timdeschryver.dev/blog/tye-starting-and-running-multiple-apis-with-a-single-command#running-the-environment
