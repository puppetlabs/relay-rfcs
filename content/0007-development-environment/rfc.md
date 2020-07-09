# Development Environment (2020-07-07)

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

## Summary

This RFC defines a tool that wraps existing container technologies to setup a
local Kubernetes cluster that runs the necessary Relay services for running the
Relay SaaS on a developer's computer.

## Motivation

Having a quick and simple tool for bootstrapping, maintaining and updating a
local cluster will reduce churn in the development cycle and help eliminate the
deployment of bugs. The testing harness for running workflows kicked off from
the API is very complicated. This project hopes to reduce the amount of times a
developer writes some code, commits, waits for CI builds, waits for CD to deploy
against a remote cluster, then is able to run the code against a real
environment.

## Product-level explanation

[Explain this change in a way that our product team can understand it and pitch
it to customers for feedback. Make heavy use of examples and ensure you
introduce any new concepts at a high level.]

### V1

We introduce a tool called RDE (relay development environment) into the
deployment repository.

#### RDE

RDE (from now on referred to as `rde`) is a shell script that wraps
[K3d](https://github.com/rancher/k3d) and is opinionated in how it builds the
development environment. We avoid the common pit-fall of over-engineering and
reinventing the wheel by only providing configuration for the most necessary
things.

If `rde` exists in `$PATH`, then relay repositories use it to update their
respective deployments with a local build _without_ configuration at all.

Clusters are singleton. There either is or isn't a relay development cluster on
a machine.

`rde` works on MacOS and Linux as long as a running Docker service is present.

#### Creating/Starting a cluster

Starting and creating, for all intents and purposes, are the same thing. With
`rde` you invoke the `cluster start` command to ensure a cluster is up and running,
whether it exists or not.

#### Stopping a cluster

To stop a cluster, you invoke the `cluster stop` command and all the supporting
containers will be shutdown.

#### Destroying a cluster

To destroy a cluster, you invoke the `cluster delete` command, which will stop
and destroy running containers and delete any saved metadata and data
files/directories. This can be used to start a new cluster from scratch.

#### Using `kubectl`

To ensure there's no confusion as to which cluster operations team members are
accessing to perform maintenance, `kubectl` commands against the development
cluster are invoked using `rde kubectl`, which knows how to configure kubectl
with the proper kubeconfig file.

Obviously someone can choose to populate their global kubeconfig with the
cluster credentials and change context if they want, but any mistakes caused
doing so is on them.

#### Bootstrapping

* k3d creates a 3 node Kubernetes cluster and `rde` ensures it's running and
  healthy
* A kubectl config file is saved the `rde` data directory.
* Critical 3rd party resources are deployed to the cluster using `kubectl apply`.
    * tekton
    * knative
* Critical 3rd party resources are deployed to the cluster using Helm.
    * Postgresql
    * Vault
    * Redis
    * Ambassador
* Relay resources are deployed to the cluster using `kubectl apply`.
    * relay-core CRDs
* Helm chart configuration values are generated and populated into a
  `values.yaml` and saved to disk.
* Relay services are deployed to the cluster using Helm. See how container
  image management works below.
    * relay-core
    * relay-api
    * relay-ui

#### Image management

The key `rde` being useful is proper container image management. This is a
feature that allows a developer to easily write code locally and run it against
a cluster without committing and waiting for a continuous integration cycle.

Container images for Relay services are initially pulled using the `latest` tag
and retagged to `local` (e.g. `relay-api:local`) unless a `local` tag already
exists for that image.  These local images are then imported into the cluster by
`rde`.

When a development build script for a Relay service is ran, the resulting
docker image is tagged using `local` and if the `rde` utility exists in `$PATH`,
a call to `rde update images` is made to instruct `rde` to import the new image
and force a new ReplicaSet to be created for the deployment.

#### Initial user

`rde` has a command `rde user create`, which adds a user record to the Relay
database.

#### Persisted configuration

In order to start and stop cluster, and manage deployments and users, `rde`
accesses all the data it needs from a `config.json` file that is populated when
the cluster is created. This contains the kubeconfig, database credentials,
vault credentials, helm values, and references to deployment images.

This file is the source of authority for all cluster configuration. Changes made
to it will reflect down into anything else that exists for the cluster on disk
(such as the helm values.yaml). This being said, `rde` will not use this file to
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

In this version, `rde` will live in `relay-deploy` as a standalone shell script
or Go wrapper around k3d. This can be installed to a developers `$PATH` either
by coping the script or binary, or running a `make` target from `relay-deploy`.

This version will require the developer to install a couple hard dependencies in
order for `rde` to function:
    * k3d
    * kubectl

The benefit of the Go-based wrapper is k3d won't need to be installed by hand as
`rde` will pull it in as a package. In this scenario, only kubectl will need to
exist on the system.

## Drawbacks

This might encourage developers to write less unit and integration tests. This
must be avoided. We have to ensure all pull requests contain proper unit and
integration tests for the service. It must be emphasized that `rde` is not a
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
for general use. `rde` must always wrap existing tools and be opinionated around
how to run Relay on a developer's machine.

## Success criteria

Developers can easily, without writing complex configuration, start a
development cluster and run whatever version of a relay service they want.

## Future possibilities

Ideally `rde` should be able to obtain a list of supported service versions, then
allow a developer to `next` through them to create various possible service
configurations.

We should make the cluster lightweight enough to possibly use it as an
integration testing system. This would allow us to smoke test the entire stack,
continuously, all day, cycling through supported versions and running a test
suite against it.
