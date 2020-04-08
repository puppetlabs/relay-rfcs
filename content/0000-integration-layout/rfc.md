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

[...]

```
📦 github
┣ 📂 actions
┇ ┣ 📂 queries
┇ ┣ 📂 steps
┇ ┇ ┣ 📂 issue-create
┇ ┇ ┇ ┣ 📜 Dockerfile
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
# version:
#   source: git-tag
# version:
#   source: file
#   file: VERSION

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
# license:
#   source: file
#   file: LICENSE

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

[...]

### Images

[...]

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

We propose to add a new property to queries, steps, and triggers, mutually
exclusive with the existing `image` property, named `action`. It has the
following format:

```
<integration>[#<channel>]/<name>[@<version-pattern>]
```

For example, to use the latest `create-issue` step compatible with major version
3 on the beta GitHub release channel:

```yaml
steps:
- name: create-my-issue
  action: puppet/github#beta/create-issue@3
  spec:
    auth: !Connection github
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
