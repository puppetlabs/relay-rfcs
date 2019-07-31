# Workflows Controller (2019-07-31)

## Stakeholders

* **Recommenders:** Brian Maher
* **Agreers:** Noah Fontes, Rick Lane, Kyle Terry
* **Performers:** TBD
* **Inputers:** Brad Heller
* **Deciders:** Noah Fontes, Rick Lane, Kyle Terry

## Summary

"Rebrand" the SecretAuth CRD and controller to be a WorkflowRun CRD
and puppet workflows controller. Existing "workflow run initialization
logic" would move into this controller and new resource deallocation
functionality would also be added to this controller. Additionally,
the workfow run DB records will be synchronized with the WorkflowRun
CRD via a watcher in the `nebula-api`.

## Motivation (aka Problems To Solve)

* Processing a request to create a new workflow run requires a great
  deal of initialization which is currently happening "inline" with
  the HTTP request which is problematic since initialization may be
  prematurely aborted if the HTTP socket is closed leaving the
  workflow run in an "unknown" state indefinitely. It is also annoying
  from a customer perspective since the redirect to the workflow run
  detail page is delayed even though the run appears in the list of
  workflow runs.
* At some point we need to perform resource deallocation so that pod
  logs can be perged and etcd records for the various Kubernetes
  resources can be perged. Also, there is a little bit of CPU overhead
  for each resource in Kubernetes due to the "resyncperiod" of
  Kubernetes controllers.  Where should this garbage collection
  process run? From a customer perspective, if a "pod" is deleted the
  logs for a workflow run are lost and we display an ugly error when
  attempting to view the log.
* Workflow run DB records are currently synchronized only when
  visiting a workflow run detail page. This means some information
  displayed in the workflow detail page (which lists the workflow
  runs) may not be accurate.

## Product-level explanation

* Today, workflow run creation is confusing since the "Run Workflow"
  button appears to hang in a spinning state even though the list of
  workflow runs has been updated with a new entry.
* Today, a workflow run may forever be in an unknown state if the user
  navigates off the workflow detail page before the run has been
  completely initialized.
* Today, logs may "disappear" causing a 500 error when trying to view
  past logs. This occurs when a pod has been deleted.
* Today, the workflow detail page may display incorrect status
  information about a workflow run if the workflow run detail page has
  not been recently viewed.

## Engineering-level explanation

Here is the proposed workflow run "lifecycle":

* A request to create a new workflow run is initiated as such:
  * A new WorkflowRun CRD is created in the Kubernetes cluster which
    includes a "signature" of who initiated the workflow run. If there
    is any failure, the error is returned.
  * The workflow run ID (NOT run number) is returned.
* The workflows controller watches the `WorkflowRun` CRD.
  * If the `WorkflowRun` does not reference a `PipelineRun` in the
    annotations, then the controller will query for all pre-existing
    Kubernetes resources that have a label which references this
    `WorkflowRun` instance. It will then initialize any resources
    which "should" exist and delete any resources which should not
    exist. In this way, the `WorkflowRun` record represents "desired
    state" and the job of the controller is to update the various
    Kubernetes resources so the desired state can occur. All resources
    created will be labeled so it references this `WorkflowRun`
    instance. When initialization is complete, the `WorkflowRun` CRD is
    updated so it references the `PipelineRun` which acts as the "source
    of truth" for the current state of the `WorkflowRun`.
* The workflows controller also watches the `PipelineRun` CRD.
  * If it enters a terminal state, the controller will query for all
    Kubernetes resources that have a label which references the same
    `WorkflowRun` instance that this `PipelineRun` references. If the
    `WorkflowRun` CRD is missing any status information, then it will
    be updated appropriately. If the `WorkflowRun` CRD is missing any
    log location information, then the logs will be archived, and the
    log location information will be saved back the `WorkflowRun` CRD.
    Finally, these Kubernetes resources will be deleted and when all of
    this is successful, the `WorkflowRun` will set a status field to
    indicate that it is in a terminal state.
  * If any status information is updated (location of the task log, or
    status running for example), this information is saved back to the
   `WorkflowRun` CRD.
* The `nebula-api` will watch the `WorkflowRun` CRD.
  * If this is a new workflow run instance, a workflow run DB record
    is created in "pending" state and the `WorkflowRun` CRD is updated
    with the workflow run number which was used.
  * If any status information is updated, this information is saved
    back to the workflow run DB record.
  * If the status field was updated to indicate that it is in a
    terminal state, all status information is saved back to the
    workflow run DB record, and the `WorkflowRun` CRD is deleted.

The work necessary to achieve this is as follows along with T shirt
sizing and priority:

* (S,High) A new workflow run details route needs to allow looking up
  the workflow run detail based on the workflow run ID.
* (M,Low) "rebrand" the secretauth controller to be the workflows
  controller. Specifically:
  * Rename the docker image name to be `puppet-workflows-controller`
  * Rename the entrypoint command to be `puppet-workflows-controller`
  * Move `pkg/controllers/secretauth` to `pkg/controllers/workflows`
  * Move `cmd/nebula-secret-auth-controller` to
   `cmd/puppet-workflows-controller`
  * Update the various `nebula-deploy` commands to the new name.
  * Usher out the new deployment (probably need to manually delete the
    secretauth deployment from the cluster).
* (XS,High) Create a new `WorkflowRun` CRD.
* (S,High) Add `WorkflowRun` CRD watching to the workflows controller.
* (S,High) Copy the pipeline run initialization code from `nebula-api` into
  the workflows controller when it observes a state change for the
  `WorkflowRun` CRD. Also add the logic that saves the "it has been
  initialized" state back to the `WorkflowRun` CRD.
* (M,High) Update the pipeline run initialization code so each resource
  initialized is labeled with a reference back to the originating
  `WorkflowRun` CRD, and update the initialization code so it does not
  construct resources which have already been initialized.
* (S,High) Update `nebula-api` so it no longer initializes pipeline
  runs, but instead creates the `WorkflowRun` CRD.
* (S,High) Update workflows controller so `PipelineRun` state is
  synchronized back to the `WorkflowRun`.
* (S,High) Update workflows controller so it archives logs as part of
  the resource cleanup and saves this information back to the
  `WorkflowRun` CRD.
* (S,High) Update `nebula-api` so the workflow run DB state is
  synchronized with the `WorkflowRun` state and remove the code which
  currently synchronizes this state when obtaining workflow run
  details. If a `WorkflowRun` is not associated with a DB record, then
  create a new DB record, allocate a run number, and save the run
  number back to the `WorkflowRun` CRD.
* (S,High) Update `nebula-api` log fetching so it can fetch archived
  logs.
* (S,High) Update workflows controller so it performs more through
  resource cleanup when the `PipelineRun` enters a terminal
  state. This task depends on log archiving and fetching work being
  completed.
* (optional) Optimize workflow initialization. Potential
  optimizations:
  * (XL,Low) Run a single metadata API service used by all workflow
    runs so we don't need to wait for the metdata API service to
    launch as part of resource initialization. This service would now
    need to be secured. One option for securing this is to have the
    metadata API validate a token which restricts the scope to a
    particular vault location, and have this token be signed by a
    public key which is advertised in a well known location. The
    private part of this key would be "owned" by the workflows
    controller.
  * (S,Low) Create all resources except for the `PipelineRun` while waiting
    for the metadata API to initalize.
* (optional,M,Low) Move log archiving functionality into a custom
  entrypoint init container. This would allow us to perform resource
  cleanup quicker and would reduce the overall network and cpu
  overhead to the cluster.

Potentially conflicting features:

* The log archiving work that Brian Maher is working on.
* ???

## Drawbacks

* We have higher priority work.
* Resource synchronizations can be brittle, although Kubernetes has
  already solved these brittleness problems for us since this is the
  architecture of Kubernetes. The "cost" though is some additional CPU
  and memory overhead in our controller and `nebula-api`.

## Rationale and alternatives

### Why is this design the best in the space of possible designs?

There are probably other designs that have not been considered that
are better, however the solution described here mimics the standard
Kubernetes architecture which uses desired state stored in CRDs and
continually attempts to synchronize this desired state back. This
allows us to deal with various error edge-cases more gracefully if a
process crashes or is restarted.

### What other designs have been considered and what is the rationale for not choosing them?

Varations on the above design have been considered. Here are the
varations we have considered:

* Leave the `SecretAuth` controller and CRD as-is and create a new
  controller for this. The downfall to this is that `PipelineRun`
  objects now need to be "watched" from two locations and resource
  cleanup logic is now distributed across controllers. It is also one
  more process that would need to be managed in the cluster.
* Should we simply use the `Pipelines` CRD rather than creating a new
  `WorkflowRun` CRD? Having our own CRD means we can evolve the
  structure without impacting Tekton, and creating new CRDs is pretty
  easy in Kubernetes, so we might as well just do that now.
* When dispatching a new workflow run, perhaps we should make it 100%
  async. Specifically, don't wait for the DB or CRD writes to be
  successful. The advantage of this is we can return from the API call
  even quicker. The downfall to this is that if a write to the DB or
  CRD fails we can't message this to the original API caller.
* Should we just synchronize the workflow run DB state directly in the
  workflow controller? The downfall to this is that the workflow
  controller would now need to manage DB state and therefore any DB
  schema changes would need to stay in sync between the workflow
  controller process and the `nebula-api` process. However, the
  advantage of synchronizing the DB directly in the controller is that
  state transitions can be updated quicker.
* We could create the workflow run DB record before creating the
  `WorkflowRun` CRD, and simply delete any `WorkflowRun` instances
  that have no backing DB record. However, this also means we can't
  use a CRD to dispatch a new `WorkflowRun`. It also means slightly
  more complex workflow initialization and slightly slower response
  times when creating a new workflow run.

### What is the impact of not doing this?

We would continue to see the problems outlined in the Motivation
section.

### What specific risks are associated with this design?

Potential instability of the product if the tasks outline above are
incomplete or not completed in the correct order.

## Success criteria

The problems outlined in the problem statement will be resolved.

## Unresolved questions

* Do we need a mechanism to ensure a `WorkflowRun` CRD instance can
  only be created by an account owner, or is Kubernetes RBAC
  sufficient?
* Should the initial `WorkflowRun` CRD record simply define the
  workflow ID to run? ...and move the logic which fetches the workflow
  yaml into the `WorkflowRun` state syncing code? The advantage here
  is that initiating a `WorkflowRun` becomes even lower overhead. The
  disadvantage is more coordination is required between the workflow
  controller and the `nebula-api` since the `nebula-api` `WorkflowRun`
  synchronization code would need to resolve the workflow yaml, and
  then the workflow controller would need to "wait" until the
  `WorkflowRun` has the complete workflow json populated. Note that we
  can always defer this work.

* [Insert a list of unresolved questions and your strategy for addressing them before the change is complete. Include potential impact to the scope of work.]

## Future possibilities

Having a `WorkflowRun` CRD allows us to define other Kubernetes
resources to coordinate with in the future. It also allows us to
initiate a workflow run using `kubectl`.
