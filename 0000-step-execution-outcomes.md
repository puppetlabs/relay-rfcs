# Step execution model and outcomes (2020-02-10)

## Stakeholders

* **Recommenders:** Noah Fontes, Rick Lane
* **Agreers:** Brad Heller, Kyle Terry
* **Performers:** Rick Lane
* **Inputers:** Noah Muldavin
* **Deciders:** Brad Heller, Noah Muldavin, Kyle Terry

## Problem

We lack clarity on the possible status a step can be in when a workflow is run.
We've also conflated computational work with other processes in a way that is
seemingly both confusing to our system and to end users. The UI has strange
logic to encapsulate this, and as we look at adding additional step types and
statuses, it isn't getting any simpler.

## Summary

This change formally defines a step and the different execution statuses that a
step may take on as part of a single finite state machine. We additionally
propose to separate approvals into a distinct subsystem that is subject to
different rules that have more clarity for end users.

## Motivation

This change is relatively simple technically (updating some state fields in our
Kubernetes workflow resource) and provides a great deal of clarity for
implementers of workflow visualization UIs and end users.

## Product-level explanation

We address the issue of different outcomes per step type (that is, container
steps having `success` and `failure` but approval steps having `accepted` and
`rejected`) by making the definition of a step more strict.

This RFC defines a **step** as a unit of computational work. Other processes,
such as approvals and triggers, are not steps, and shall not be included with
steps in workflows. We expect there to be multiple step types in the future (for
example, groups of steps), but they will all have a common `success` or
`failure` outcome.

### Asks

As it relates to our current implementation, we introduced the concept of step
types to support approvals, a non-work process. We therefore propose a new
generic structure for approvals, **asks**, that is external to, and referential
to, steps. We believe this will create a clearer user experience. A separate RFC
will address the technical implementation of asks, but we highlight the change
here because we eliminate the `approved` (expressed as `success`) and `rejected`
step statuses as part of this RFC.

In summary, today we have a YAML document with an approval step:

```yaml
steps:
- name: my-gate
  type: approval
- name: deploy
  dependsOn: approval
  image: projectnebula/terraform
  # ...
```

We propose to replace this with a more substantial feature:

```yaml
asks:
- name: my-gate
  image: projectnebula/ask-manual-gate
  spec:
    question: !Fn.concat ['Do you want to deploy ', !Parameter project, '?']

steps:
- name: deploy
  when: !Answer my-gate
  image: projectnebula/terraform
  # ...
```

This allows customers to bring their own asks as long as they conform to our
asks API. (In this case, we use our own manual gate image, which may also be the
default.)

## Engineering-level explanation

Currently, the set of possible statuses for a step are `pending`, `in-progress`,
`success`, `failure`, `skipped`, `timed-out`, `cancelled`, and `rejected`, none
of which are rigorously defined.

We propose to amend the set of execution statuses possible for all steps to be
defined by a single FSM. In addition, we propose removing non-work steps
entirely as well as the `rejected` status (see the discussion in the
product-level explanation).

### Resolution

This document refers to execution statuses as **resolved** or **unresolved**.
This is a concept that can be generalized to any data field that is used in a
condition. Data that is **unresolved** cannot be used to make a decision yet; it
represents a lock around the data itself with a value that indicates why the
lock is being held. Data that is **resolved** is no longer locked, and its value
is directly available for consumption. A condition will wait indefinitely for
all *necessary* unresolved data to be resolved (conditions are allowed to
short-circuit).

### Execution status

![Flow diagram](0000-step-execution-outcomes/flow.png)

The new step statuses are:

* `initializing` (unresolved): The workflow run resource has been created, but
  the execution backend (Tekton, metadata API) is not running yet.
* `pending` (unresolved): The execution backend started and the step is waiting
  for all of its `when`-conditions to be satisfiable (*not* satisfied) before
  running.
* `in-progress` (unresolved): If the `when`-conditions are satisfied, the step
  executes. If the particular step type does not perform any work (as is the
  case with an approval step type, for example), this status is skipped and the
  step transitions directly to completed.
* `success` (resolved): All work associated with this step completed in a way
  the step semantically deems successful.
* `failure` (resolved): All work associated with this step completed in a way
  the step semantically deems unsuccessful.
* `system-error` (resolved): If an error occurs that causes a serious lack of
  accounting for the management of a step, the step transitions to a
  `system-error` status. This generally implies a hardware failure, Kubernetes
  error, metadata API error, or other management error that indicates a serious
  problem with executing the Nebula workflow. It is different from a `failure`
  status, which indicates that the work being supervised failed, not that the
  supervision itself failed. When a step enters a `system-error` status, the
  entire workflow is halted and the workflow run is considered indeterminate.
* `skipped` (resolved): If either the `when`-conditions are not satisfied or the
  workflow is cancelled prior to the step running, the step transitions to the
  `skipped` status.
* `timed-out` (resolved): If the step is `in-progress` and reaches a user- or
  system-configured timeout, it may be terminated and transition to the
  `timed-out` status.
* `cancelled` (resolved): If the user requests workflow cancellation while a
  step is in a `in-progress` status, the underlying process is immediately
  interrupted and transitioned to this status.

These are a superset of the existing step statuses, so this serves to clarify
their current meaning as well as reduce ambiguity by adding additional statuses.

We define a new YAML tag, `!Status`, to refer to the resolved (terminal)
status of a step. For example, we may want to run a particular step to perform cleanup work if another step timed out:

```yaml
when:
- !Fn.equals [!Status some-step, timed-out]
```

### API model

We propose an update to the API to propagate this information to users. We amend
the `status` property of the `AnyWorkflowRunStepState` schema:

```yaml
AnyWorkflowRunStepState:
  type: object
  properties:
    status:
      type: string
      description: Execution status of this workflow step
      enum:
      - initializing
      - pending
      - in-progress
      - success
      - failure
      - system-error
      - skipped
      - timed-out
      - cancelled
      example: pending
    # ...
```

### Additional considerations

Note that we do not reference `dependsOn` when describing the state transition
from `pending` to `in-progress`. `dependsOn` is now defined as equivalent to a
particular `when`-condition, namely:

```yaml
dependsOn: step-a
```

⥴

```yaml
when: !Fn.equals [!Status step-a, success]
```
