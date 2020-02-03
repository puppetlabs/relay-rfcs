# Workflow conditionals (2020-02-03)

## Stakeholders

* **Recommenders:** Kyle Terry, Rick Lane, Noah Fontes
* **Agreers:** Kyle Terry, Rick Lane, Noah Fontes, Brad Heller
* **Performers:** Kyle Terry, Rick Lane
* **Inputers:** Eric Sorenson
* **Deciders:** Kyle Terry, Rick Lane, Noah Fontes

## Problem

There is a lack of user-defined reactivity to variable data
compared with a logical expression during a workflow run.

Some examples include:

A user cannot define an alternate branch of steps to take if their environment
is staging as opposed to production.

A user cannot define what happens given the output of a previous step without
the use of baking all logic into a single step with a script. Say we have a GKE
provisioner that creates a k8s cluster and does a lot of heavy handed resource
bootstrapping, we would want to skip all those steps if the cluster already
exists and is running successfully.

## Summary

This RFC seeks to address that lack of reactivity with the introduction of a
mechanism for defining steps to take in a workflow based on conditions met using
a new comparison operator/function in the workflow YAML.

## Motivation

Prevents the proliferation of discrete workflows to change how a run reacts given a
specific environment or condition. This introduces a more user-friendly feel
to the product.

Opens the door for better run failure handling. Allows a user to define a set
of step to take given a failure condition.

It might reduce support requests because it encourages smaller steps and
encourages the user to NOT write very long shell scripts that could lead to
complex bugs that are hard to debug during the lifecycle of a workflow run.

## Product-level explanation

This RFC introduces the new keyword `when` to the YAML dictionary that defines a
step in a workflow file. This keyword allows a user to do a logical check using
an new equality function `!Fn.equals` on any data available to a step. This can include
Parameters, Outputs and Secrets.

See [Engineering-level explanation](#engineering-level-explanation) for workflow
examples.

## Engineering-level explanation

```yaml
apiVersion: v1
name: simple-workflow
description: simple workflow example

steps:
- name: create-cluster
  image: projectnebula/gke-cluster-creator:latest
  spec:
    ...

- name: bootstrap-cluster-resources
  image: mycompany/my-cluster-bootstrapper:latest
  when:
  - !Fn.equals
    - !Output create-cluster cluster-exists
    - "production"
  - !Output create-cluster cluster-exists
  spec:
    ...

- name: cluster-bootstrap-cleanup
  image: mycompany/my-cluster-bootstrapp-cleaner:latest
  dependsOn: [bootstrap-cluster-resources]
  spec:
    ...
```

`when`: This is a keyword on the step definition. It is a list of expressions
that all need to return `true` in order for the step to run. If a workflow
contains a `when` in any step, then a [Tekton
Condition](https://github.com/tektoncd/pipeline/blob/master/docs/conditions.md)
resource is created for each `when` with a `check` image that can read the data
being compared.

`Fn.equals`: is a function available the workflow yaml that returns true if the
left side is equal to the right side. It takes a list of arguments that MUST
have a length of 2. Both arguments MUST be comparable types. Comparable types
are `string`, `integer`, `float`. This function is declared in nebula-sdk in the
workflow spec fnlib package.

## Drawbacks

Since Tekton's condition resources are a bit limited at the moment, this will
require a split in logic handling the comparison. We will need to create a
custom image for the Condition resource's check that knows how to talk to the
metadata-api and pass the list of conditionals for evaluation.

## Rationale and alternatives

### Why is this design the best in the space of possible designs?

It leverages both things we have already built and things Tekton already
provides. Introducing a single keyword capable of using all data mechanisms
(Outputs, Secrets, Parameters) that already exist means we can move fast when
creating the feature and keep it familiar with the rest of the workflow.

### What other designs have been considered and what is the rationale for not choosing them?

TODO

### What is the impact of not doing this?

Users will not have a way of reacting to variable data in workflows and will in
turn create duplicate workflows with minor changes to support branching logic.

### What specific risks are associated with this design?

There's the risk of introducing a lot of complexity into an already-complex and
distributed system that's hard to debug. One issue I expect is deadlocked
workflow runs, so we will need to take great care in how we handle pausing a run
to check a conditional.

## Success criteria

```
     task-first
         /\
        /  \
task-yes    task-no
```

## Unresolved questions

* How do we generate Condition resources at the beginning of a run to support
  this?
* Is the metadata-api responsible for doing the comparison?

## Future possibilities

TODO
