# Workflow conditionals (2020-02-03)

## Stakeholders

* **Recommenders:** Kyle Terry, Rick Lane, Noah Fontes
* **Agreers:** Kyle Terry, Rick Lane, Noah Fontes, Brad Heller
* **Performers:** Kyle Terry, Rick Lane
* **Inputers:** Eric Sorenson
* **Deciders:** Kyle Terry, Rick Lane, Noah Fontes

## Problem

There is a lack of user-defined reactivity to variable data compared with a logical expression during a workflow run.

Some examples include:
* A user cannot define an alternate branch of steps to take if their environment is staging as opposed to production.
* A user cannot define what happens given the output of a previous step without the use of baking all logic into a single step with a script. Say we have a GKE provisioner that creates a k8s cluster and does a lot of heavy handed resource bootstrapping, we would want to skip all those steps if the cluster already exists and is running successfully.

## Summary

This RFC seeks to address that lack of reactivity with the introduction of a mechanism for defining steps to take in a workflow based on conditions met using a new comparison operator/function in the workflow YAML.

## Motivation

Prevents the proliferation of discrete workflows to change how a run reacts given a specific environment or condition. This introduces a more user-friendly feel to the product.

It might reduce support requests because it encourages smaller steps and encourages the user to NOT write very long shell scripts that could lead to complex bugs that are hard to debug during the lifecycle of a workflow run.

## Product-level explanation

This RFC introduces the new keyword `when` to the YAML dictionary that defines a step in a workflow file. This keyword allows a user to do a logical check using new functions on any data available to a step. This can include Parameters, Outputs and Secrets. This explicitly does not include success/failure of individual steps or the workflow (not at this time). Approvals may or may not be included, but is not the primary motivation at this time.

The exact specifics of the `when` syntax is undecided at this time. Compare with [Argo](https://github.com/argoproj/argo/tree/master/examples#conditionals) and potentially [Tekton](https://docs.google.com/document/d/1R6WlDMC3vuY5StiEIFg5MP18n7_kOa1QHxB10CRlbw0). More citations needed.

Possible example functions may include:
* `!Fn.equals`
* `!Fn.notEquals`
* `!Fn.true`
* `!Fn.false`

The `when` keyword is considered an unordered list of implicit `and` conditions, without any support for `or`, `not`, etc. No logic will be performed to validate the efficacy of the conditional logic. The following examples would never progress, produce no errors, and eventually timeout:

```yaml
- when:
  - !Fn.equals [[!Parameter environment], "production"]
  - !Fn.equals [[!Parameter environment], "staging"]
```

```yaml
- when:
  - !Fn.notEquals [[!Parameter environment], "production"]
  - !Fn.equals [[!Parameter environment], "production"]
```

## Engineering-level explanation

```yaml
apiVersion: v1
name: simple-workflow
description: simple workflow example

parameters:
  name: environment
  default: staging

steps:
- name: create-cluster
  image: projectnebula/gke-cluster-creator:latest
  spec:
    ...

- name: bootstrap-cluster-resources
  image: mycompany/my-cluster-bootstrapper:latest
  when:
  - !Fn.equals [[!Parameter environment], "production"]
  - !Fn.true [!Output create-cluster, cluster-exists]
  spec:
    ...

- name: cluster-bootstrap-cleanup
  image: mycompany/my-cluster-bootstrapp-cleaner:latest
  dependsOn: [bootstrap-cluster-resources]
  spec:
    ...
```

`Fn.equals`: is a function available in the workflow yaml that returns true if the left side is equal to the right side. It takes exactly 2 arguments. Both arguments MUST be comparable types. Comparable types are `string`, `integer`, `float`.

`Fn.notEquals`: is a function similar to `Fn.equals` that returns true if the left side is not equal to the right side.

`Fn.true`: is a function available in the workflow yaml that returns true if the given input evaluates to true. It takes a single argument that must resolve to type `boolean`.

`Fn.false`: is a function similar to `Fn.true` that returns true if the given input evaluates to false. It takes a single argument that must resolve to type `boolean`.

These functions will be declared in [nebula-sdk](https://github.com/puppetlabs/nebula-sdk) in the [workflow spec fnlib](https://github.com/puppetlabs/nebula-sdk/tree/master/pkg/workflow/spec/fnlib) package.

`when`: This is a keyword on the step definition. It is a list of expressions that all need to return `true` in order for the step to run. For each workflow step that contains a `when`, a [Tekton Condition](https://github.com/tektoncd/pipeline/blob/master/docs/conditions.md) resource is created with a `check` image that can read the data being compared, and associated to the corresponding Task via the [PipelineTask](https://github.com/tektoncd/pipeline/blob/master/pkg/apis/pipeline/v1alpha2/pipeline_types.go). See [Conditional PipelineRun example](https://github.com/tektoncd/pipeline/blob/master/examples/pipelineruns/conditional-pipelinerun.yaml) for a simple reference.

In accordance with current processing, once the workflow yaml is received via the Nebula WorkflowRun CRD, the Workflow Controller will translate the workflow to the necessary Tekton Tasks, Conditions, Pipelines, etc. and initiate the PipelineRun. Once this has occurred, no further manipulation of the resources will occur, and the PipelineRun will be solely responsible for the workflow execution.

Since Tekton's condition resources are a bit limited at the moment, this will require a custom image to be created for the Condition resources's check. This pattern has already been established and validated as part of the approval logic. The current approval image may be expanded to handle more generic conditions, or a similar image may be created specifically for conditions.

The list of conditionals for evaluation will be passed to the image. The approval image is already capable of contacting the metadata-api to handle the requisite data for comparison, and a similar pattern will be used for conditions. The exact mechanism for passing and/or accessing the data are implementation details that will be determined later.

## Drawbacks

Tekton Conditions are rudimentary at the moment, and have not been fully implemented, but that does not represent either a blocker or a reason not to use them.

## Rationale and alternatives

### Why is this design the best in the space of possible designs?

It leverages both things we have already built and things Tekton already provides. Introducing a single keyword capable of using all data mechanisms (Outputs, Secrets, Parameters) that already exist means we can move fast when creating the feature and keep it familiar with the rest of the workflow.

### What other designs have been considered and what is the rationale for not choosing them?

TODO

We have not fully determined what the exact syntax for this will be. Nor have we determined how this will relate to potentially similar workflow functionality (evaluating success/failure, approvals, etc.).

### What is the impact of not doing this?

Users will not have a way of reacting to variable data in workflows and will in turn create duplicate workflows with minor changes to support branching logic.

### What specific risks are associated with this design?

Increased complexity in the workflow syntax (and overall flow) may lead to difficulty for a user if not designed properly.

## Success criteria

```
     task-first
         /\
        /  \
task-yes    task-no
```

## Unresolved questions

TODO

Primarily the outstanding concerns with the exact workflow syntax.

## Future possibilities

TODO
