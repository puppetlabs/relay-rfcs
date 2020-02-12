# Step execution model and outcomes (2020-02-10)

## Stakeholders

* **Recommenders:** Noah Fontes, Rick Lane
* **Agreers:** Brad Heller, Kyle Terry
* **Performers:** Rick Lane
* **Inputers:** Noah Muldavin
* **Deciders:** Brad Heller, Noah Muldavin, Kyle Terry

## Problem

We lack clarity on the possible states a step can be in when a workflow is run.
We have also conflated the execution state with the outcomes of a step.
Execution state is well defined across step types, but outcomes are not. The UI
has strange logic to encapsulate this, and as we look at adding additional step
types and states, it isn't getting any simpler.

## Summary

This change divides the current step state into two independent properties of a
step, the execution state and the outcome. It concretely defines the execution
state and sets out rules for establishing the possible outcomes for a step.

## Motivation

This change is relatively simple technically (updating some state fields in our
Kubernetes workflow resource) and provides a great deal of clarity for
implementers of workflow visualization UIs.

## Engineering-level explanation

Currently, the set of possible states for a step are `pending`, `in-progress`,
`success`, `failure`, `skipped`, and `cancelled`. These don't properly express
the flow of a step, nor do they allow for more semantic diversity of outcomes in
additional step types (e.g., approvals).

We propose to amend the set of execution states possible for all steps to be
defined by a single FSM. In addition, we propose that outcomes and execution
flow be modeled as two separate properties of a step.

### Resolution

This document refers to both execution states and outcomes as **resolved** or
**unresolved**. This is a concept that can be generalized to any data field that
is used in a condition. Data that is **unresolved** cannot be used to make a
decision yet; it represents a lock around the data itself with a value that
indicates why the lock is being held. Data that is **resolved** is no longer
locked, and its value is directly available for consumption. A condition will
wait indefinitely for all *necessary* unresolved data to be resolved (conditions
are allowed to short-circuit).

### Execution states

![Flow diagram](0000-step-execution-outcomes/flow.png)

The new step states are:

* `initializing` (unresolved): The workflow run resource has been created, but
  the execution backend (Tekton, metadata API) is not running yet.
* `pending` (unresolved): The execution backend started and the step is waiting
  for all of its `when`-conditions to be satisfiable (*not* satisfied) before
  running.
* `running` (unresolved): If the `when`-conditions are satisfied, the step
  executes. If the particular step type does not perform any work (as is the
  case with an approval step type, for example), this state is skipped and the
  step transitions directly to completed.
* `completed` (resolved): All work associated with this step is finished and its
  outcomes are guaranteed to be available for satisfying `when`-conditions.
* `system_error` (resolved): If an error occurs that causes a serious lack of
  accounting for the management of a step, the step transitions to a
  `system_error` state. This generally implies a hardware failure, Kubernetes
  error, metadata API error, or other management error that indicates a serious
  problem with executing the Nebula workflow. It is different from a `failure`
  outcome, which indicates that the work being supervised failed, not that the
  supervision itself failed. When a step enters a `system_error` state, the
  entire workflow is halted and the workflow run is considered indeterminate.
* `skipped` (resolved): If either the `when`-conditions are not satisfied or the
  workflow is cancelled prior to the step running, the step transitions to the
  `skipped` state.
* `timed_out` (resolved): If the step is `running` and reaches a user- or
  system-configured timeout, it may be terminated and transition to the
  `timed_out` state.
* `cancelled` (resolved): If the user requests workflow cancellation while a
  step is in a `running` state, the underlying process is immediately
  interrupted and transitioned to this state.

We define a new YAML tag, `!ExecutionState`, to refer to the resolved (terminal)
state of a step. For example, we may want to run a particular step to perform cleanup work if another step timed out:

```yaml
when:
- !Fn.equals [!ExecutionState some-step, timed_out]
```

### Outcomes

There are two outcomes that apply to all step types: `pending` (unresolved) and
`indeterminate` (resolved). They are automatically chosen by the Nebula step
supervisor as needed.

Any time after a step enters the `running` state and before it enters the
`completed` state, it may **set** its outcome according to its step type.
Once an outcome is set, it may not be changed.

If no outcome has been set for a given step, the following rules apply to
determine the outcome:

* If the step is in any unresolved state, its outcome is `pending`.
* If the step is in any resolved state, its outcome is `indeterminate`.

Each step type has a particular set of fixed possible outcomes, one or more of which must be **desired**, the default behavior when matching in `dependsOn`.

We define the outcomes for `container` and `approval` step types:

* `container`: `success` (desired), `failure`
* `approval`: `approved` (desired), `rejected`

We define a new YAML tag, `!Outcome`, to refer to the outcome of a step. For
example, we may want to run a step when another step has failed:

```yaml
when:
- !Fn.equals [!Outcome some-step, failure]
```

### API model

We propose an update to the API to propagate this information to users. We amend
the `AnyWorkflowRunStepState` schema, removing `status` and replacing it with
`execution_state` and `outcome`:

```yaml
AnyWorkflowRunStepState:
  type: object
  required:
  - execution_state
  - outcome
  execution_state:
    type: string
    description: Execution state of this workflow step
    enum:
    - initializing
    - pending
    - running
    - completed
    - system_error
    - skipped
    - timed_out
    - cancelled
    example: pending
  outcome:
    type: string
    description: The result of this workflow step
    enum:
    - pending
    - indeterminate
    example: pending
  # ...
```

We amend `ContainerWorkflowRunStepState` and `ApprovalWorkflowRunStepState` to
provide additional enumeration values for their `outcome`s:

```yaml
ContainerWorkflowRunStepState:
  allOf:
  - $ref: '#/components/schemas/AnyWorkflowRunStepState'
  - type: object
    description: State representation of a container step
    properties:
      outcome:
        enum:
        - pending
        - indeterminate
        - success
        - failure
      # ...

ApprovalWorkflowRunStepState:
  allOf:
  - $ref: '#/components/schemas/AnyWorkflowRunStepState'
  - type: object
    description: State representation of an approval step
    properties:
      outcome:
        enum:
        - pending
        - indeterminate
        - approved
        - rejected
      # ...
```

Because this is a breaking change, we must define a new API version. We provide
the following guidance for backward compatibility with previous API versions,
which use a `status` field to roughly represent this information:

* If the execution state is `initializing` or `pending`, the status field value
  should be `pending`.
* If the execution state is `skipped`, the status field value should be
  `skipped`.
* If the execution state is `cancelled`, the status field value should be
  `cancelled`.
* If the execution state is `running`, the status field value should be
  `in-progress`.
* If the execution state is `completed` and the outcome is desired, the status
  field value should be `success`.
* If the execution state is `completed` and the outcome is not desired, or if
  the execution state is `system_error` or `timed_out`, the status field value
  should be `failure`.

### Additional considerations

It is possible that in the future the outcome of a step will be known long
before the step is completed. For example, a step may succeed and then perform
some cleanup work.

Further, note that we do not reference `dependsOn` when describing the state
transition from `pending` to `running`. `dependsOn` is now defined as equivalent
to a particular `when`-condition, namely:

```yaml
dependsOn: step-a
```

⥴

```yaml
when: [!Fn.equals [!Outcome step-a, *desired]]
```

This is a slight change from the existing implementation of `dependsOn`, which
assumes that the step must also be in a `completed` state before dependents
start running.
