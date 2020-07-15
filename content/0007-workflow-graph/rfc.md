# Workflow Graph v2 ([2020-07-15])

### Prioritize

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

### Nice to have

- [PN-181](https://tickets.puppetlabs.com/browse/PN-181): Refine workflow graph connectors
- [PN-960](https://tickets.puppetlabs.com/browse/PN-960): Overhauled V2 of UI Workflow Graph Viz
    - Graph viz should support cards of varying sizes. This will take some investigation (spike ticket). Likely we will end up with a system where the cards can be one of a certain finite set of sizes, or where one dimension (width) is fixed while the other is fluid.

### Recommend to not do

- [PN-325](https://tickets.puppetlabs.com/browse/PN-325): Minimap or small version of workflow graph
