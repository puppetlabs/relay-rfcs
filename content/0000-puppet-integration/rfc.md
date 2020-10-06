# Initial Puppet integration (2020-09-28)

## Stakeholders

* **Recommenders:** Noah Fontes
* **Agreers:** Hunter Haugen, Deepak Giridharagopal
* **Performers:** Noah Fontes, Hunter Haugen
* **Inputers:** Rick Lane, Kyle Terry
* **Deciders:** Hunter Haugen, Deepak Giridharagopal

## Problem

Puppet and Relay are currently two distinct product offerings with little
interoperability. This is not optimal for our portfolio, and we hope to address
the potential symbiotic relationship between the two platforms with an initial
alpha-quality integration strategy.

## Summary

This RFC defines a structure for a Puppet module, an auxiliary process that acts
as a sidecar to the Puppet server, and a new connection type in Relay. These
three components provide the basis for bidirectional communication between an
open-source Puppet or Puppet Enterprise (PE) install and Relay.

## Motivation

Relay provides an opportunity to bolster existing Puppet solutions with an
event-based execution model that could make operating systems supervised by
Puppet substanitally easier. We want to investigate what those opportunities
look like to make the best broad-spectrum portfolio decisions possible.

## Product-level explanation

This RFC provides an implementation plan for initially integrating with
open-source Puppet and PE. We intend to demonstrate the technical feasibility of
an integration as well as demonstrating clear value for existing Puppet users in
adopting Relay.

We will initially provide these Relay workflows to integrate with Puppet:

1. Notification to a Slack channel when a Puppet run detects configuration
   changes for a given resource or profile
1. Dry run of Puppet on a subset of nodes, manual approval, and dispatching a
   real run back to Puppet

Puppet users will be required to install a Puppet module,
[puppetlabs-relay](https://github.com/puppetlabs/puppetlabs-relay), which will
provide on-premise connectivity to the Relay environment. Within Relay, users
will need to set up workflows with push triggers to receive data from Puppet and
a special connection type to send data to Puppet.

## Engineering-level explanation

To provide bi-directional communication between on-premise Puppet installs and
the Relay SaaS service, we'll use a Puppet module that includes plugins to send
data to workflows via triggers (outbound) and a scheduled process that polls for
requests to run Puppet on specific nodes (inbound). In particular, we want to
avoid complex or undesirable networking scenarios like maintaining an
out-of-band VPN connection or opening ports on corporate firewalls.

### Module structure

Our Puppet module defines one plugin, a report processor to run Relay workflows,
and one class, `relay::bridge`, that installs the report processor (using
[`ini_setting`](https://github.com/puppetlabs/puppetlabs-inifile) or
`pe_ini_setting`) and a bidirectional process to handle requests to run Puppet.

### Report processor

The report processor sends report data to one or more workflow push triggers.
Initially, it will send a subset of report data because the complete report data
is very large. As part of this work, we will implement an offloading mechanism
to allow large event, spec, and output data to flow through to triggers and
steps seamlessly.

For customers with a large number of nodes, the report processor may be a good
forcing function to implement concurrency quotas in Relay Core. This is outside
the scope of this RFC.

### Relay connection

In Relay, we define a new connection type, `puppet`. This connection type is
unique compared to our other connections because it will generate and store a
secret API token that users will need to use to configure the Puppet module.
Like similar services that provide API tokens, when the connection is added in a
Relay UI, the API token will be returned to the user exactly once; if they
subsequently lose it, they will need to generate a new one. Like other Relay
connections, the token will be securely stored in Vault for use in steps that
integrate with Puppet installs.

In the future, we will unify push trigger tokens and Puppet connection tokens
under a single API token entity with fine-grained permissions that a Relay
administrator can configure. This is not in the scope of this RFC.

### Bridge process

The bridge process is a small script that runs as a sidecar to a PE or open
source Puppet install. It is responsible for periodically checking in with the
Relay SaaS API to find new runs to execute and to provide information about
existing runs.

When the bridge process starts, it requests the list of pending Puppet runs
associated with its configured connection API token from the Relay SaaS. Then,
for each pending run, it forks. Each fork daemonizes and the main process exits.

For a given fork, the bridge process performs the following actions:

1. Accepts the run by `POST`ing to the relevant API endpoint. If the API rejects
   this request, the process simply exits successfully with no further work to
   do (because another process has already accepted the run).
1. Using the configured execution mechanism, either for PE or open source
   Puppet, starts the Puppet run.
1. Updates the state of the run to `in-progress` in the Relay SaaS API.
1. Continues to monitor the run's progress. As a heartbeat mechanism, updates
   the state of the run in the Relay API periodically.
1. When the run completes, whether successful or not, updates the state of the
   run in the Relay API to indicate the completion.
1. Exits.

We schedule the bridge process using a systemd timer that runs on an interval
defined in the Puppet module, by default every 30 seconds.

The bridge process and relevant configuration are installed by the
`puppetlabs-relay` module automatically if a Relay connection token is provided.

#### Puppet Enterprise orchestrator

To kick off a run using PE, we use the [LTS orchestrator
API](https://puppet.com/docs/pe/2019.8/orchestrator_api_commands_endpoint.html#orchestrator_api_post_command_deploy).
This endpoint creates a job that we can monitor for updates until the run
completes on all desired nodes.

Global configuration settings:

* `orchestrator_api_url` (required): A resolvable URL to the orchestrator API.

#### Open source Puppet

For open-source Puppet, the situation is a bit more complex. There's no
well-defined inventory list or API endpoint we can use to delegate
responsibility for the run to some other process.

Instead, we provide two mechanisms to execute Puppet directly on nodes from
within the bridge process:

1. **Bolt:** We provide the node list from the run as an inventory file to Bolt
   and ask to execute `puppet agent -t` on each node.

   Configuration settings:

   * `bolt_path`: Path to the Bolt binary on the node running the bridge
     process. Defaults to `bolt` in `$PATH`.

1. **SSH:** For users that do not have Bolt installed or do not want to use
   Bolt, we execute SSH directly, in parallel, on each requested node.

   Configuration settings:

   * `openssh_client_path`: Path to the OpenSSH client binary to use to connect
     to each node. Defaults to `ssh` in `$PATH`.
   * `parallelism`: Number of concurrent runs to allow. Defaults to 5.

Global configuration settings:

* `backend`: One of `bolt` or `ssh`. Defaults to Bolt if `bolt` is in `$PATH`,
  otherwise `ssh`.
* `node_puppet_path`: Path to the Puppet binary on each node taking part in the
  run. Defaults to `puppet` in `$PATH`.

#### Detecting PE

It is possible to detect whether we use a PE compiler for a given module by
testing `is_function_available('pe_compiling_server_version')`.

### Relay API

Because this integration is alpha-quality, we want to ensure any API changes we
make are appropriately scoped to ensure we don't mislead developers. To this
end, we introduce a new API URL namespace, `/_puppet`, with the following
endpoints:

* `POST /_puppet/runs`

  Ask the bridge process to perform a Puppet run on a specified set of nodes.
  Used by a Relay step to start a Puppet run. This does not change any state on
  a Puppet node, but just makes the run request discoverable by the bridge
  process.

  If the bridge process is running on PE, `environment` is required. Otherwise,
  it is optional and will follow the usual process to discover the environment
  (either from the agent configuration on the node or an
  [ENC](https://puppet.com/docs/puppet/6.18/nodes_external.html)).

  If the bridge process is running on an open-source Puppet install, only the
  `nodes` scope is supported. Otherwise, the scope may be one of `application`,
  `nodes`, `query`, or `node_group`. See the [corresponding PE
  documentation](https://puppet.com/docs/pe/2019.8/orchestrator_api_commands_endpoint.html#orchestrator_api_post_command_deploy)
  for more information.

  Request:

  ```json
  {
    "environment": "staging",
    "scope": {
      "nodes": ["node-a.example.com", "node-b.example.com"]
    },
    "noop": false,
    "debug": false,
    "trace": false,
    "evaltrace": false
  }
  ```

  Response (201 Created):

  ```json
  {
    "id": "8e659097-254a-4b27-bb93-88efea9a9f0e"
  }
  ```

* `GET /_puppet/runs`

  Retrieve the list of runs currently outstanding that have not completed. Used
  by the bridge process to discover new runs to start.

  Response (200 OK):

  ```json
  {
    "runs": [
      {
        "id": "8e659097-254a-4b27-bb93-88efea9a9f0e",
        "state": {
          "status": "pending"
        },
        "environment": "staging",
        "scope": {
          "nodes": ["node-a.example.com", "node-b.example.com"]
        }
      }
    ]
  }
  ```

* `GET /_puppet/runs/{runId}`

  Retrieve the run with the given ID. Used by a Relay step to wait for a run to
  complete and to provide relevant debugging information to the Relay user
  executing the workflow.

  Response (200 OK):

  ```json
  {
    "id": "8e659097-254a-4b27-bb93-88efea9a9f0e",
    "state": {
      "status": "in-progress",
      "job_id": "https://orchestrator.example.com:8143/orchestrator/v1/jobs/1234",
      "next_update_before": "2020-09-30T15:00:00Z",
      "updated_at": "2020-09-30T14:55:00Z"
    },
    "environment": "staging",
    "scope": {
      "nodes": ["node-a.example.com", "node-b.example.com"]
    }
  }
  ```

* `POST /_puppet/runs/{runId}/accept`

  Take ownership of a given run. Used by the bridge process to prevent race
  conditions. A call to this method succeeds exactly once for any given run and
  returns 202 Accepted. If this resource has been accessed before, it returns
  409 Conflict.

* `PUT /_puppet/runs/{runId}/state`

  Update the state of a given run. Used by the bridge process to inform the SaaS
  service of progress on a run.

  If the status is set to `in-progress`, the bridge process must provide an
  additional field, `next_update_before`, to indicate when it intends to provide
  another state update.

  If the status is set to `complete`, the bridge process may optionally select
  to provide an outcome other than `finished`, such as `failed`, to indicate
  that the user may need to perform additional steps to investigate what
  happened as part of the run.

  If the bridge process is running as part of a PE install, it is expected to
  also provide the orchestrator job ID as part of the request.

  The API responds with 202 Accepted to state changes.

  Requests:

  * To update the status to when a run is in progress:

    ```json
    {
      "status": "in-progress",
      "job_id": "https://orchestrator.example.com:8143/orchestrator/v1/jobs/1234",
      "next_update_before": "2020-09-30T15:00:00Z"
    }
    ```

  * To indicate that a run is complete (finished or failed):

    ```json
    {
      "status": "complete",
      "job_id": "https://orchestrator.example.com:8143/orchestrator/v1/jobs/1234",
      "outcome": "finished"
    }
    ```

### Relay Puppet integration

We'll create a new Relay integration,
[`Puppet`](https://github.com/relay-integrations/relay-puppet), with the following steps:

* `run`: Run Puppet on a set of nodes.

  This step `POST`s to the `/_puppet/runs` endpoint in the Relay API to create a
  new run request. It then monitors the state of the run until it misses a
  heartbeat more than `timeoutAfterMissedHeartbeats` times (a multiple of the
  duration between `updated_at` and `next_update_before` in the state
  information) or the run completes.

  Spec:

  ```yaml
  connection: !Connection {type: puppet, name: my-puppet-connection}
  timeoutAfterMissedHeartbeats: 3
  environment: staging
  scope:
    nodes:
    - node-a.example.com
    - node-b.example.com
  noOp: false
  debug: false
  trace: false
  evalTrace: false
  ```

  Outputs:

  ```yaml
  jobID: https://orchestrator.example.com:8143/orchestrator/v1/jobs/1234
  outcome: finished
  ```

## Drawbacks

This design is intentionally _very_ Puppet-specific and even then does not
provide integration with much of any PE or Puppet functionality beyond run
execution. So this comes with the cost of excluding early users that might seek
a more generic on-premise solution.

Even in its relative simplicity, it requires a decent amount of work to add and
maintain new API endpoints and connection types in the Relay service. I worry
somewhat about making radical changes to the connection types in particular in
the future, and we may need to consider implementing versioning for connections
sooner than anticipated.

## Rationale and alternatives

### Why is this design the best in the space of possible designs?

Truthfully, this likely isn't the best technical design for this need. But it is
cheap to design, implement, test, validate, and support. In this case, since our
primary objective is to validate the product integration and use case, we accept
the trade-off in reusability and extensibility.

### What other designs have been considered and what is the rationale for not choosing them?

An obvious design choice is the full implementation of the on-premise Relay Core
offering. However, we don't yet have a clear way to bundle it with PE (much less
open source Puppet) as it requires an operating Kubernetes cluster to work. In
the future, we do see this being an appealing choice for PE customers,
especially those with strict data locality requirements.

We considered a long-running Web server or client process that we proxy to our
service to provide direct RPCs. This could be, for example, something like
[Inlets](https://github.com/inlets/inlets) or a WebSocket-based protocol.
However, we decided the maintenance requirements and design complexity were too
high for this proof of concept.

### What is the impact of not doing this?

We may not get the initial information we need to validate our Puppet
integration use case. We might also make poor decisions in the future when
deciding how to implement our generic on-premise solution.

### What specific risks are associated with this design?

As stated, this is intended to be alpha-quality software. We have a slight risk
of not adequately communicating this and being forced to maintain the APIs
defined here beyond their usable lifetimes.

While this design does not require firewall changes for most on-premise networks
that allow outbound internet traffic, it cannot work in a fully air-gapped
environment. It also exposes internal PE node data to our SaaS service if the
report processor plugin is used. Some customers may not be able to use this
module for regulatory or compliance reasons.

## Success criteria

When this RFC is fully implemented, we will be able to ask Relay to run Puppet
in on-premise environments and make decisions based on the result of the run. We
will also be able to send reports from Puppet runs to Relay to kick off
workflows.

## Unresolved questions

* Is it possible to automatically discover the PE orchestrator URL?
* Can we automatically place the module on the correct PE node using a
  preconfigured node group?
