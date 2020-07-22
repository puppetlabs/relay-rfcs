# Standalone Workflow Execution (2021-07-07)

## Stakeholders

* **Recommenders:** Kyle Terry
* **Agreers:** Noah Fontes, Noah Muldavin, Geoff Woodburn
* **Performers:** Kyle Terry
* **Inputers:** Rick Lane, Sebastian Prokuski
* **Deciders:** Kyle Terry, Noah Fontes, Noah Muldavin

## Problem

We don't have a way to run the full stack of services on a development machine
that allows developers to write code and run workflows against that code. We
have "feature clusters", but they are largely the same size as production and
are costly, time-wise, to setup, maintain and keep up to date.

Similarly, we have a core repository that contains the components to run
workflows, but we don't have a way of easily running those workflows in a
standalone workflow-engine. This also means that workflow and step development
can be painful for users. Having access to a standalone cluster, that's managed
by a simple tool makes debugging workflows and steps much easier by removing the
run-test-debug loop from a completely opaque system.

## Summary

This RFC defines an extension to the Relay CLI that wraps existing container and
Kubernetes management technologies to setup a local Kubernetes cluster that runs
the necessary Relay services for executing workflows in a standalone
environment. This RFC also defines a secondary mechanism to automatically run
workflows after bootstrapping has completed. This will allow the tool to be used
for setting up and managing a full-stack development environment on a local
machine.

## Motivation

The ability to run workflows in a standalone system allows developers of
workflows (Relay users) and Puppet developers who work on the Relay Saas to
reduce churn when testing new code. For the latter, we want to avoid the wait
time for building and pushing new changes to a remote testing cluster, which
requires a cycle through Continuous Integration (CI).

## Product-level explanation

This next section will describe the versions of a tool used to create, manage
and destroy development clusters.

### V1

We add a new set of commands to the [Relay
CLI](https://github.com/puppetlabs/relay) used to manage a local Kubernetes
cluster running the core components that execute workflows.

We also generalize workflow translation and validation code currently in
relay-api into [relay-core](https://github.com/puppetlabs/relay-core) so it can be
used by the new cli commands.

#### Relay CLI Changes

Our goal is to add opinionated commands that can setup a cluster of Kubernetes
nodes as docker containers and bootstrap the bare minimum of services to
successfully execute a `WorkflowRun` CRD tenantless. The new commands are
opinionated in how they build, bootstrap and manage the workflow engine. We
avoid the common pit-fall of over-engineering and reinventing the wheel by only
providing configuration for the most necessary things to run Relay workflows.

To handle the above, we will wrap and leverage
[K3d](https://github.com/rancher/k3d) packages, which can bootstrap a working
Kubernetes cluster inside docker containers.

Clusters are singleton. There either is or isn't a relay development cluster on
a machine. Multi-cluster management is out of the scope of this tool, but if
user-demand for this feature increases, we will address it in a future RFC.

This works on MacOS and Linux as long as a running Docker service is present.

#### Commands

The following are commands to be added to the Relay CLI to support standalone
clusters:

* `relay dev cluster <start|stop|delete>` - Manage the cluster's run state.
* `relay dev kubectl` - Execute kubectl commands against the cluster.
* `relay dev kubeconfig download` - Prints cluster resources and configuration. In
  this example it would print the cluster kubeconfig file as YAML.
* `relay dev image <import|delete>` - Manage container images stored inside the cluster.
* `relay config <set|get>` - Manage relay configuration.
  Currently only used to manage the contexts around production and the development
  cluster.

#### Creating/Starting a cluster

Starting and creating, for all intents and purposes, are the same thing. With
`relay dev` you invoke the `cluster start` command to ensure a cluster is up and running,
whether it exists or not.

#### Stopping a cluster

To stop a cluster, you use `relay dev` and invoke the `cluster stop` command and
all the supporting containers will be shutdown.

#### Destroying a cluster

To destroy a cluster, you use `relay dev` and invoke the `cluster delete`
command, which will stop and destroy running containers and delete any saved
metadata and data files/directories. This can be used to start a new cluster
from scratch.

#### Using `kubectl`

We don't want to mask the truth behind how workflows run. They run in Kubernetes
as pods. Because Kubernetes is such a powerful system, we want to leverage it as
much as possible. Especially when we are developing for the SaaS. If something
fits better as a controller runtime than it does as a goroutine in the API, we
want to encourage developers to try that. For this reason we need to expose the
cluster to the user and ensure they can use standard kubectl commands against
it.

To ensure there's no confusion as to which cluster someone is accessing to
perform maintenance, `kubectl` commands against the standalone cluster are
invoked using `relay dev kubectl`, which knows how to configure kubectl with
the proper kubeconfig file.

Obviously someone can choose to populate their global kubeconfig with the
cluster credentials and change context if they want, but any mistakes caused
doing so is on them. The tool provides a new command `relay dev get kubeconfig`
that will print the file.

### Context switching

Similar to kubectl, we add the ability to switch contexts to the Relay CLI. This
allows us to setup a `dev` context that will allow the relay command to run
workflows against the local cluster. We add a default context called `relay.sh`
that points to the production API. Any context that indicates a local
standalone cluster will not require a login.

To achieve this in a clean way that better blends the relay command between the
SaaS and a local dev cluster, we add a `config` subcommand that has two further
subcommands `set` and `get`.

If you create a dev cluster, it will install a `dev` context. You can switch to
it using `relay config set context dev`. If you want to switch back to
production, you can use `relay config set context relay.sh`.

To avoid complexity, the initial version of the `config` command will not allow
any other additional contexts to be added. Although it will be possible to add
another context by modifying the configuration file by hand. This allows a user
to add a non-production environment, such as staging, for debugging purposes.

#### Bootstrapping

* `relay` creates a data directory in a known location
    - This utilizes the current XDG-compatible directories already supported in
      the CLI configuration by pulling workflows from
      `${HOME}/.config/relay/workflows` for instance.
* `relay` creates a json configuration file in the data directory
* k3d creates a 3 node Kubernetes cluster and `relay` ensures it's running and
  healthy
* Critical 3rd party resources are deployed to the cluster using `kubectl apply`.
    - tekton
    - knative
* Critical 3rd party resources are deployed to the cluster using Helm
    - Ambassador
* Relay resources are deployed to the cluster using `kubectl apply`
    - relay-core CRDs
* Helm chart configuration values are generated and populated into a
  `values.yaml` and saved to disk inside the `config.json` configuration file
* Relay services are deployed to the cluster using Helm. See how container
  image management works below
    - relay-operator
    - metadata-api

This will provide a cluster capable of running workflows that are issued to the
cluster as a `WorkflowRun` resource using kubectl. But `relay` knows how to create
that using workflow YAML by using the same packages for workflow translation the
API uses.

The last part of the bootstrap phase is running workflows.

* `relay` creates a known workflows directory and installs workflows to install
  more core components, such as Vault.
* `relay` runs all workflows inside the known workflows directory.

The workflow directory hook is handy so we can further bootstrap a development
environment for the SaaS or make sure any resources we want will automatically
exist on the cluster without complicating `relay` any futher. We just let the
cluster utilize its own ability to run workflows.

#### Image management

The key to `relay` being useful is proper container image management. This is a
feature that allows a developer to easily write code locally and run it against
a cluster without committing and waiting for a continuous integration cycle.

Container images for Relay services are initially pulled using the `latest` tag
and retagged to `local` (e.g. `relay-operator:local`) unless a `local` tag already
exists for that image.  These local images are then imported into the cluster by
`relay`.

When a development build script for a Relay service is ran, the resulting
docker image is tagged using `local` and if the `relay` utility exists in `$PATH`,
a call to `relay dev image import` is made to instruct `relay` to import the new
image. Unrelated to image management, the build script can then trigger a
workflow run for the Relay service in question.

The image import command supports the `-u` flag to update the image. This action
will check if the host has a more up to date version of the image. This allows
an external script to "push" newly built images to the cluster and then trigger
a workflow run that will apply it.

To delete an image from the cluster, the `image delete` command exists.

#### Persisted configuration

In order to start and stop cluster, and manage deployments and users, `relay`
accesses all the data it needs from a `config.json` file that is populated when
the cluster is created. This contains the kubeconfig, database credentials,
vault credentials, helm values, and references to deployment images.

This file is the source of authority for all cluster configuration. Changes made
to it will reflect down into anything else that exists for the cluster on disk
(such as the helm values.yaml). This being said, `relay` will not use this file to
update things like database and vault credentials. Any changes there will need
to be done manually.

## Engineering-level explanation

While versions beyond V1 have not been planned, I can see requests for changes
to this system once it's put to use by the team and we should plan for bigger
changes using extension RFCs to this one. Below I've outlined rough steps that
should get us to V1.

### V1: Opinionated and just works

The first version should not allow for much configuration. It should provide the
developer with a working cluster that can update Relay service deployments with
new container images and run workflows without having to POST them to an
instance of `relay-api`.

In this version, we will add new commands to the Relay CLI. These commands will
utilize packages from K3d, Kubernetes and other tools in the space that do a lot
of the heavy lifting already.

This version will require the developer to install a couple hard dependencies in
order for `relay dev cluster` to function:

* kubectl
* docker

### SaaS Development Environment

Once we get `relay-operator` working and running workflows, we create a
development-environment workflow that will install remaining SaaS infra. These
steps will include helm installs for Postgres, Redis, relay-api, and relay-ui.

The UI and API services will be exposed on host ports through the k3s ingress
controller.

## Drawbacks

This might encourage developers to write less unit and integration tests. This
must be avoided. We have to ensure all pull requests contain proper unit and
integration tests for the service. It must be emphasized that `relay cluster` is
not a replacement for continuous integration.

## Rationale and alternatives

### Why is this design the best in the space of possible designs?

Running a local cluster means the cycle of writing code and running it against
the relay system to see what happens can be quick. Developers can experiment a
lot more with structure and risky changes. This also allows for verification of
bug fixes to happen quicker.

Other hopes for this system is catching subtle differences between two services
that can be added to the test suite. For example: if you write code around the
assumption that all services running are using the same message parsing module,
you can quickly test this new code around various versions of the services. If
you catch a failure between two versions, you can modify your code to support
the different version and add test cases for that specifics. Or you can raise a
red flag and start a proper deprecation of old message structures.

### What other designs have been considered and what is the rationale for not choosing them?

Other ideas that have been pitches are an always-on remote development cluster
in GKE or the like. This has the drawback of being costly, has long development
churn, and most-likely requires a CI build to run code.

### What is the impact of not doing this?

Developers will continue writing critical pieces of code and not running it
locally before they commit. This causes a lot of frustration when subtle nuances
between the services prevent the code from working properly. This can be
effected by message payloads being slightly different between service versions,
causing issues that weren't cause with testing code because of running version
differences.

### What specific risks are associated with this design?

Over-engineering. We don't want to get distracted by creating yet another tool
for general use. `relay` must always wrap existing tools and be opinionated around
how to run Relay on a developer's machine.

## Success criteria

Users, workflow authors and developers can develop workflows without connecting
to the remote Relay production service.

Relay Developers can easily, without writing complex configuration, start a
development cluster and run whatever version of a Relay service they want.

## Future possibilities

Ideally `relay` should be able to obtain a list of supported service versions, then
allow a developer to `next` through them to create various possible service
configurations.

We should make the cluster lightweight enough to possibly use it as an
integration testing system. This would allow us to smoke test the entire stack,
continuously, all day, cycling through supported versions and running a test
suite against it.
