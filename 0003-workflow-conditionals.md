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

- A user cannot define an alternate branch of steps to take if their environment is staging
  as opposed to production.
- A user cannot define what happens given the output of a previous step without
  the use of baking all logic into a single step with a script. Say we have a GKE
  provisioner that creates a k8s cluster and does a lot of heavy handed resource
  bootstrapping, we would want to skip all those steps if the cluster already
  exists and is running successfully.

## Summary

This RFC seeds to address that lack of reactivity with the introduction of a
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

[Explain this change in a way that our product team can understand it and pitch
it to customers for feedback. Make heavy use of examples and ensure you
introduce any new concepts at a high level.]

## Engineering-level explanation

[Explain, in detail, the technical manifestation of this change. Detail the
scope of work including time or size estimates. Include any potential conflicts
with features that are currently underway.]

## Drawbacks

[Why should we _not_ include this change in the product?]

## Rationale and alternatives

### Why is this design the best in the space of possible designs?

[...]

### What other designs have been considered and what is the rationale for not choosing them?

[...]

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
