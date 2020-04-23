# Layout of an integration (2020-04-07)

## Stakeholders

* **Recommenders:** Noah Fontes
* **Agreers:** Brad Heller, Kenaz Kwa
* **Performers:** [...]
* **Inputers:** Rick Lane, Sebastian Prokuski, Michael Sigler, Geoff Woodburn
* **Deciders:** Brad Heller, Kenaz Kwa, Eric Sorenson, Geoff Woodburn

## Problem

We have grand ideas about what we can do with an integration, but we don't know
how to organize one yet.

## Summary

This RFC sets out a standard directory structure that all integrations must
conform to. It also clarifies expected metadata and provides extension points
for future work.

## Motivation

Having a standard structure and metadata for modules has benefited Puppet's
other portfolio products. Having a consistent layout makes it easy for us to
build tooling and removes a significant mental burden from community
contributors trying to get started with our platform.

## Product-level explanation

Throughout this RFC, we use the GitHub integration as the example.

### Directory layout

Integrations have a well-defined directory structure. Note that this is _not_ a
source control repository structure; with a few exceptions (described below),
integrations are not dependent on source control. You can have many integrations
in one source control repository or not use source control at all.

An empty directory structure for a newly created integration:

```
📦 my-integration
┣ 📂 actions
┇ ┣ 📂 queries
┇ ┣ 📂 steps
┇ ┗ 📂 triggers
┣ 📜 README.md
┗ 📜 integration.yaml
```

How the GitHub directory structure might look:

```
📦 github
┣ 📂 actions
┇ ┣ 📂 queries
┇ ┣ 📂 steps
┇ ┇ ┣ 📂 issue-create
┇ ┇ ┇ ┣ 📂 media
┇ ┇ ┇ ┇ ┗ 📜 ticket.svg
┇ ┇ ┇ ┣ 📜 Dockerfile
┇ ┇ ┇ ┣ 📜 README.md
┇ ┇ ┇ ┣ 📜 spec.schema.json
┇ ┇ ┇ ┣ 📜 outputs.schema.json
┇ ┇ ┇ ┗ 📜 step.yaml
┇ ┇ ┣ 📜 pull-request-create.yaml
┇ ┇ ┣ 📜 pull-request-create@2.yaml
┇ ┇ ┗ 📜 pull-request-merge.yaml
┇ ┗ 📂 triggers
┇   ┣ 📜 issue-opened.yaml
┇   ┣ 📜 pull-request-merged.yaml
┇   ┣ 📜 pull-request-opened.yaml
┇   ┣ 📜 pushed.yaml
┇   ┗ 📜 release-published.yaml
┣ 📂 images
┇ ┣ 📂 pr
┇ ┇ ┣ 📜 Dockerfile
┇ ┇ ┗ 📜 build.yaml
┇ ┗ 📂 webhook
┇   ┣ 📜 Dockerfile
┇   ┗ 📜 build.yaml
┣ 📂 media
┇ ┗ 📜 logo.svg
┣ 📜 README.md
┗ 📜 integration.yaml
```

### Top-level metadata

The presence of the `integration.yaml` (or `integration.json`) file indicates
the root of an integration directory. The content provides basic information
about the integration:

```yaml
# The schema version. Required. Must be exactly the string "integration/v1".
apiVersion: integration/v1

# The schema kind. Required. Must be exactly the string "Integration".
kind: Integration

# The integration slug. Required.
name: github

# The release channel for this integration. One of alpha, beta, rc or stable.
# Optional. Defaults to stable.
channel: stable

# The version of the integration. Required. This is an arbitrary text string
# used to identify the bundle of actions for this release. Using dates or
# incrementing integers is recommended. Each version replaces the one before it
# on the specified release channel.
#
# As a convenience, this information may come from another text file or from a
# Git tag (if applicable). We may add additional sources in the future.
version: v20200408
#version:
#  source: git-describe
#version:
#  source: file
#  file: VERSION

# The shortest possible accurate description of this integration. Required. May
# just be the name of the service.
summary: GitHub

# A paragraph or two describing what this integration does. Optional. Markdown.
description: |
  GitHub is the predominant Git repository hosting service. Host your private
  or public Git repositories here. I'm telling you things you already know.

# License information. Optional. If not specified, this integration is assumed
# to be unlicensed proprietary code. Otherwise, must be a valid SPDX License
# Expression (similar to npm) or an object referencing a license file.
license: Apache-2.0
#license:
#  source: file
#  file: LICENSE

# Owner of this integration. Required, but may be optional in the future if it
# can be inferred from the provider of the integration.
owner:
  name: Puppet, Inc.
  email: relay@puppet.com
  url: https://puppet.com

# URL to this integration's Website. Optional.
homepage: https://github.com/relay-integrations/github

# URL to this integration's source code. Optional. If the homepage is a GitHub
# link, may be inferred automatically.
source: git://github.com/relay-integrations/github.git

# URL or path relative to this file to an icon or icons representing this
# integration. Optional.
icon: media/logo.svg
#icon:
#  tiny: media/logo-tiny.svg
#  medium: media/logo.svg

# Free-form labels to help people find this integration. Optional.
tags:
- git
- source control
```

A minimal example to demonstrate what can be safely omitted:

```yaml
apiVersion: integration/v1
kind: Integration
name: github
version: v20200408
summary: GitHub
license: Apache-2.0
owner:
  name: Puppet, Inc.
  email: relay@puppet.com
  url: https://puppet.com
```

### Top-level README

The `README.md` file in the root of the integration contains an overview of the
integration, including implementation examples, in markdown format.

Although we won't enforce a rigorous structure, we provide a template for users
to modify with the following sections:

* External requirements
* Getting started
* Examples
* Contributing

### Actions

Each kind of action (query, step, and trigger) has its own metadata format that
share a common structure to identify the Docker image to use as a base for the
action.

Each action in an integration has its own directory corresponding to the slug
for the action. This directory contains metadata for the action as well as any
files needed for the action.

As a convenience to authors, we provide one normalization feature: if no extra
files are needed for a given action, the structure
`actions/<type-plural>/<name>[@version]/<type>.yaml` may be abbreviated to
`actions/<type-plural>/<name>[@version].yaml`. If both the abbreviated YAML file
and directory exist simultaneously, the directory takes priority and a warning
shall be emitted by developer tooling.

#### README

Each action may include a `README.md` file. If present, it should include an
overview of the action, including usage examples, in markdown format.

Like the top-level README, we'll provide a template for each kind of action. All
templates will include a section for examples; some actions may have additional
sections.

#### Versioning

An action itself only has one version identifier: a major version. The major
version represents API compatibility only. That is, if any inputs (spec) or
outputs of an action change, the version must be incremented. Otherwise, it can
stay the same.

Versioning in integrations is somewhat nuanced because we're trying to solve two
pragmatic problems:

1. We don't want integration authors to be stuck in version hell. We want to
   avoid having to bump the major version of an integration if one action has an
   incompatible change. We also want to avoid authors having to mess with
   individual version numbers in each action.
2. Users need to be able to quickly find the version of an action that supports
   what they need. Once they find it, they depend on it not to arbitrarily break
   (semantic versioning).

The solution is combining multiple pieces of information, some from the
integration and some from the action. The complete version identifier for an
action is the tuple of the integration release channel, the action major
version, and the integration version. For example, for the GitHub integration on
the beta channel with release version `v20200408` and the step `issue-create`
with major version 3, the full version identifier is `(beta, 3, v20200408)`.

In general, we expect users to stick with expressing their version requirements
in terms of a release channel (usually defaulting to stable) and a major
version. However, if a user does want to pin versions, they do have the ability
to identify a specific release version.

We expect that some integrations will need to simultaneously maintain two major
versions of the same action. Action directory names (or abbreviated YAML files)
can optionally contain the major version separated from the name by an `@`
character. If one or more versioned action directory names exist along with a
non-versioned directory name, the non-versioned directory's major version must
be larger than the largest versioned directory's major version.

#### Common metadata

All action metadata has the following fields:

```yaml
# The schema version. Required. Must be exactly the string "integration/v1".
apiVersion: integration/v1

# The schema kind. Required. Must be one of "Query", "Step", or "Trigger"
# corresponding to its directory location.
kind: Step
#kind: Query
#kind: Trigger

# The name of the action. Required. Must be exactly the name of the directory
# containing the action.
name: issue-create

# The version of the action. Required. Must be an integer. If specified in the
# directory name, must be exactly the version in the directory name.
version: 3

# High-level phrase describing what this action does. Required.
summary: Create an issue

# Single-paragraph explanation of what this action does in more detail.
# Optional. Markdown.
description: |
  Creates a new issue (if issues are enabled).

# URL or path relative to this file to an icon or icons representing this
# action. Optional. Defaults to the integration icon.
icon: media/ticket.svg
#icon:
#  tiny: media/ticket-tiny.svg
#  medium: media/ticket.svg

# The mechanism to use to construct this step. Required. Must be an action
# builder. See the Builders section below.
build:
  # The schema version for builders. Required. For now, must be the exact
  # string "build/v1". We may consider supporting custom third-party builders
  # in the future.
  apiVersion: build/v1

  # The builder to use. Required.
  kind: Docker

  # Additional Docker build arguments.
  args:
    MYARG: hello
```

#### Query metadata

Query metadata is out of scope for this RFC because the feature has not been
defined yet.

#### Step metadata

Step metadata extends the common action metadata:

```yaml
schemas:
  # The schema for the Relay spec for this container. Optional. Accepts any
  # value if not specified. May either be inlined into the YAML or reference an
  # external file.
  spec:
    $schema: http://json-schema.org/draft-07/schema#
    type: object
    properties:
      title:
        type: string
        description: The title of the issue to open
  #spec:
  #  source: file
  #  file: spec.schema.json

  # The schema for the outputs of this container. Optional. If not specified,
  # outputs may take any form. May either be inlined into the YAML or reference
  # an external file.
  outputs:
    $schema: http://json-schema.org/draft-07/schema#
    type: object
    properties:
      number:
        type: integer
        description: The new issue number
  #outputs:
  #  source: file
  #  file: outputs.schema.json
```

An example of a simple practical step metadata file:

```yaml
apiVersion: integration/v1
kind: Step
version: 1
summary: Create an issue
build:
  apiVersion: build/v1
  kind: Docker
```

#### Trigger metadata

Trigger metadata extends the common action metadata:

```yaml
# The implemented trigger types. Required. For now, only "webhook" is supported.
# When the trigger container is started by Relay, the requested responder is
# passed to the container in the RELAY_RESPONDER environment variable.
responders:
- webhook

schemas:
  # The schema for the Relay spec for this container. Optional. Accepts any
  # value if not specified. May either be inlined into the YAML or reference an
  # external file.
  spec:
    $schema: http://json-schema.org/draft-07/schema#
  #spec:
  #  source: file
  #  file: spec.schema.json

  # The schema for events emitted by this container. Optional. If not
  # specified, the structure of events is not validated. May either be inlined
  # into the YAML or reference an external file.
  event:
    $schema: http://json-schema.org/draft-07/schema#
  #event:
  #  source: file
  #  file: event.schema.json
```

### Builders

We want users to know that they have the flexibility of using a Dockerfile to
build actions when they need to, but we also want to provide convenient
shortcuts where possible. Builders allow us to abstract the notion of image
construction so that it's easy to just dump some source code (Go, Python, etc.)
in a directory and have it automatically work with our service.

Our tooling, in particular the `relay devel new` command, should be able to
prompt the user for the builder to use and set up sensible defaults for the new
action.

#### `DockerOverride`

This builder creates an action by customizing another image.

```yaml
# The image to inherit. Required. Either a string referencing an existing
# Docker image or an object referencing another builder in this integration.
image: alpine:latest
#image:
#  source: file
#  file: ../../images/pr

# Change the entrypoint of the image. Optional.
command: /bin/pr-create

# Change the command arguments to the entrypoint. Optional.
args: [--force]

# Change standard input for the entrypoint. Optional.
input: |
  This is my pull request description.

# Additional environment variables to define. Optional.
env:
  MYENV: value
```

#### `Docker`

This builder creates an action using a Dockerfile collocated with the action
metadata.

```yaml
# The path to the Dockerfile to build. Optional. Defaults to Dockerfile in the
# current directory.
dockerfile: Dockerfile

# The Docker build context. Optional. Defaults to the current directory.
context: .

# Additional Docker build arguments. Optional.
args:
  MYARG: hello
```

#### `Go`

This builder creates an action using Go source code.

```yaml
# The path to the directory to copy as the Go module to build. Optional.
# Defaults to the current directory.
context: .

# The path to one or more commands to install. Required. Relative to the
# context. Commands are installed to /usr/local/bin in the built container.
install:
- ./cmd/pr-create
- ./cmd/pr-merge

# The entrypoint for the container. Optional. Defaults to the first command
# built.
command: pr-merge

# Arguments to the command. Optional.
args: [--force]

# Standard input for the container. Optional.
input: |
  A very nice pull request.
```

#### `PythonPackage`

This builder creates an action using Python setuptools, installing any
dependencies automatically. If the source code includes `scripts` or
`console_scripts` in its setuptools call, they will be installed to the image.

```yaml
# The path to the directory to copy as the Python package to build. Optional.
# Defaults to the current directory.
context: .

# The entrypoint for the container. Optional. If not specified, defaults to the
# Python interpreter.
command: pr-merge

# Arguments to the to the entrypoint. Optional. If not specified, defaults to
# [-m, <package-name>].
args: [--force]

# Standard input for the container. Optional.
input: |
  A very nice pull request.
```

### Intermediate images

It's often helpful to be able to derive a bunch of actions from a single source
image. For example, you may have an integration-specific library you need to use
with every action. Or it may be enough to just need the Python interpreter
installed in your steps.

Define an intermediate image for an integration by creating a `build.yaml` in an
`images/<image-name>` directory. This `build.yaml` has the same syntax as the
`build` property of an action. For example, a Python intermediate image
`build.yaml` might look like:

```yaml
apiVersion: build/v1
kind: PythonPackage
context: my/package
```

For simple cases (e.g., `DockerOverride`), `images/<image-name>.yaml` is
equivalent to `images/<image-name>/build.yaml` with the same semantics as
abbreviated action YAML files.

Intermediate images are never published, but they can be referenced by path (as
in the image reference to a file in `DockerOverride`) or using the special
syntax `FROM @@<image-name>` in a Dockerfile or `@@<image-name>` in an `image`
property.

### Providers

As we expand, we anticipate supporting integrations from third parties. Each
third party that wishes to make integrations available will be required to
register with our library API (out of the scope of this RFC). Providers may be
individuals or organizations. Providers are uniquely identified by a slug.

A provider slug prefixes an integration name. It is separated from the
integration name with a forward slash.

The first-party provider slug is `puppet`. We reserve the slugs `puppet`,
`puppetlabs`, and `relay`, and may reserve additional slugs at the
recommendation of Legal. For the first-party provider, including the slug in a
reference to an integration is optional. For example, both `puppet/github` and
`github` refer to the same integration.

### Referencing integrations in workflows

Currently, the only way to reference an action in a workflow is by explicitly
specifying the Docker image to run. As we develop a library API, we intend to
supplement (not remove!) this functionality with the ability to simply specify
the name of an action in the context of an integration.

As defined in a workflow file, we propose to add a new property to queries,
steps, and triggers named `action`. It is mutually exclusive with `image`. It
has the following format:

```
<integration-name>[^<channel>]/<action-name>@<action-major-version>[.<integration-version>]
```

Note that it is unnecessary to specify what type of action to reference, as that
is implied by the location within the workflow file.

For example, to use the latest `issue-create` step compatible with major version
3 on the beta GitHub release channel:

```yaml
steps:
- name: create-my-issue
  action: puppet/github^beta/issue-create@3
  spec:
    connection: !Connection {type: github, name: my-github}
    # ...
```

## Engineering-level explanation

Our goal for integrations (i.e., a package of related queries, steps, and
triggers) remains to enable a developer experience that is as efficient and
painless as possible. This RFC builds on our existing [step repository
structure](https://github.com/puppetlabs/nebula-steps), itself an evolution of
the task containers in
[nebula-tasks](https://github.com/puppetlabs/nebula-tasks/pull/66). The
foundational work in this case is the
[Spindle](https://github.com/puppetlabs/nebula-sdk#spindle) utility that
generates Dockerfiles from a `container.yaml` specification.

Of the layout and processes proposed, Spindle addresses a small part of the
builder functionality. The rest of the work is novel. We propose an iterative
implementation of this RFC.

### Integrations V1: Templates and Spindle

For our first phase, we [create a template
repository](https://help.github.com/en/github/creating-cloning-and-archiving-repositories/creating-a-template-repository)
on GitHub with stubbed examples for root-level metadata, actions, and
supplementary files (READMEs, etc.).

In addition to each action containing its corresponding metadata file, they
contain a Spindle-compatible `container.yaml` file that we use to generate
Dockerfiles. We migrate our generation and build scripts from `nebula-steps` so
we can build images using our existing tooling.

We migrate all well-understood steps from `nebula-steps` to appropriate
integrations and archive `nebula-steps`.

In this phase, all information from the metadata files is ignored, including
versions and builders. We do not yet provide backward compatibility for
integrations; this is an alpha-level implementation at best.

### Integrations V2: CLI integration

The second phase replaces our GitHub template repository with a set of Relay CLI
commands. We also deprecate Spindle in favor of Relay CLI commands.

We add the following commands to the Relay CLI (also see [RFC
0004](https://github.com/puppetlabs/nebula/blob/master/rfcs/content/0004-relay-cli/rfc.md)):

* `relay devel init [integration-name]`: Scaffolds an integration using the our
  prescribed directory layout.
* `relay devel new action -t {step,query,trigger}`: Within an integration
  layout, scaffolds a new action with the requested type.
* `relay devel generate [--write]`: Generate the Dockerfiles for each action in
  this integration from their respective builders.
* `relay devel build`: Build Docker images for each action in this integration.

  This command has a few important flags:
  * `--repository-template`: A template for constructing a Docker repository
    name from an integration and action. For example,
    `projectnebula/{{.Integration.Name }}-{{ .Action.Kind }}-{{ .Action.Name}}`
    would yield something like `projectnebula/github-step-issue-create`.
  * `--tag-template`: A template for constructing a Docker tag name from an
    integration and action. May be specified multiple times to create multiple
    tags for the same action. For example, `{{ .Action.Version }}` and `{{
    .Action.Version }}-{{ .Integration.Version }}`.
  * `--engine`: The engine to use to build the images. We'd like to support both
    Docker and Kaniko at a minimum.

  If the Dockerfiles present are not up-to-date relative to what `relay devel
  generate` would write, we present a warning when running this command.
* `relay devel publish`: An initial implementation of the command that will
  eventually submit the integration to our library. In this phase, it simply
  pushes what `relay devel build` produces to a Docker registry.

In this phase, we only support the `Docker` builder (probably without `input`,
which is somewhat more complicated to implement). We simply delete each
`container.yaml` and the top-level generation and build scripts from each
integration repository. Note that this practically locks us into a specific SDK
version (because downloading Ni is hardcoded into each Dockerfile by Spindle).

### Integrations V3: Builders and intermediate images

In this phase, we add support for our additional builders: `DockerOverride`,
`Go`, and `PythonPackage`. We also add support for building and referencing
intermediate images.

We add the following command to the Relay CLI:

* `relay devel new image`: Scaffold a new intermediate image.

We refactor our existing integrations to take advantage of intermediate images
or other builders where appropriate.

### Integrations V4: Referencing integrations in workflows

Thus far, we've been able to avoid making changes to the API or workflow
execution engine to accommodate our new integration layout. To support
referencing an action using our desired syntax, we stub a library API that
dereferences an action reference to a Docker image. This may be a simplistic
implementation of [PN-175](https://tickets.puppetlabs.com/browse/PN-175).

We amend the API and/or workflow controller to perform this dereferencing where
required.

We also update `relay devel build` and `relay devel publish` to push to the
library API by default instead of requiring a complex repository and tag
template scheme.

We add support for channels other than `stable`.

### Integrations V5: A cohesive library

We've been ignoring most of the descriptive metadata in our integrations,
copying it by hand to the library section of the website as needed. This phase
introduces a more sophisticated `relay devel publish` that submits top-level
metadata, actions as Docker images, and relevant documentation as a single
atomic unit.

We update the library section of the website to reference the library API. Our
goal for this phase is to ensure that when an integration is updated, changes
are immediately propagated across our platform.

### Integrations V6: External providers and library enhancements

We link Relay accounts to our library API enabling users to publish their own
content publicly or privately (within their account). This implementation will
be discussed in a future RFC.

## Drawbacks

No one will ever be perfectly happy with a rigorously enforced project layout.
However, in the same spirit as [gofmt](https://blog.golang.org/gofmt), we hope
integrations following this layout will be easy to maintain and uncontroversial.

Some portions of the metadata files described by this RFC stray from our guiding
principle of terseness in anticipation of future enhancements (for example,
explicit `apiVersion`s). Normally we would not put so much emphasis here, but to
be as clear to the community as possible our intent to maintain compatibility,
we've elected to address a larger surface area as part of our initial design.

## Rationale and alternatives

### Why is this design the best in the space of possible designs?

The customizability of our actions is a reasonably important differentiator for
our product. We want to balance that with a bit of rigor that lowers the
cognitive burden of structuring a development environment that creates Docker
images compatible with our execution engine.

We believe this layout is convenient, easy to reason about, and extensible
enough to handle most of our needs for those forseeable future. It includes
extension points to add new action types, and generally provides a level of
nesting that lets us incorporate as much additonal content as we wish alongside
action definitions without being overly verbose.

### What other designs have been considered and what is the rationale for not choosing them?

We started with nothing: bring your Dockerfile and get an image. After about 10
images this became painfully repetitive and managing common patterns was
impossible.

Spindle, which supports a more generic form of the builders proposed here, is
fairly useful but not sufficient in many ways. For example, while it supports
intermediate templates, it can't add content to them because it only builds a
single image and therefore only has a single build context.

We briefly considered only supporting Python, which makes the layout simpler to
reason about (provide a Python package with requisite hooks), but doing this
really reduces the flexibility of a container-based system. And since our
execution environment would require us to build containers anyway, it seemed
artifically limiting.

### What is the impact of not doing this?

We retain the current architecture of arbitrary directories containing Docker
containers that happen to work with our service. While easy to write, they're
quite difficult to maintain and they don't have the cohesive feel that an
integration package does. We don't think we'll get community involvement with
our current approach, nor will we be able to effectively promote our actions as
a library.

### What specific risks are associated with this design?

The engineering plan is complex and a complete implementation of this RFC
requires minimal stubs of several other system components to exist. Correctly
executing it will require a good deal of oversight. We also anticipate changes
to our design as we better understand customer requirements, and we worry that
we may not have properly accounted for places where those changes are likely to
occur.

## Success criteria

We will be successful if:

1. Qualitatively, as we progress through the implementation of this RFC, we find
   it increasingly easier to produce and consume actions.
2. We have a dynamically-generated library of actions that exposes all relevant
   metadata.
3. Our `relay devel` CLI command supports managing an integration layout.

## Future possibilities

* We have not discussed testing actions. This is an expansive topic. We leave it
  for a future RFC.
* While we have defined a mechanism for providing JSON Schemas for different
  components of actions, we haven't discussed how they will be used. We
  anticipate rendering documentation and automatic validation of step
  specifications at a minimum. However, we leave these specifics to a future
  RFC.
