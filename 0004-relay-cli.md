# [Insert your RFC name] ([Insert the current date])

## Stakeholders

* **Recommenders:** Eric0 
* **Agreers:** 
* **Performers:** 
* **Inputers:** 
* **Deciders:** 

## Problem

We expect that most of our users will want to interact with Relay via an "as code" experience, but we currently do not have a clear picture of what that should look like. This doc attempts to provide design specifications and detailed workflows to address that issue.

## Summary

The Puppet Relay CLI should provide the primary way that power users interact with the service. Its functionality should focus on the key workflows that _workflow and step authors_ need to be successful with the product. A close secondary use case is enabling _low-friction integrations_ with other tools in the ecosystem, by enabling "Unix philosophy"-compliant interactions: shell pipelines, parsable output, and standard input/output conventions.

## Motivation

XXX

## Product-level explanation

The command should follow the [puppet-nogui design principles](https://github.com/puppetlabs/puppet-nogui/blob/master/patterns/design_principles.md) and the Heroku ["12 factor CLI" principles](https://medium.com/@jdxcode/12-factor-cli-apps-dd3c227a0e46). Some specific refinements based on our target user and use cases:

* The CLI should provide a single entry point into everything a user needs to do to interact with the product. (Tentatively named `relay`). It should be easy for users to install and keep up to date. 
* The IA should follow a `relay [noun] [verb] [targets...]` form: the "nouns" should be a limited set of top-level items that users need to interact with and the "verbs" are specific to the noun they're acting on.
* It should enable _progressive discovery_: if a required argument is omitted, the contextual help should guide the user to provide the next level of input rather than erroring without a clear next step. This lets the user build up the command line with repeated invocations with greater specificity (whereas requiring a `--help` argument at each level does not)
* The tool should be able to detect when it's run by human vs via script or in a shell pipeline; if there's a human running it, it should default to a richer experience (color output, interactive prompts).
* For clean pipeline/non-interactive usage, it should conform to conventions and best practices for Unix (and Powershell?) tools: use exit codes to indicate success or failure, keep error/warning/diagnostic output on a different fd from real output, stick to low-bit ascii characters and no colours.

## Engineering-level explanation

Here's a "tree" style diagram of the information architecture for the command.

Some assumptions that need validation/discussion:
* the `init` subcommand is akin to `git init`: it's for authors who are creating a new Relay and want a directory structure set up with the correct scaffolding and metadata to author/test/package it in the correct way. If we need to provide a packaging format, more top-level commands top operate on the Relay as a whole will be needed.
* the `step` operations may need to be handled differently because the CRUD operations work on remote container registries, not against the service itself

```
relay
├── version - display version info
├── init - intialize the current directory as a new Relay
├── help - top-level contextual help; every subcommand also supports 'help' as an argument
├── auth - operations on authentication tokens against the service
│  ├── list - show current auth tokens (XXX maybe "show"? "list" is consistent w/other commands)
│  └── login, logout - ibid
├── integration - An authenticated connection to an external service, such as an API token or password
│  ├── add, delete, edit, list - ibid
│  └── test - (XXX maybe? sends an API call to Relay to verify connectivity works)
├── step - An action belonging to a relay that accomplishes a specific task
│  ├── add, delete, edit, list - ibid
│  └── run - perform an ad-hoc run of a given step; could support no-op?
├── trigger
│  ├── add, delete, edit, list - ibid
│  └── send - craft and send a payload to the service as if you were the triggering application
└── workflow
   ├── add, delete, edit, list - ibid
   └── run - perform an ad-hoc run of a given workflow.
```

## Drawbacks

CLI design is tough! Getting it right is important.

## Rationale and alternatives

### Why is this design the best in the space of possible designs?

This follows best practice for modern CLI / text-based workflows in our space.

This specification draws heavily from Heroku's [CLI Style Guide](https://devcenter.heroku.com/articles/cli-style-guide) and Jeff Dickey's writing on the subject.

Many of the design principles draw from JD Welch's work on [Puppet-NoGUI](https://github.com/puppetlabs/puppet-nogui) 

### What other designs have been considered and what is the rationale for not choosing them?

XXX

### What is the impact of not doing this?

We'd restrict the usability of it to just GUI features, which are tougher and slower to implement.

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
