# Workflow Controller (2019-07-31)

## Stakeholders

* **Recommenders:** Noah Fontes
* **Agreers:** Rick Lane, Kyle Terry
* **Performers:** Rick Lane, Kyle Terry
* **Inputers:** Brad Heller
* **Deciders:** Rick Lane, Kyle Terry

## Problem

Right now, executing a workflow run requires a delicate dance of creating and
polling Kubernetes resources, with some work done by the API and some done by
the task subsystem in the SecretAuth controller. It complicates adding new
functionality to workflows, causes performance issues, and makes reporting on
errors very difficult.

## Summary

This RFC proposes to rename the SecretAuth <abbr title="Custom Resource
Definition">CRD</abbr> to WorkflowRun. The Nebula API would submit a set of
steps using this CRD and poll it for state changes. We move logic of converting
the CRD to a Tekton PipelineRun to a workflow controller, which is a superset of
the existing SecretAuth controller.

## Motivation

* Processing a request to create a new workflow run requires a great deal of
  initialization which is currently happening "inline" with the HTTP request
  which is problematic since initialization may be prematurely aborted if the
  HTTP socket is closed leaving the workflow run in an "unknown" state
  indefinitely. It is also annoying from a customer perspective since the
  redirect to the workflow run detail page is delayed even though the run
  appears in the list of workflow runs.
* At some point we need to perform resource deallocation so that pod logs can be
  perged and etcd records for the various Kubernetes resources can be purged.
  Also, there is a little bit of CPU overhead for each resource in Kubernetes
  due to the "resyncperiod" of Kubernetes controllers.  Where should this
  garbage collection process run?

## Product-level explanation

Today, workflow run creation is confusing since the "Run Workflow" button
appears to hang in a spinning state even though the list of workflow runs has
been updated with a new entry. It takes a long time to finally respond
successfully, and even then we don't propagate all state correctly, including
informing the user of relevant errors.

This change would make it easier to detect and propagate errors because it only
requires watching one item (the WorkflowRun CRD) for state changes. Also, the
API can make the assumption that once the CRD is created, the system will take
care of the rest of the execution process, meaning that we can usually return
from the workflow run creation endpoint in milliseconds.

## Engineering-level explanation

Here is the proposed workflow run "lifecycle":

1. A request to create a new workflow run is initiated as such:
   1. The API creates and persists a workflow run database record.
   1. A new WorkflowRun CRD is created in the Kubernetes cluster. This CRD
      references the current state of the workflow (e.g., using a secure hash)
      and includes the relevant workflow run ID as a label for further querying.
      If there is any failure, the error is returned. If the error is transient
      (e.g., network error), the API will attempt to recreate the WorkflowRun
      resource until it succeeds or receives a permanent error.
1. The workflow controller watches `WorkflowRun` CRDs.
   * If the `WorkflowRun` does not reference a `PipelineRun` in the annotations,
     the controller will check whether the required Tekton backing resources
     (`Pipeline` and `Task` definitions) exist based on the given workflow
     state. It reconciles this state and creates or updates the Tekton resources
     as needed.

     Finally, the workflow controller creates a `PipelineRun` CRD to execute the
     workflow run.
   * If a `WorkflowRun` CRD is deleted:
     * If the underlying `PipelineRun` is still running, the workflow controller
       attempts to cancel it.
     * Once all dependent resources have been cancelled, they are permanently
       deleted.
1. The workflow controller watches the `PipelineRun` CRD for state changes.
   * If it enters a terminal state, the controller will query for all Kubernetes
     resources that have a label which references the same `WorkflowRun`
     instance that this `PipelineRun` references. If the `WorkflowRun` CRD is
     missing any status information, then it will be updated appropriately. If
     the `WorkflowRun` CRD is missing any log location information, then the
     logs will be archived, and the log location information will be saved back
     the `WorkflowRun` CRD. Finally, these Kubernetes resources will be deleted
     and when all of this is successful, the `WorkflowRun` will set a status
     field to indicate that it is in a terminal state.
   * If any status information is updated (location of the task log, or status
     running for example), this information is saved back to the `WorkflowRun`
     CRD.
1. The API watches the `WorkflowRun` CRD.
   * If any status information is updated, this information is saved back to the
     workflow run DB record.
   * If the status field was updated to indicate that it is in a terminal state,
     all status information is saved back to the workflow run DB record, and the
     `WorkflowRun` CRD is deleted.

The work necessary to achieve this is as follows along with T shirt
sizing and priority:

| Component | Size | Priority | Description |
|-----------|------|----------|-------------|
| Workflow Controller | M | Low | Rebrand the SecretAuth controller to be the workflow controller. |
| Workflow Controller | XS | High | Spec out and create a new `WorkflowRun` CRD. |
| Workflow Controller | S | High | Add `WorkflowRun` CRD watching to the workflows controller. |
| Workflow Controller | M | High | Copy the `PipelineRun` initialization code from `nebula-api` into the workflow controller when it observes a state change for the `WorkflowRun` CRD. Also add the logic that saves the initialization state back to the `WorkflowRun` CRD. |
| Workflow Controller | M | High | Update the `PipelineRun` initialization code so each resource initialized is labeled with a reference back to the originating `WorkflowRun` CRD, and update the initialization code so it does not construct resources which have already been initialized. |
| Workflow Controller | M | High | Change the existing `PipelineRun`-related initialization code to work with the new `WorkflowRun` resources. |
| API | S | High | Replace `PipelineRun` creation with `WorkflowRun`. |
| Workflow Controller | L | Medium | Add `Pipeline` and `Task` reconciliation based on information given in the `WorkflowRun` CRD. |
| API | S | Medium | Remove `Pipeline` and `Task` creation. |
| Workflow Controller | S | High | Synchronize terminal state of `PipelineRun`s to `WorkflowRun`s. |

## Drawbacks

Resource synchronizations can be brittle, although Kubernetes has alreadysolved
these brittleness problems for us since this is the architecture of Kubernetes.
The "cost" though is some additional CPU and memory overhead in our controller
and `nebula-api`.

## Rationale and alternatives

### Why is this design the best in the space of possible designs?

There are probably other designs that have not been considered that are better,
however the solution described here mimics the standard Kubernetes architecture
which uses desired state stored in CRDs and continually attempts to synchronize
this desired state back. This allows us to deal with various error edge-cases
more gracefully if a process crashes or is restarted.

### What other designs have been considered and what is the rationale for not choosing them?

Varations on the above design have been considered. Here are the varations we
have considered:

* Leave the `SecretAuth` controller and CRD as-is and create a new controller
  for this. The downfall to this is that `PipelineRun` objects now need to be
  watched from two locations and resource cleanup logic is now distributed
  across controllers. It is also one more process that would need to be managed
  in the cluster.
* Should we simply use the `PipelineRun` CRD rather than creating a new
  `WorkflowRun` CRD? Having our own CRD means we can evolve the structure
  without impacting Tekton (an issue which we've already encountered), and
  creating new CRDs is pretty easy in Kubernetes, so we might as well just do
  that now.
* Should we just synchronize the workflow run DB state directly in the workflow
  controller? The downfall to this is that the workflow controller would now
  need to manage DB state and therefore any DB schema changes would need to stay
  in sync between the workflow controller process and the `nebula-api` process.
  However, the advantage of synchronizing the DB directly in the controller is
  that state transitions can be updated quicker.

### What is the impact of not doing this?

Without any real customers, we're stuck in this situation right now:

```
% kubectl get namespace | grep '^workflow-run-' | wc -l
317
```

That's 300+ workflow run-related namespaces in production that we can't
correctly synchronize and clean up. This problem will get worse until our
cluster stops working.

### What specific risks are associated with this design?

Potential instability of the product if the tasks outline above are incomplete
or not completed in the correct order.

## Success criteria

* RTT for dispatching a workflow run from the API is less than one second from
  an end-user's perspective.
* Any errors in the Tekton `PipelineRun` process execution are correctly
  propagated to the user.
* We are able to begin cleaning up resources correctly in Kubernetes, ensuring
  our cluster remains stable.

## Unresolved questions

* We haven't defined the shape of the CRD yet. Specifically, what information
  should be passed in? We need to know the steps to convert them to Tekton
  tasks. Maybe that should be a separate CRD altogether?
* We have some unanswered questions around streaming logs (before the logs are
  archived). Currently we pull that information directly from the relevant
  `PipelineRun` in the API. It's likely that we'll want some sort of abstraction
  over this, but we don't have clarity on what that looks like right now.

## Future possibilities

Having a `WorkflowRun` CRD allows us to define other Kubernetes resources to
coordinate with in the future. It also allows us to initiate a workflow run
using `kubectl`.
