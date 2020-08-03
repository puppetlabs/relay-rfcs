# Workflow Graph v2 (2020-07-15)

## Stakeholders

* **Recommenders:** Nathan Ward
* **Agreers:** Noah Muldavin
* **Performers:** Nathan Ward, Hunter Haugen
* **Inputers:** Sebastian Prokuski, Geoff Woodburn, Hunter Haugen
* **Deciders:** Nathan Ward, Noah Muldavin

## Problem

We currently have to take screenshots of workflows for the "Graph" section of workflow documentation on https://relay.sh.

## Summary

This RFC proposes turning the workflow graph into a shared React component in relay-ui and making it consumable by relay-website as well as relay-app.

## Motivation

This change will reduce the overhead of documenting workflows by removing the need to take screenshots when documenting a workflow and updating screenshots when they change.

## Product-level explanation

In addition to removing the need to take screenshots of graphs, this change will improve the quality of the graph section of workflows on the website and open the door for future improvements like upfront validation.

## Engineering-level explanation

In addition to moving [WorkflowGraph](https://github.com/puppetlabs/relay-ui/tree/development/packages/relay-app/src/client/components/WorkflowGraph) into the [relay-components](https://github.com/puppetlabs/relay-ui/tree/development/packages/relay-components) package, it will need to be made more generic and compatible with a read-only mode without the extra details available to the app.

The workflow graph should have more distinct implementations of the graph with different card components and different access to data. For example, the run diagram will have run status info, and the preview in the app will have the list of configured connections and other config data, whereas the preview in the website will only have the parsed revision. (The implementation should expose that extra data to the inner card components in a clean way. An implementation of the graph could be a map from step type to React components to render the cards.)

We currently have shared base components and sub-components like `BaseCard` and `CardTitle`. The shared workflow graph library may include both the logic for determining layout, other top-level information, and mapping step type to card component implementation, but also include a library of card sub-components such that separate implementations remain visually consistent even though they might show different data.

Lastly, in terms of data flow, we will need to expose a "validation" endpoint that can be used by the website build system to turn a YAML workflow file in a revision data structure.

### Goal

Implement workflow graph on docs site

### Tickets

- [PN-960](https://tickets.puppetlabs.com/browse/PN-960): Overhauled v2 of Workflow Graph
    - [PN-1183](https://tickets.puppetlabs.com/browse/PN-1183) Migrate [WorkflowGraph](https://github.com/puppetlabs/relay-ui/tree/development/
    - [PN-1182](https://tickets.puppetlabs.com/browse/PN-1182) Move `generateGraphLayout` to shared workflow-graph package
packages/relay-app/src/client/components/WorkflowGraph) to [packages/relay-components](https://github.com/puppetlabs/relay-ui/tree/development/packages/relay-components)
    - [PN-1185](https://tickets.puppetlabs.com/browse/PN-1185) Create workflow file validation endpoint
        - Unauthenticated endpoint?
        - CLI command?
    - [PN-1186](https://tickets.puppetlabs.com/browse/PN-1186) Add build step to relay-website that uses workflow file validation endpoint to turn workflow file into revision data
    - [PN-1184](https://tickets.puppetlabs.com/browse/PN-1184) Extend workflow graph for use in website, supporting preview and run modes
    - [PN-1103](https://tickets.puppetlabs.com/browse/PN-1103) Update trigger workflow card designs
    - [PN-1104](https://tickets.puppetlabs.com/browse/PN-1104): Card design updates
        - Update color of connections logos
        - Add integrations logos
    - [PN-1101](https://tickets.puppetlabs.com/browse/PN-1101): Fix dotted lines from trigger cards
    - [PN-1092](https://tickets.puppetlabs.com/browse/PN-1092) Refactor workflow cards for extensibility
        - Refactor BaseCard to move ConnectionLogo to DefaultCard
        - Card visualization content should be schematized so that we can easily add new types of visualizations

### Tickets: nice to have

- [PN-181](https://tickets.puppetlabs.com/browse/PN-181) Refine workflow graph connectors
    - [PN-185](https://tickets.puppetlabs.com/browse/PN-185) Spike use of dagre for edge drawing
    - Spike use of Bezier curves for workflow graph connectors
- [PN-1187](https://tickets.puppetlabs.com/browse/PN-1187) Add support for multiple card sizes

### Tickets: Recommend to not do

- [PN-325](https://tickets.puppetlabs.com/browse/PN-325): Minimap or small version of workflow graph

## Drawbacks

There is an agreement that this is an improvement both to the workflow graph component itself and the ability for multiple apps to consume it. The drawbacks or largely time investment and need to block other graph changes while work is in flight.

## Rationale and alternatives

### Why is this design the best in the space of possible designs?

Sharing workflow graph code between the different packages of relay-ui results in code reuse, ability to automate documentation, and share future improvements.

### What other designs have been considered and what is the rationale for not choosing them?

The status quo is manually taking screenshots of workflow graphs for documentation. An alternative design is parsing YAML without the use of a validation endpoint.

### What is the impact of not doing this?

Not doing this work would require screenshots to be taken for each new or updated workflow, resulting in a documentation maintenance burden.

### What specific risks are associated with this design?

The risks include the maintenance of a shared codebase with the possibility of a change for the website breaking something in the app or vice versa.

## Success criteria

Immediate success criteria is the ability to deploy the website with workflow documentation that includes the workflow graph component rather than screenshots.

## Unresolved questions

The exact design of the YAML validation endpoint is yet to be determined. This endpoint will be required for turning the workflow YAML file into the revision data required by the workflow graph.

## Future possibilities

This change unlocks the ablity to add extra features to workflow graphs on the website like exposing validation upfront or interactivity.
