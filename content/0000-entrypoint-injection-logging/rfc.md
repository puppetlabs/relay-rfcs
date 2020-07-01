# Entrypoint injection and tenant pod logging (2020-06-30)

## Stakeholders

* **Recommenders:** Noah Fontes
* **Agreers:** Rick Lane, Kyle Terry
* **Performers:** Rick Lane, Kyle Terry
* **Inputers:** Brad Heller, Deepak Giridharagopal
* **Deciders:** Brad Heller, Rick Lane, Kyle Terry

## Problem

Our logging infrastructure is based on a resource ownership model that implies
supervision of every pod from beginning to end (because this is how Tekton
works). However, with the introduction of Knative Serving, we no longer have
that luxury. As a result, we do not retain logs for webhook triggers.

This RFC also begins to address a number of other log infrastructure problems.
For example, we cannot filter sensitive data in the controller because it does
not have access to relevant secrets.

## Summary

This RFC introduces a new container management process called an
**entrypointer** that replaces the entrypoint for containers we manage. It also
adds support for receiving log streams to the metadata API.

## Motivation

[Why should we include this change in the product? How does it solve the problem
described? How does it help customers be more successful? Include rationale for
prioritizing this change in a product backlog.]

## Product-level explanation

[Explain this change in a way that our product team can understand it and pitch
it to customers for feedback. Make heavy use of examples and ensure you
introduce any new concepts at a high level.]

## Engineering-level explanation

### Metadata API

We propose to add support to the metadata API to upload logs to a configurable
storage backend. Initially, we will only support Google Cloud Storage and the
filesystem (the two storage backends already supported by Horsehead).

#### HTTP resources

We add the following HTTP resources to the metadata API:

* `POST /logs`

  Create a new log stream.

  Request:

  ```json
  {
    "name": "stdout",
  }
  ```

  Response (200):

  ```json
  {
    "id": "9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08",
    "name": "stdout"
  }
  ```

  The ID may be randomly generated and stored (e.g., in a ConfigMap) or produced
  by combining information only known to the metadata API and the requested
  name. In any case, it should not be possible to guess a stream's ID if you
  only know its name.

* `POST /logs/{streamId}/data`, `PATCH /logs/{streamId}/data`

  Append arbitrary binary data to the log stream. We anticipate extending this
  endpoint in the future to support structured logging using a different media
  type (for example, using `ni log`). For now, use `Content-Type:
  application/octet-stream` and send log data directly.

  The server responds with 202.

* `POST /logs/{streamId}/markers`

  Mark a segment of the logs. The only available marker is `dropped`, but an
  array is used for future extension. Ranges should follow the format of the
  `Content-Range` header.

  Request:`

  ```json
  {
    "range": "bytes 0-5/*",
    "markers": [
      "dropped"
    ]
  }
  ```

  The server responds with 202.

#### Ordering challenges

It may be desirable to submit log data in parallel. In this case, you must track
byte ranges on the client and use the `Content-Range` header with the `PATCH`
method to specify the byte range of the data being submitted. The metadata API
should make a best-effort attempt to keep content ordered, for example, by using
[composite objects](https://cloud.google.com/storage/docs/composite-objects) as
needed. If the log data cannot be ordered properly, simply append it to the
storage.

#### Performance implications

Log processing is typically a high volume operation. The metadata API service is
not optimized for this right now, so we'll need to ensure we attach appropriate
metrics to it.

We highly encourage clients to submit logs with `Content-Encoding: gzip` when
possible.

Additionally, clients should be prepared to receive 429 (Too Many Requests)
responses with a
[`Retry-After`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Retry-After)
header. Additional log data will not be accepted from the client until the
specified time has elapsed.

#### Dropped logs

If, for whatever reason, a client drops a segment of log data it otherwise
wanted to submit, it should report it to the metadata API as soon as possible
using the `markers` resource. This resource will never apply backoff rules.

Marking unavailable logs as dropped allows the metadata API to quickly move past
any pending ordering issues.

### Entrypointer

To provide the metadata API with logs from a tenant container, we inject a
management process into every container. Although we anticipate the scope of
this process to increase with time, initially we expect it to roughly perform
the following actions in order:

* Set up a [process](https://golang.org/pkg/os/exec/#Cmd) pointing at the
  original entrypoint of the container.
* Create two log streams, `stdout` and `stderr`, in the metadata API.
* Create Goroutines to copy the `Stdout` and `Stderr` of the process to the
  metadata API's log data resource.
* Execute the underlying process.

Additional functionality of the entrypointer is out of the scope of this RFC.

It may be helpful to reference Tekton's
[entrypointer](https://github.com/tektoncd/pipeline/blob/v0.13.2/cmd/entrypoint/main.go)
(and [operator
configuration](https://github.com/tektoncd/pipeline/blob/v0.13.2/pkg/pod/entrypoint.go)).

The source code for this command is to be made public in the
[relay-core](https://github.com/puppetlabs/relay-core) repository.

#### Teeing logs

One benefit of the entrypointer is that we can direct log data out of our own
cluster logs. We don't particularly want to be handling customer logs
out-of-band; having them searchable in, for example, Google Cloud Logging is an
unfortunate artifact of our current implementation.

To that end, we should not tee standard output/error of the tenant process to
standard output/error of the container.

#### Determining the original entrypoint

If the CRD defines the `command` field for a container, the original entrypoint
is simply that value. Otherwise we must look up the entrypoint from the upstream
image. We perform some of this work already in the API, but we'll likely want to
also support it in the operator. The [Tekton
implementation](https://github.com/tektoncd/pipeline/blob/v0.13.2/pkg/pod/entrypoint_lookup.go)
is a good reference.

#### Injection

Although the entrypoint process itself is small in both size and complexity,
injecting it at the correct point in the processing for Tekton and Knative
Serving is more complicated.

In every case, we set the container command for both Tekton and Knative Serving
to our entrypoint binary. For example, assuming our binary exists in the
container at `/relay/lib/entrypoint` and the original concatenated entrypoint
and arguments are `["/bin/sh", "-c", "echo hi"]`, we'd set the container command
and args to something like:

```yaml
command: /relay/lib/entrypoint
args: [--, /bin/sh, -c, echo hi]
```

There are a few steps required to make the entrypointer available in each pod.

First, we need to set up a PVC with the required files. We amend our Tenant CRD
to allow us to specify a PVC template. If not specified, we choose a default
template with, e.g., no storage class specified.

```go
type EntrypointInjection struct {
  // +optional
  VolumeClaimTemplate *corev1.PersistentVolumeClaim `json:"volumeClaimTemplate,omitempty"`
}

type TenantSpec struct {
  // ...

  // +optional
  EntrypointInjection EntrypointInjection `json:"entrypointInjection,omitempty"`
}
```

Note that any template must set `accessModes` to `[ReadOnlyMany]`. No other
values are acceptable.

When we create or update a tenant, we create the PVC in the tenant namespace. We
also create and monitor a
[Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/) to
populate the PVC using a tools container.

We report the status of the tools PVC in the tenant `status` conditions.

Once the PVC is created, the relay operator mutating webhook modifies all pods
to include a volume that references the PVC and a mount at `/relay`.

#### Versioning

We expect the Relay operator to take a version tag for the tools container to
deploy to tenants. When the version tag changes, the operator must cycle through
all existing tenants and create a new Job to create a new PVC. The operator
should not replace the old PVC as it may be in use by existing pods.

#### Tenants and WorkflowRuns

The legacy WorkflowRun CRD does not currently reference a tenant.

TK: How do we want to update this?

### API

We modify the API to stream logs directly from GCS. This removes one
long-standing tech debt item that allows the API to directly access pod logs.

We add the following resource:

* `GET /api/workflows/{workflowName}/triggers/{triggerSyncId}/logs`

  Retrieve the log for a trigger with the given synchronization ID. This
  functionally behaves similarly to the step logs resource, but a trigger is
  always considered to be "in progress," so trigger logs can always be followed.

We also modify the `WorkflowTriggerState` schema to include the an `id` field
containing the synchronization ID.

## Drawbacks

[Why should we _not_ include this change in the product?]

## Rationale and alternatives

### Why is this design the best in the space of possible designs?

TK: Expand on additional value of injector.

### What other designs have been considered and what is the rationale for not choosing them?

TK: External monitoring of pods.

TK: Why we use a PVC instead of some other method.

To place the binary in the container, we need to either copy files into it
using, e.g., an init container and shared volume or have the files pre-allocated
in, e.g., a PersistentVolumeClaim or ConfigMap.

Tekton doesn't support init containers, but we could relatively easily use
[workspaces](https://tekton.dev/docs/pipelines/workspaces/). An init Task would
be a dependency for every other Task in the Pipeline. This introduces some
complexity around scheduling; namely, it forces the entire PipelineRun to run on
the same node (this is a general limitation of PVCs). It also means that we
can't use our own affinities in the future.

Unfortunately, Knative Serving doesn't provide any useful functionality to run
init containers or inject PVCs into pods. A [discussion in their issue
tracker](https://github.com/knative/serving/issues/4579) points to an
implementation by the Kubeflow folks that uses a mutating webhook to get what
they (and we) want. Although I'm generally wary of bypassing APIs like this,
we're already doing it for a number of other pod fields and it works well.

Finally, etcd's configuration limits the size of ConfigMaps to 1MB, so while we
could probably pull off injecting a small binary written in C or C++, we can't
use any of the languages we're familiar with as a team.

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

[Does this change unlock other desired functionality? Will it be highly
extensible, very constrained, or somewhere in between?]
