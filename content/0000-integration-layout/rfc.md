# Layout of an integration (2020-04-07)

## Stakeholders

* **Recommenders:** Noah Fontes
* **Agreers:** Brad Heller, Kenaz Kwa
* **Performers:** [...]
* **Inputers:** Rick Lane, Sebastian Prokuski, Michael Sigler, Geoff Woodburn
* **Deciders:** Brad Heller, Kenaz Kwa, Geoff Woodburn

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
┇ ┇ ┇ ┣ 📜 Dockerfile
┇ ┇ ┇ ┣ 📜 spec.schema.json
┇ ┇ ┇ ┣ 📜 outputs.schema.json
┇ ┇ ┇ ┗ 📜 step.yaml
┇ ┇ ┣ 📜 pull-request-create.yaml
┇ ┇ ┗ 📜 pull-request-merge.yaml
┇ ┗ 📂 triggers
┇   ┣ 📜 issue-opened.yaml
┇   ┣ 📜 pull-request-merged.yaml
┇   ┣ 📜 pull-request-opened.yaml
┇   ┣ 📜 pushed.yaml
┇   ┗ 📜 release-published.yaml
┣ 📂 images
┇ ┣ 📂 pr
┇ ┇ ┗ 📜 Dockerfile
┇ ┗ 📂 webhook
┇   ┗ 📜 Dockerfile
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

# The schema kind. Optional if the filename is exactly "integration.yaml" or
# "integration.json". If specified, must be exactly the string
# "Integration".
kind: Integration

# The integration slug. Required, but may be optional in the future if it can
# be inferred from a package filename, Git repository data, or parent directory
# name.
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

# Free-form labels to help people find this integration. Optional.
tags:
- git
- source control
```

A minimal example to demonstrate what can be safely omitted:

```yaml
apiVersion: integration/v1
name: github
version: v20200408
summary: GitHub
license: Apache-2.0
owner:
  name: Puppet, Inc.
  email: relay@puppet.com
  url: https://puppet.com
```

### README

The `README.md` file in the root of the integration contains an overview of the
integration, including implementation examples, in markdown format.

### Actions

Each kind of action (query, step, and trigger) has its own metadata format that
share a common structure to identify the Docker image to use as a base for the
action.

Each action in an integration has its own directory corresponding to the slug
for the action. This directory contains metadata for the action as well as any
files needed for the action.

As a convenience to authors, we provide one normalization feature: if no extra
files are needed for a given action, the structure
`actions/<type-plural>/<name>/<type>.yaml` may be abbreviated to
`actions/<type-plural>/<name>.yaml`. If both the abbreviated YAML file and
directory exist simultaneously, the directory takes priority and a warning shall
be emitted by developer tooling.

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

#### Common metadata

All action metadata has the following fields:

```yaml
# The schema version. Required. Must be exactly the string "integration/v1".
apiVersion: integration/v1

# The schema kind. Optional if possible to infer from the location in the
# actions subdirectory of the integration. If specified, must be one of "Query",
# "Step", or "Trigger", corresponding to its directory location.
kind: Step
# kind: Query
# kind: Trigger

# The name of the action. Optional. If specified, must be exactly the name of
# the directory containing the action.
name: issue-create

# The version of the action. Required. Must be an integer.
version: 3

# High-level phrase describing what this action does. Required.
summary: Create an issue

# Single-paragraph explanation of what this action does in more detail.
# Optional. Markdown.
description: |
  Creates a new issue (if issues are enabled).

# The mechanism to use to construct this step. Required. Must be an action
# builder. See the Builders section below.
build:
  # The schema version for builders. Optional, implied to be "builder/v1" if
  # not specified. We may consider supporting custom third-party builders in
  # the future.
  apiVersion: builder/v1

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
version: 1
summary: Create an issue
build:
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

# The Docker build context. Defaults to the current directory.
context: .

# Additional Docker build arguments.
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

For example, to use the latest `create-issue` step compatible with major version
3 on the beta GitHub release channel:

```yaml
steps:
- name: create-my-issue
  action: puppet/github^beta/create-issue@3
  spec:
    connection: !Connection {type: github, name: my-github}
    # ...
```

## Engineering-level explanation

[Explain, in detail, the technical manifestation of this change. Detail the
scope of work including time or size estimates. Include any potential conflicts
with features that are currently underway.]

## Drawbacks

## Rationale and alternatives

### Why is this design the best in the space of possible designs?

[...]

### What other designs have been considered and what is the rationale for not choosing them?

[...]

### What is the impact of not doing this?

[...]

### What specific risks are associated with this design?

[...]

## Success criteria

[What will we observe when this change is completely implemented? What metrics
can we use to measure success? Can we integrate any learnings into future
processes?]

## Unresolved questions

* [Insert a list of unresolved questions and your strategy for addressing them
  before the change is complete. Include potential impact to the scope of work.]

## Future possibilities

### Library API

We imagine a library API for looking up actions. This is outside the scope of
this RFC. As a taste, imagine the following API URL:

```
https://library.relay.sh/integrations/puppet/github/channels/beta/steps/create-issue@3
```
