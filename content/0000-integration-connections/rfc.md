# Connections in integrations (2021-06-15)

## Stakeholders

* **Recommenders:** Noah Fontes
* **Agreers:** Kenaz Kwa
* **Performers:** [...]
* **Inputers:** Hunter Haugen, Rick Lane, Luke Lengl, Jesse Scott
* **Deciders:** Kenaz Kwa, Rick Lane

## Problem

Connections are currently one of the few opaque parts of the Relay ecosystem
from a customer's perspective. Connections are authoritatively defined by the
Relay API by three schemas:

* An input schema, which defines what data a user needs to provide to establish
  a connection
* A storage schema, which defines the fields available to a step or trigger
  consuming the connection
* A "display-once" schema, which allows the API to show a constructed value once
  as part of the response when creating the connection

Unfortunately, none of those schemas are directly exposed to users, and even the
input schema is basically rewritten by the web frontend (so cannot be ported to
the CLI, for example).

## Summary

This RFC amends [RFC 0006](../0006-integration-layout/rfc.md) to add a new
component to our standard integration directory structure for connections. We
also propose a migration strategy for existing connections.

## Motivation

By providing at least the storage schema of connections to end users, we can
reduce confusion about what fields are available for use in a workflow. However,
we also have more opportunities here to allow users to start defining their own
connections without needing to go through the heavy-handed process of having our
engineering teams develop API code to implement them.

## Product-level explanation

### Connection metadata

We introduce a new directory, `connections`, a sibling to `steps` and triggers`,
to the integration directory structure. A newly created integration might now
look like:

```
📦 my-integration
┣ 📂 connections
┣ 📂 queries
┣ 📂 steps
┣ 📂 triggers
┣ 📜 README.md
┗ 📜 integration.yaml
```

For our GitHub example:

```
📦 github
┣ 📂 connections
┇ ┣ 📂 token
┇ ┇ ┣ 📜 storage.schema.json
┇ ┇ ┣ 📜 display-once.schema.json
┇ ┇ ┗ 📜 connection.yaml
┇ ┗ 📜 user-password.yaml
┣ 📁 queries
┣ 📁 steps
┣ 📁 triggers
┣ 📜 README.md
┗ 📜 integration.yaml
```

The connections directory contains zero or more connection metadata directories.
Each connection metadata directory must have a file called `connection.yaml`
defining the connection. The `connection.yaml` file has the following fields:

```yaml
# The schema version. Required. Must be exactly the string "integration/v1".
apiVersion: integration/v1

# The schema kind. Required. Must be exactly "Connection".
kind: Connection

# The name of the connection. Required. Must be exactly the name of the file
# without the ".yaml" suffix or version identifier, if present.
name: token

# The version of the connection. Required. Must be an integer. If specified in
# file name, must be exactly the version in the file name.
version: 1

# High-level phrase describing this connection type. Required.
summary: GitHub token

# A paragraph or two describing this connection type. Optional. Markdown.
description: |
  This connection allows a step or trigger to access the GitHub APIs using an
  OAuth 2 token or PAT.

schemas:
  # This schema defines the shape of this connection when provided to a
  # container image.
  storage:
    $schema: http://json-schema.org/draft-07/schema#
  #storage:
  #  source: file
  #  file: storage.schema.json

  # The schema for any information that should be shown to the user when the
  # connection is created.
  displayOnce:
    $schema: http://json-schema.org/draft-07/schema#
  #displayOnce:
  #  source: file
  #  file: display-once.schema.json

# It is the responsibility of providers to map credential information to the
# storage and display-once schemas. Each provider defines a mechanism for
# configuring and evaluating a connection, and a user selects exactly one when
# setting up a connection for the first time. Required.
providers:
-
  # The name of this provider. Used as an internal identifier. Must be unique
  # among providers in the file. Required.
  name: pat

  # A high-level summary for this provider to use when presenting it as an
  # option to the user.
  summary: Personal access token

  # A further description to present to users.
  description: |
    Use a GitHub personal access token to authenticate.

  # A source defines the mapping of input data to the storage and display-once
  # schemas.
  source:
    # The schema version for the source. Required. For now, must be the exact
    # string "connection-source/v1". We may consider supporting custom
    # third-party sources in the future.
    apiVersion: connection-source/v1

    # The source to use. Required.
    kind: Form
```

Like image metadata, connection metadata is versioned, and the version must be
incremented when backward-incompatible changes are made, like changing the
storage or display-once schemas. To define multiple versions simultaneously, we
use the same file structure, e.g. `token@1.yaml` and so on.

### Built-in sources

Initially, we'll provide a handful of built-in sources, and we expect to add
others in the future.

#### Form

The form source maps the storage schema to an input form. That is, each field of
the storage schema becomes an input that the user is able to specify. In use,
this will work like the form builder for step specifications, and probably can
leverage the same system for configuration.

#### OAuth 2.0

The OAuth 2.0 source puts the user through an OAuth 2.0 or OpenID Connect
authorization code flow.

Because we don't want to lock users into the SaaS product when they're using
integrations (i.e., these connection definitions should be fully supported using
Relay Core), we provide a few shortcuts here to allow authors to specify
commonly-used servers, but leave most of the processing of them to an external
solution.

To use the OAuth 2.0 source, start with the following:

```yaml
apiVersion: connection-source/v1
kind: OAuth2

# Scopes that must always be requested for this connection to work.
scopes:
- 'read:user'
- 'user:email'
```

Then you'll need to choose a mechanism for authorizing the user:

* **Custom server:**

  ```yaml
  server:
    authCodeURL: https://github.com/login/oauth/authorize
    tokenURL: https://github.com/login/oauth/access_token
  ```

* **OpenID Connect:**

  ```yaml
  server:
    # The issuer URL contains a file at the path
    # .well-known/openid-configuration that informs the rest of the setup
    # process.
    issuerURL: https://accounts.google.com
  ```

  Using OpenID Connect as the authorization mechanism will always imply `openid`
  and `offline_access` scopes.

* **Short name:** We define a few short names that expand to a complete OAuth
  2.0 or OpenID Connect configuration. These are equivalent to specifying the
  full configuration, but are less prone to accidental input errors.

  * `bitbucket`
  * `github`
  * `gitlab`
  * `google`
  * `microsoft`
  * `slack`

  To use one:

  ```yaml
  server: github
  ```

* **Partially user-selected:** Some use cases, such as GitHub Enterprise,
  require the user both to provision a client ID and secret themselves and to
  configure some component of the authorization server URLs themselves.
  Therefore, we support partially configuring any part of the server or omitting
  it entirely.

  For example:

  ```yaml
  server:
    authCodeURL: login/oauth/authorize
    tokenURL: login/oauth/access_token
  ```

  In this case, because an authority is not present, a user will be prompted to
  provide a base URL for the authorization server.

### Testing a connection

Over time, a connection that was initially validated may become invalid: for
example, API keys might be deleted or OAuth 2.0 authorization revoked upstream
of the connection.

For OpenID Connect-authorized connections, the URL at `userinfo_endpoint` will
always be used if it is present. For other authorization mechanisms, or if the
OpenID Connect server does not provide that endpoint, you may also define a
`probe`:

```yaml
probe:
  url: https://api.github.com/user
  headers:
    Accept: application/vnd.github.v3+json
```

For OAuth 2.0 connections, the access token will be checked to verify that it is
a `Bearer` token, and then per [IETF RFC
6750](https://datatracker.ietf.org/doc/html/rfc6750) it will be sent in the
using the `Authorization` request header with the scheme set to `Bearer`. It is
not possible to define other schemes at this time, and `token_type`s other than
`Bearer` are not supported.

If no test probe is available, the connection availability will be updated only
when the connection data is retrieved for use. In some cases, it may be
impossible to determine whether the connection is usable, so we recommend always
defining a `testProbe` if possible.

[This is complicated for form sources. We need some way to map arbitrary fields
from the form onto the request. Any ideas?]

### Authenticated subject metadata

It is often helpful to display information about the subject (user)
authenticated with a connection. For OpenID Connect, this information comes from
the ID token or the `userinfo_endpoint`. Otherwise, it is possible to map
information from the probe onto [OpenID Connect-compatible userinfo
claims](https://openid.net/specs/openid-connect-core-1_0.html#StandardClaims).

This mapping supports [IETF RFC
6901](https://datatracker.ietf.org/doc/html/rfc6901) (JSON pointer) syntax to
select fields from a JSON response and
[XPath](https://www.w3.org/TR/2017/REC-xpath-31-20170321/) syntax to select
fields from an XML response. Other media types are not supported and cannot be
used.

For example:

```yaml
probe:
  userInfoMapping:
    sub: /login
    name: /name
    profile: /html_url
    picture: /avatar_url
    email: /email
```

### Referencing connections

We need to be able to reference connection types in the schemas for steps and
triggers. Currently, we do this using the special keyword
`x-relay-connectionType`. The value for this property must be one of the opaque
types exposed by the API.

We extend this property with the ability to reference connections from
in-library integrations (the presence of a slash indicates this). The reference
syntax is the same as the container image referencing syntax defined in RFC
0006.

For example:

```yaml
x-relay-connectionType: github/token@1
```

### Compatibility

We'll continue supporting the existing API types as well, incrementally mapping
them to integration connections. This should prevent any issues with existing
in-use connections.

### Referencing connections in workflows

With the compatibility shim in place, we can continue to reference connections
using the same mechanism as before, e.g., using the expression syntax
`${connections.github.my-connection}`. The new reference syntax will also be
supported as a type: `${connections.'github/token@1'.my-connection}`.

## Engineering-level explanation

[Explain, in detail, the technical manifestation of this change. Detail the
scope of work including time or size estimates. Include any potential conflicts
with features that are currently underway.]

### Deliverables

[If the scope of the work in this RFC is to take longer than a single
engineering sprint, include specific deliverables, ideally one or more for each
sprint. Ensure each deliverable represents a complete, usable piece of
functionality, not just a work stream or list of tasks.]

* [Insert date of deliverable, in T+*X* weeks format]: [Describe the
  deliverable, including changes and refactors required from previous
  deliverables.]

### Operational impact

[Does this work require changes to our infrastructure or operations? Include any
increase or decrease to security surface areas or additional requirements for
system metrics and monitoring.]

## Rationale and alternatives

### Why is this design the best in the space of possible designs?

[Describe the benefits, both to product and engineering, by implementing this
solution. Include a comparison of alternative designs as part of the narrative
if possible.]

### What is the impact of not doing this?

[...]

## Drawbacks

[Why should we _not_ include this change in the product? What specific risks are
associated with this design?]

## Success criteria

[What will we observe when this change is completely implemented? What product
metrics can we use to measure success? Can we integrate any learnings into
future processes?]

## Unresolved questions

* [Insert a list of unresolved questions and your strategy for addressing them
  before the change is complete. Include potential impact to the scope of work.]

## Future possibilities

[Does this change unlock other desired functionality? Will it be highly
extensible, very constrained, or somewhere in between?]
