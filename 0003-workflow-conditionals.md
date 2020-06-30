# Workflow Conditions (2020-02-03)

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
* A user cannot define what happens given the output of a previous step without the use of baking all logic into a single step with a script.

## Summary

This RFC seeks to address that lack of reactivity with the introduction of a mechanism for defining steps to take in a workflow based on conditions met using a new comparison operator/function in the workflow YAML.

## Motivation

Prevents the proliferation of discrete workflows to change how a run reacts given a specific environment or condition. This introduces a more user-friendly feel to the product.

It might reduce support requests because it encourages smaller steps and encourages the user to NOT write very long shell scripts that could lead to complex bugs that are hard to debug during the lifecycle of a workflow run.

## Product-level explanation

This RFC introduces the new keyword `when` to the YAML dictionary that defines a step in a workflow file. This keyword allows a user to do a logical check using new functions on any data available to a step. Initially this includes Parameters and Outputs.

Initials functions include:

* `!Fn.equals`
* `!Fn.notEquals`

The `when` keyword is considered an unordered list of implicit `and` conditions, without any support for `or`, `not`, etc.

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

- name: deploy-common-resources
  image: projectnebula/gke-cluster-deployer:latest
  spec:
    ...
  dependsOn:
  - create-cluster

- name: deploy-staging-resources
  image: projectnebula/gke-cluster-deployer:latest
  when:
  - !Fn.equals [[!Parameter environment], "staging"]
  spec:
    ...
  dependsOn:
  - deploy-common-resources

- name: deploy-production-resources
  image: projectnebula/gke-cluster-deployer:latest
  when:
  - !Fn.equals [[!Parameter environment], "production"]
  spec:
    ...
  dependsOn:
  - deploy-common-resources

- name: deploy-monitoring
  image: projectnebula/gke-cluster-deployer:latest
  when:
  - !Fn.notEquals [[!Output create-cluster, environment], "dev"]
  spec:
    ...
  dependsOn:
  - deploy-common-resources

```

`Fn.equals`: is a function available in the workflow yaml that returns true if the left side is equal to the right side. It takes exactly 2 arguments. Both arguments MUST be comparable types. Comparable types are `string`, `integer`, `float`, and `boolean`.

`Fn.notEquals`: is a function similar to `Fn.equals` that returns true if the left side is not equal to the right side.

These functions will be declared in [nebula-sdk](https://github.com/puppetlabs/nebula-sdk) in the [workflow spec fnlib](https://github.com/puppetlabs/nebula-sdk/tree/master/pkg/workflow/spec/fnlib) package.

`when`: This is a keyword on the step definition. It is a list of expressions that all need to return `true` in order for the step to run. For each workflow step that contains a `when`, a [Tekton Condition](https://github.com/tektoncd/pipeline/blob/master/docs/conditions.md) resource is created with a `check` image that can read the data being compared, and associated to the corresponding Task via the [PipelineTask](https://github.com/tektoncd/pipeline/blob/master/pkg/apis/pipeline/v1alpha2/pipeline_types.go). See [Conditional PipelineRun example](https://github.com/tektoncd/pipeline/blob/master/examples/pipelineruns/conditional-pipelinerun.yaml) for a simple reference.

In accordance with current processing, once the workflow yaml is received via the Nebula WorkflowRun CRD, the Workflow Controller will translate the workflow to the necessary Tekton Tasks, Conditions, Pipelines, etc. and initiate the PipelineRun. Once this has occurred, no further manipulation of the resources will occur, and the PipelineRun will be solely responsible for the workflow execution.

Since Tekton's condition resources are a bit limited at the moment, this will require a custom image to be created for the Condition resources's check. This pattern has already been established and validated as part of the approval logic. The current approval image may be expanded to handle more generic conditions, or a similar image may be created specifically for conditions.

The list of conditions for evaluation will be passed to the image. The approval image is already capable of contacting the metadata-api to handle the requisite data for comparison, and a similar pattern will be used for conditions. The exact mechanism for passing and/or accessing the data are implementation details that will be determined later.

## Drawbacks

 No logic will be performed to validate the efficacy of the conditional logic. The following examples would never progress, produce no errors, and eventually timeout:

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

Initially, steps that support conditions will not be able to converge on a common step later in the workflow.

Secrets will not be supported (initially), due their nature, and any such use would likely reference parameters disguised as secrets. Exposing secrets for this purpose would potentially lead to security concerns.

The following are not supported as conditions at this time (but likely will be in the future):

* The outcome of steps or the workflow (success, failure, etc.)
* The outcome of approvals (approved, rejected, etc.)
* The step/workflow status (running, completed, cancelled, etc.)

The use and context of the `dependsOn` keyword will later transition to be a specific subset of `when` processing that implies the default (successful) outcome condition. Other similar features and keywords may be added to avoid increased complexity while retaining full flexibility; potentially `onSuccess` or `onFailure` for example. `dependsOn` will considered to be in aggregate with any other `when` conditions.

## Rationale and alternatives

### Why is this design the best in the space of possible designs?

It leverages both things we have already built and things Tekton already provides. Introducing a single keyword capable of using all data mechanisms that already exist means we can move fast when creating the feature and keep it familiar with the rest of the workflow.

### What other designs have been considered and what is the rationale for not choosing them?

Other similar syntax:

* [Argo](https://github.com/argoproj/argo/tree/master/examples#conditionals)
* [Tekton](https://docs.google.com/document/d/1R6WlDMC3vuY5StiEIFg5MP18n7_kOa1QHxB10CRlbw0)

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

None.

## Future possibilities

* More condition functions may be added in the future.
* The outcome of a step (success/failure) may be added as conditions later.
* The outcome of approvals (approved, rejected) may be added as conditions later.
* The status of a step may be added as conditions later.
* External events will eventually be supported by this same system/design. For example, manual or automatic approvals from Slack, PRs, JIRA, etc. will be possible (preferably as integrations, without explicitly relying on a waiting step).
