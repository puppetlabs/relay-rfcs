# Workflow Graph v2 ([2020-07-15])

## Stakeholders

* **Recommenders:** Nathan Ward
* **Agreers:** Noah Muldavin
* **Performers:** Nathan Ward
* **Inputers:** Sebastian Prokuski, Geoff Woodburn
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

The workflow graph should have more distinct implementations of the graph with different card components and different access to data. For example, the run diagram will have run status info, and the preview in the app will have the list of configured connections and other config data, whereas the preview in the website will only have the parsed revision.

The implementation should expose that extra data to the inner card components in a clean way. An implementation of the graph may be a map from step type to React components to render the cards.

We currently have shared base components and sub-components like `BaseCard` and `CardTitle`. The shared workflow graph library may include both the logic for determining layout, other top-level information, and mapping step type to card component implementation, but also include a library of card sub-components such that separate implementations remain visually consistent even though they might show different data.

### Tickets

- [PN-960](https://tickets.puppetlabs.com/browse/PN-960): Overhauled V2 of UI Workflow Graph Viz
    - Migrate [WorkflowGraph](https://github.com/puppetlabs/relay-ui/tree/development/packages/relay-app/src/client/components/WorkflowGraph) to [packages/relay-components](https://github.com/puppetlabs/relay-ui/tree/development/packages/relay-components)
    - Workflow graph should be usable in both the app and website, for both 'preview' and 'run' modes. This will require a system where the card content can be customized for a specific 'implementation' of the graph, while still leveraging underlying layout algorithms, and potentially sub-components to maintain visual consistency across 'implementations'
    - Card visualization content should be schematized so that we can easily add new types of visualizations
    - Graph-drawing algorithm should be plug-able through an internally developed api, so that we can potentially write our own at a later date without having to redo the rest of the visualization logic
- [PN-1103](https://tickets.puppetlabs.com/browse/PN-1103): Update trigger workflow card designs
- [PN-1104](https://tickets.puppetlabs.com/browse/PN-1104): Card design updates
    - Update color of connections logos
    - Add integrations logos
- [PN-1101](https://tickets.puppetlabs.com/browse/PN-1101): Fix dotted lines from trigger cards
- [PN-1092](https://tickets.puppetlabs.com/browse/PN-1092): Refactor BaseCard to move ConnectionLogo to DefaultCard
- [PN-468](https://tickets.puppetlabs.com/browse/PN-468): Visualize condition statement during a workflow run (update the connector styling to indicate when the connection has a condition attached)

### Tickets: nice to have

- [PN-181](https://tickets.puppetlabs.com/browse/PN-181): Refine workflow graph connectors
- [PN-960](https://tickets.puppetlabs.com/browse/PN-960): Overhauled V2 of UI Workflow Graph Viz
    - Graph viz should support cards of varying sizes. This will take some investigation (spike ticket). Likely we will end up with a system where the cards can be one of a certain finite set of sizes, or where one dimension (width) is fixed while the other is fluid.
- [PN-185](https://tickets.puppetlabs.com/browse/PN-185): Spike use of dagre for edge drawing

### Tickets: Recommend to not do

- [PN-325](https://tickets.puppetlabs.com/browse/PN-325): Minimap or small version of workflow graph


## Drawbacks

There is an agreement that this is an improvement both to the workflow graph component itself and the ability for multiple apps to consume it. The drawbacks or largely time investment and need to block other graph changes while work is in flight.

## Success criteria

Immediate success criteria is the ability to deploy the website with workflow documentation that includes the workflow graph component rather than screenshots.

## Future possibilities

This change unlocks the ablity to add extra features to workflow graphs on the website like exposing validation upfront or interactivity.
