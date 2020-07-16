# Container entrypoints (2020-07-15)

## Stakeholders

* **Recommenders:** Noah Fontes
* **Agreers:** Rick Lane, Kyle Terry
* **Performers:** Rick Lane
  ([PN-1129](https://tickets.puppetlabs.com/browse/PN-1129))
* **Inputers:** Brad Heller
* **Deciders:** Brad Heller, Rick Lane, Kyle Terry

## Problem

A number of technical roadmap items require some degree of interoperability
within a container instead of in their periphery or as part of external
supervision. We don't want to offload that complexity onto users, so we need to
put part of our control plane into users' containers.

## Summary

This RFC introduces a container management process called an **entrypointer**
that replaces the existing entrypoint for tenant containers we manage.

## Motivation

Having a supervisor process in a tenant container lets us perform arbitrary
pre-run and post-run operations.

## Product-level explanation

Although this change does not in itself introduce any user-facing functionality,
it enables future support for all of the following features:

* Automatic injection of the `ni` binary
* Setting and unsetting environment variables, including from the spec
* Handling signals to PID 1 (e.g., `ni log fatal`)
* Automatic streaming of standard output and standard error as logs
* User-defined scripts that run before the container starts
* Handling exit codes from the delegate entrypoint to enable the [new step
  execution flow](../0005-step-execution-outcomes/rfc.md)

## Engineering-level explanation

Our initial entrypointer is a shim over the delegate tenant command and
arguments. It will:

* Set up a [process](https://golang.org/pkg/os/exec/#Cmd) pointing at the
  original entrypoint of the container.
* Execute the underlying process.

It may be helpful to reference Tekton's
[entrypointer](https://github.com/tektoncd/pipeline/blob/v0.13.2/cmd/entrypoint/main.go)
(and [operator
configuration](https://github.com/tektoncd/pipeline/blob/v0.13.2/pkg/pod/entrypoint.go)).

The source code for this command is to be made public in the
[relay-core](https://github.com/puppetlabs/relay-core) repository.

### Determining the original entrypoint

If the CRD defines the `command` field for a container, the original entrypoint
is simply that value. Otherwise we must look up the entrypoint from the upstream
image. We perform some of this work already in the API, but we'll likely want to
also support it in the operator. The [Tekton
implementation](https://github.com/tektoncd/pipeline/blob/v0.13.2/pkg/pod/entrypoint_lookup.go)
is a good reference.

### Injection

Although the entrypoint process itself is small in both size and complexity,
injecting it at the correct point in the processing for Tekton and Knative
Serving is more complicated.

In every case, we set the container command for both Tekton and Knative Serving
to our entrypoint binary. For example, assuming our binary exists in the
container at `/var/lib/puppet/relay/entrypoint` and the original concatenated
entrypoint and arguments are `["/bin/sh", "-c", "echo hi"]`, we'd set the
container command and args to something like:

```yaml
command: /var/lib/puppet/relay/entrypoint
args: [--, /bin/sh, -c, echo hi]
```

There are a few steps required to make the entrypointer available in each pod.

First, we need to set up a PVC with the required files. We amend our Tenant CRD
to allow us to specify a PVC template. If not specified, we choose a default
template with, e.g., no storage class specified.

```go
type ToolInjection struct {
  // VolumeClaimTemplate is an optional definition of the PVC that will be
  // populated and attached to every tenant container.
  //
  // +optional
  VolumeClaimTemplate *corev1.PersistentVolumeClaim `json:"volumeClaimTemplate,omitempty"`
}

type TenantSpec struct {
  // ...

  // +optional
  ToolInjection ToolInjection `json:"toolInjection,omitempty"`
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
to include a volume that references the PVC and a mount at
`/var/lib/puppet/relay`.

#### Init container contingency

If, for any reason, PVCs don't provide the functionality we want, we can fall
back to the slower but still reasonable init container strategy successfully
used by Tekton. In this case, we would simply use the mutating webhook to inject
an init container with our tooling script and a shared `emptyDir` volume to
every relevant pod.

### Versioning

We expect the Relay operator to take a version tag for the tools container to
deploy to tenants as a command-line argument or configuration value. When the
version tag changes, the operator must cycle through all existing tenants and
create a new Job to create a new PVC. The operator should not replace the old
PVC as it may be in use by existing pods.

### Tenants and WorkflowRuns

The legacy WorkflowRun CRD does not currently reference a tenant. To take
advantage of the PVC, we need to amend it. To maintain backward compatibility
while we migrate, WorkflowRuns without a tenant use the old mechanism for log
processing (the operator handles uploading the logs).

```go
type WorkflowRunSpec struct {
  // ...

  // +optional
  TenantRef *corev1.LocalObjectReference `json:"tenantRef,omitempty"`
}
```

In the future, we intend to migrate the `nebula.puppet.com/v1` WorkflowRun type
to `relay.sh/v1beta1` Run. This change is outside the scope of this RFC, but at
that time the tenant will become a required field and the reconciliation process
will mirror webhook triggers.

## Drawbacks

There is some operational complexity associated with building and mounting a PVC
per-tenant. It also consumes storage resources (i.e., costs real money).

## Rationale and alternatives

### Why is this design the best in the space of possible designs?

This design tries to optimize flexibility to implement planned features and
technical complexity. I designed it with a hard requirement that we not put
further burden on users to migrate their workloads under a framework or SDK we
provide.

The open-ended design of a tooling populator container and entrypoint process
give us a good deal of control over execution flow within a single tenant
container. We hope that the PVC mounting strategy will not be overly burdensome
technically.

We also have some prior art to work from, since Tekton uses a mechanism very
similar to this (albeit with an init container) to inject functionality into
task containers.

### What other designs have been considered and what is the rationale for not choosing them?

To place the entrypoint binary in the container, we need to either copy files
into it using, e.g., an init container and shared volume or have the files
pre-allocated in, e.g., a PersistentVolumeClaim or ConfigMap.

Tekton doesn't support init containers, but we could relatively easily use
[workspaces](https://tekton.dev/docs/pipelines/workspaces/). An init Task would
be a dependency for every other Task in the Pipeline. This introduces some
complexity around scheduling; namely, it forces the entire PipelineRun to run on
the same node using a process they call the affinity assistant. It also means
that we can't use our own affinities in the future.

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

Supporting the features associated with our desired step execution flow would
either fall to a more complex external management process or to the user (i.e.,
forcing them to use a specific framework). Some features would be impossible;
for example, exposing environment variables with secrets would require us to
make unacceptable security compromises.

### What specific risks are associated with this design?

* We haven't fully vetted the PVC approach. For example, with GCE-PD we'll need
  to provision regional disks, at which point each tenant will be restricted to
  running in two zones instead of three. Since the PVCs will be distributed
  across tenants we should still retain very high availability guarantees
  barring other unforeseen issues.
* There is a cultural risk that the entrypointer binary may feel like trusted
  code. It is not. It runs directly in a user-controlled environment without any
  separation from its delegate process. For example, the delegate process might
  directly read or manipulate the memory of the entrypointer.

  In general, the entrypointer's logic for performing authenticated work should
  mirror user code: it should just communicate with the metadata API. It is
  imperative that we not include any sensitive credentials (global or otherwise)
  in the entrypoint binary or environment.
* Conversely, if our own code is insecure, we introduce a potential attack
  vector to users' containers.
* We recognize that this may make compatibility with operating systems and
  architectures other than Linux/amd64 more difficult in the future, but we have
  no near- or medium-term plans for this.
* Although we expect our injection strategy to be broadly compatible with
  existing Linux containers, we anticipate a small subset of containers will not
  successfully run under our injected supervisor process. We will have to
  monitor this and fix it on a per-occurrence basis.

## Success criteria

When this RFC is implemented, all tenant containers (defined by steps and
webhook triggers) will use our entrypoint instead of the container- or
user-specified one.

## Future possibilities

This RFC proposes a broader semantic change to our platform: we now encourage
some close locality with tenant containers. We have historically operated with
an observer-only model. We expect that once we start to leverage the potential
the entrypointer process provides we'll be able to implement far more
interesting features than we've explored in the product-level explanation.
