# Standalone Workflow Execution (2020-07-07)

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
standalone workflow-engine.

## Summary

This RFC defines a tool that wraps existing container and Kubernetes management
technologies to setup a local Kubernetes cluster that runs the necessary Relay
services for executing workflows in a standalone environment. This RFC also
defines a secondary mechanism to automatically run workflows after bootstrapping
has completed. This will allow the tool to be used for setting up and managing a
full-stack development environment on a local machine.

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

NOTE: _rde_ is used a placeholder for search and replace until a name is agreed
upon for the tool. Relay Development Environment (rde) was the name given to the
tool during the first draft of this RFC, so I used it here just for
convenience.

We introduce a tool called _rde_ into the relay-core repository.

#### _rde_

_rde_ is a software tool that wraps [K3d](https://github.com/rancher/k3d) and is
opinionated in how it builds and bootstraps the workflow engine. We avoid the common
pit-fall of over-engineering and reinventing the wheel by only providing
configuration for the most necessary things to run Relay workflows.

Clusters are singleton. There either is or isn't a relay development cluster on
a machine. In V2, we will explore the ability to run more than 1 cluster by
name, but this is notoriously complicated, so we are avoiding this complexity in
V1.

_rde_ works on MacOS and Linux as long as a running Docker service is present.

NOTE: We use the term cluster to describe the system as a whole because its
backend will always be a cluster of Kubernetes nodes. I've iterated on this a
number of times and describing the system as a cluster is always the one that
makes sense.

#### Commands

`_rde_ cluster start` - Start the cluster if it exists. Create and start the cluster
if it does not exist.

`_rde_ cluster stop` - Stop all nodes in the cluster, sending proper termination
signals to the Docker containers.

`_rde_ cluster delete` - Stop and delete the cluster and any metadata.

`_rde_ kubectl` - Execute kubectl commands against the cluster.

`_rde_ images import` - Imports an image from a remote or the local host into
the cluster.

`_rde_ images update` - Updates all images imported into the cluster.

`_rde_ relay` - Execute relay CLI commands against the cluster.

#### Creating/Starting a cluster

Starting and creating, for all intents and purposes, are the same thing. With
_rde_ you invoke the `cluster start` command to ensure a cluster is up and running,
whether it exists or not.

#### Stopping a cluster

To stop a cluster, you invoke the `cluster stop` command and all the supporting
containers will be shutdown.

#### Destroying a cluster

To destroy a cluster, you invoke the `cluster delete` command, which will stop
and destroy running containers and delete any saved metadata and data
files/directories. This can be used to start a new cluster from scratch.

#### Using `kubectl`

We don't want to mask the truth behind how workflows run. They run in Kubernetes
as pods. Because Kubernetes is such a powerful system, we want to leverage it as
much as possible. Especially when we are developing for the SaaS. If something
fits better as a controller runtime than it does as a goroutine in the API, we
want to encourage developers to try that. For this reason we need to expose the
cluster to the user and ensure they can use standard kubectl commands against
it.

To ensure there's no confusion as to which cluster someone is
accessing to perform maintenance, `kubectl` commands against the _rde_
cluster are invoked using `_rde_ kubectl`, which knows how to configure kubectl
with the proper kubeconfig file.

Obviously someone can choose to populate their global kubeconfig with the
cluster credentials and change context if they want, but any mistakes caused
doing so is on them.

#### Using `relay` (the CLI command)

We also don't want to duplicate the effort put into the Relay CLI to run
workflows, so _rde_ provides a command to execute `relay` commands against the
local cluster. Since the local cluster is essentially single-user, this won't
require a prior login command.

NOTE: to support this, we will need to design modifications to the Relay CLI to
issue workflow runs against a stand-alone workflow engine.

#### Bootstrapping

* _rde_ creates a data directory in a known location
* _rde_ creates a json configuration file in the data directory
* k3d creates a 3 node Kubernetes cluster and _rde_ ensures it's running and
  healthy
* Critical 3rd party resources are deployed to the cluster using `kubectl apply`.
    - tekton
    - knative
* Critical 3rd party resources are deployed to the cluster using Helm
    - Ambassador
    - Vault
* Relay resources are deployed to the cluster using `kubectl apply`
    - relay-core CRDs
* Helm chart configuration values are generated and populated into a
  `values.yaml` and saved to disk inside the _rde_.json configuration file
* Relay services are deployed to the cluster using Helm. See how container
  image management works below
    - relay-core

This will provide a cluster capable of running workflows that are issued to the
cluster as a `Workflow` resource using kubectl. But _rde_ knows how to create
that using workflow yaml by using the same packages for workflow translation the
API uses.

NOTE: I think we still need to create common go packages that can take a
workflow yaml and create a `Workflow` resource structure that can be shared
between this and `relay-api`. If this exists already, please let me know.

The last part of the bootstrap phase is running workflows.

* _rde_ runs all workflows provided by the user.
    - we check of a known workflow directory exists
    - if the directory exists, all yaml files are ran against the cluster

The workflow directory hook is handy so we can further bootstrap a development
environment for the SaaS or make sure any resources we want will automatically
exist on the cluster without complicating _rde_ any futher. We just let the
cluster utilize its own ability to run workflows.

#### Image management

The key _rde_ being useful is proper container image management. This is a
feature that allows a developer to easily write code locally and run it against
a cluster without committing and waiting for a continuous integration cycle.

Container images for Relay services are initially pulled using the `latest` tag
and retagged to `local` (e.g. `relay-operator:local`) unless a `local` tag already
exists for that image.  These local images are then imported into the cluster by
_rde_.

When a development build script for a Relay service is ran, the resulting
docker image is tagged using `local` and if the _rde_ utility exists in `$PATH`,
a call to `_rde_ images import` is made to instruct _rde_ to import the new
image. Unrelated to image management, the build script can then trigger a
workflow run for the Relay service in question.

#### Persisted configuration

In order to start and stop cluster, and manage deployments and users, _rde_
accesses all the data it needs from a `config.json` file that is populated when
the cluster is created. This contains the kubeconfig, database credentials,
vault credentials, helm values, and references to deployment images.

This file is the source of authority for all cluster configuration. Changes made
to it will reflect down into anything else that exists for the cluster on disk
(such as the helm values.yaml). This being said, _rde_ will not use this file to
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
new container images.

In this version, _rde_ will live in `relay-core` as a standalone shell script
or Go wrapper around k3d. This can be installed to a developers `$PATH` either
by coping the script or binary, or running a `make` target from `relay-core`.

This version will require the developer to install a couple hard dependencies in
order for _rde_ to function:
    * k3d
    * kubectl

The benefit of the Go-based wrapper is k3d won't need to be installed by hand as
_rde_ will pull it in as a package. In this scenario, only kubectl will need to
exist on the system.

## Drawbacks

This might encourage developers to write less unit and integration tests. This
must be avoided. We have to ensure all pull requests contain proper unit and
integration tests for the service. It must be emphasized that _rde_ is not a
replacement for continuous integration.

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
for general use. _rde_ must always wrap existing tools and be opinionated around
how to run Relay on a developer's machine.

## Success criteria

Developers can easily, without writing complex configuration, start a
development cluster and run whatever version of a relay service they want.

## Future possibilities

Ideally _rde_ should be able to obtain a list of supported service versions, then
allow a developer to `next` through them to create various possible service
configurations.

We should make the cluster lightweight enough to possibly use it as an
integration testing system. This would allow us to smoke test the entire stack,
continuously, all day, cycling through supported versions and running a test
suite against it.
