# Relay CLI RFC - March 2020

## Stakeholders

* **Recommenders:** Eric0 
* **Agreers:** 
* **Performers:** 
* **Inputers:** 
* **Deciders:** 

## Problem

We expect that most of our users will want to interact with Relay via an "as code" experience, but we currently do not have a clear picture of what that should look like. This doc attempts to provide design specifications and detailed workflows to address that issue.

## Summary

Since Relay is built around a strong "as-code" experience, the Relay CLI will provide the primary way that our users interact with the service. The three main use cases (in priority order) that it should support are:

* _Interacting with the service_ from a text-based terminal on Mac, Windows, and Linux systems. 
* Enabling _low-friction integrations_ with other tools, by enabling "Unix philosophy"-compliant interactions: shell pipelines, parsable output, and standard input/output conventions.
* Supporting the _development loop for authors_ (write, test, debug, commit) to create content.

## Motivation

While the web app is integral to the user experience of Relay, there's an equally critical set of experiences that are not centered on the GUI. Workflow development is inherently text-based, the 'develop-test-commit' loop is fastest when the user doesn't have to leave their editor, and providing a quality text-based interface enables fast and easy integrations with other tools. So it's important to be deliberate about the design choices that go into the CLI and authoring experience.

## Product-level explanation

Ideally every user workflow (sorry for overloading the term) that we build into the GUI should have a CLI equivalent; the terminology, information architecture, and learning journey that a user sees should translate across contexts. 

The command should follow the [puppet-nogui design principles](https://github.com/puppetlabs/puppet-nogui/blob/master/patterns/design_principles.md) and the Heroku ["12 factor CLI" principles](https://medium.com/@jdxcode/12-factor-cli-apps-dd3c227a0e46). Some specific refinements based on our target user and use cases:

* It should be easy for users to install and keep up to date. Homebrew, Linux binaries, and Chocolatey packages should be the target client platforms.
* The IA should follow a `relay [noun] [verb] [targets...]` form: the "nouns" should be a limited set of top-level items that users need to interact with and the "verbs" are specific to the noun they're acting on.
* It should enable progressive discovery: if a required argument is omitted, the contextual help should guide the user to provide the next level of input rather than erroring without a clear next step. This lets the user build up the command line with repeated invocations with greater specificity (whereas requiring a `--help` argument at each level does not)
* The tool should be able to detect when it's run by human vs via script or in a shell pipeline; if there's a human running it, it should default to a richer experience (color output, interactive prompts).
* For clean pipeline/non-interactive usage, it should conform to conventions and best practices for Unix (and Powershell?) tools: use exit codes to indicate success or failure, keep error/warning/diagnostic output on a different fd from real output, stick to low-bit ascii characters and no colours.

These screenshots from the pulumi cli illustrates many of these principles:

![Annotated screenshot of Pulumi cli entry point](media/pulumi1.png)


![Annotated screenshot of Pulumi cli with a stack subcommand](media/pulumi2.png)

## Engineering-level explanation

Here's a tree diagram of the information architecture for the command. Aside from the `devel` subcommand, which is primarily concerned with a local development loop, these perform operations directly on the service or between the local filesystem and the service.

It's useful to have conventions that make the tool behave consistently and "intuitively" (nothing about computers is really _intuitive_ but being able to reuse things you've already learned leads to a sense of mastery). Some of the specific conventions in this tool are:

* the use of `list` always means to display a column-formatted enumeration of every instance the tool can find
* the use of `show` always means to display a detailed output of one named instance
* for commands which need positional arguments, the user can specify multiple space-separated arguments
* for commands which need positional arguments and none are supplied, it should present a list of possibilities to fill those arguments

```
relay
├── auth - authenticate to the service
│  ├── list,show
│  ├── select
│  └── login,logout
├── connection - manage stored authenticated connections to external services
│  ├── add,delete,edit,list,show
│  └── test
├── devel - operations useful for content authors/editors are aggregated here
│  ├── init
│  ├── new
│  └── test
├── runs - show the history of workflow and step execution
│  ├── list
│  └── show
├── step - operate on individual steps
│  └── run
├── event - view history of events received by the service and create/send synthetic events 
│  ├── list
│  ├── send
│  └── show
├── workflow - operate on the workflows available on the service
│  ├── add,delete,edit,list,show
│  ├── replace
│  └── run
├── help - top-level help text (also displayed when `relay` is run with no arguments)
└── version - display version info
```

The next section goes into more detail on each of these subcommands.

### Global arguments

The following commands should work for all levels of noun/verb specificity and provide consistent results regardless of where they're used:

Verbosity:
* `--verbose` - show additional verbose output, to aid in understanding what goes on (i.e. showing HTTP request/response headers when communicating with the service)
* `--debug` - present maximally verbose output, to aid in debugging if something is broken (i.e. showing HTTP payload/bodies)

Help:
* `--help` - show contextual help based on the user's input thus far (run at the top level, should show nouns and global flags; run after a noun+verb, should provide positional arguments, noun-specific and global flags). `relay` without arguments should display the top-level help text and `relay help noun` should be equivalent to `relay noun --help`. Help text should be brief (a screenful of around 40 lines max) and provide context-aware links to long form documentation on the web where possible.

Interactivity:
* `--color`/`--no-color` - forcibly enable/disable color and emoji in output, overriding the tool's detection of interactivity
* `--out (text|json)` -- forcibly print output in a given format (exact formats are TBD, but at least json should be supported as a structured output so that the output can be piped through other commands)
* `--yes` - do not prompt for confirmation for operations which normally require it

### auth

This noun operates on the user's authentication against the Relay service. 

#### Usage

* `list` - enumerates existing authentication tokens, or returns a hint to `relay auth login` if none exist
* `login` - accepts a positional argument of an identity to login with, or prompts if none is supplied
* `logout` - accepts a positional argument of an identity to log out, or presents a pick-list of existing tokens if none is supplied
* `show` - display details about the currently active authentication credential
* `select` - pick from a list of available authenticated connections to use by default

#### Contextual Arguments

login:
* `[positional]` - identity to use for authentication

logout, show:
* `[positional]` - identity to use for authentication, or default to currently active one if no argument is provided
* `--all` - clear all authentication tokens

### connection 

This noun manages authenticated connections to an external service, such as an API token or password. There's no affordance for creating an account through this; we expect that will be done exclusively through the web interface. The connection information should be stored in a `$HOME/.relay` directory on Mac/Linux and XXX (equivalent location) on Windows.

#### Usage

* `add` - create a new connection; this should prompt the user for the same arguments as the GUI workflow: external service, credentials, etc and communicate with the Relay service to establish the connection. 
* `delete [arg]` - accepts a positional argument of a friendly name to delete 
* `edit [positional]` - prompts to change parameters of an existing connection
* `list` - enumerates existing connections
* `show [positional]` - provide detail about a named connection 
* `verify` - attempts to communicate with the external service to verify authentication is working

#### Contextual arguments

add:
* `[positional]` - user-friendly display name for the stored connection
* `--service` - URL to connect to
* `--user` - id to authenticate to the remote service with
* `--pass` - password/credentials to provide to the remote service (XXX maybe a bad idea... but not sure how else you'd do this non-interactively)

show:
* `[positional]` - display details about this connection
* `--watch` - follow events as they come in from this connection

### devel

The `relay devel` subcommand aggregates all of the operations that are primarily used when users are authoring or testing content. will initialize the current working directory with the scaffolding necessary to build a well-formed integration. It's intended for authors who are starting to build a new integration against an external tool/service they own or care about.

#### Usage

* `init` - initialize a directory for Relay development
* `new` - create a new workflow or step 
* `test` - test specified targets

`relay devel init` is analogous with `git init` or `pdk init` for Puppet modules. The scaffolding should include starting points for each type of content the service supports, plus metadata and documentation "prompts" to make the correct thing the easy thing for authors. A directory structure could look like:

```
foo-integration
├── actions
│  ├── queries
│  │  └── queryfoo
│  │     └── Dockerfile
│  ├── steps
│  │  ├── anotherstep
│  │  │  └── Dockerfile
│  │  └── mystep
│  │     └── Dockerfile
│  └── triggers
│     └── triggerblah
│        └── Dockerfile
├── README.md
└── metadata.json
```

<!-- Future considerations from PR comments via @impl:

it is very possible that my "local copy" workflow has files associated with it, and I'm not clear how workflow add / workflow edit would work with that. For example, we already support this:

steps:
- name: foo
 image: alpine
 inputFile: path/to/my-file.sh
I'd also like to get to this point in the near-ish future:

steps:
- name: foo
 build:
   context: steps/foo
In which case we'd actually build the container using the Dockerfile in steps/foo on our service.

The API is going to accept these additional files as an archive (zip, tgz, whatever). I think it's theoretically possible for us to locally traverse the workflow file and find any file references and magically construct the zip file. What do you think the most intuitive UX would be there?
-->

#### Contextual Arguments

`init`:
* Without any arguments, prompts to initialize the current directory
* `[positional]` - directory other than the current one to initialize

`new`: 
* Without any arguments, could interview the user to select the object to create and suggest defaults.
* `action [name] --type [step|query|trigger]` - make a new step of the specified type with a starter Dockerfile

`test`:
* `[positional]` - types of tests to run (TBD); could just do syntax validation against workflow as a start

### runs

The `relay runs` subcommand displays the history of step and workflow execution. 

#### Usage

* `list` - display a tabular view of run history, including information to aid in debugging and drilling down further like run ID, workflow name, status (active, completed, pending, etc) and result (success or failure if completed).
* `show` - provide detailed information about a specified run, similar to the "detail page" in the web UI.

#### Contextual arguments

list:
* `--out [text|json|plain|...]` - this was also mentioned in the global argument list, but there may be more specific output formats that are helpful for these lists as they are likely to be grepped or piped into other scripts 
* `--no-header` - omit header information to make for easier searching
* `--filter [...]` - could be handy to have a simple filter syntax for pre-selecting only some rows of output; for example `--filter status=active`; but maybe it's easier to leave this to grep (TBD)

show:
* `[positional] | --id [number]` - identify a single run whose details to show

### step

This subcommand operates on step actions. Actions are the main atom of functionality in Relay. Actions are provided by special-purpose containers that the service launches in response to something happening. They come in three flavors:

* Step actions - Relay runs step actions, passing in parameters and secrets, as part of an automation workflow.
* Query actions - Sometimes you'll need to break out of automated workflow to prompt for external input, like a one-time password or a human approval. Query actions enable Relay to pause and request information from the outside world before proceeding.
* Trigger actions - External systems send events to Relay, which handles them by executing a Trigger action. The Trigger action is run to determine how to respond to the event.

There's not a meaningful way to interact with query and trigger actions from the CLI because they are invoked as part of other events happening on the service. Step actions, however can be run individually as a stepping stone (sorry) towards building up more complex workflows. 

 
#### Usage

* `run` - this operates against the service, allowing for ad-hoc execution of an action. Running an action with no/minimal arguments could be a low-friction onboarding and diagnostic tool - it could run a general-purpose container on the service and dump back the input params/context, or a "hello world" type script to show users some success before they invest a lot of effort into learning.

#### Contextual Arguments

run:
* `[positional]` - a container registry path to the action which ought to run
* `--params ['{ ... json ... } | @filename | -` - json-formatted parameter names and values to supply, avoiding prompts. `@file` should read from a file, using `-` should read from stdin

### workflow

A workflow is a sequenced collection of actions, parameters, and metadata that automates some work. The `relay workflow` subcommand is responsible for managing the workflows available to the current authenticated user on the service.

#### Usage

* `add` - creates a workflow on the service based on a local file
* `delete` - removes a workflow from the service
* `download` - downloads a named workflow and opens it for editing
* `list` - enumerate available workflows
* `replace` - replace a named workflow on the service with a version stored locally on the user's filesystem
* `run` - execute a named workflow; shouldn't block on execution but rather return a run ID and URL to link to GUI

#### Contextual arguments

`add`:
* No arguments should prompt the user through the creation of a new workflow
* `[workflowname]` - create a workflow with the given name on the service
* `--file [source file]` - use the named file as input; should also support `-` to read from stdin

`delete`:
* `[workflowname]` - prompt to remove the named workflow from the service
* `--yes` - (global) avoid prompting to warn about deletion, just do it

`download`:
* `[workflowname]` - download the current version of the workflow for editing
* `--file [output filename]` - where to save the file on the filesystem; should also support `-` for stdout

`list`:
* no args necessary

`replace`:
* `[workflowname]` - name of the workflow to replace
* `--file [source file]` - use the named file as input; should also support `-` to read from stdin
* `--yes` - (global) avoid prompting to warn about overwriting, just do it

`run`:
* no arguments: could display a pick-list of workflows to run from those available, walk user through param bindings and execution
* `[workflowname]` - name of the workflow to run
* `--params ['{ ... json ... } | @filename | -` - json-formatted parameter names and values to supply, avoiding prompts. `@file` should read from a file, using `-` should read from stdin

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

Success for this is measured by user delight. Users should *want* to use the CLI because it's powerful, intuitive, and helps them accelerate their work. They'll let us know if this is the case; power users in particular are pretty vocal about their likes and dislikes from the CLI and hopefully we can build UX that they'll love.

Another form of success could come from using the CLI as a rapid-prototyping tool to quickly add functionality that would take a long time in the GUI. This "CLI-First" design ethos could lead to a lot of rapid feedback and experimentation.

## Unresolved questions

* Should we collect metrics from the CLI itself or are service-level metrics sufficient? We could learn from Wash and Bolt here.
* The whole `relay devel` subcommand experience could change greatly depending on real world usage by developers. The directory structure, metadata, and exact contents of the scaffolding are under specified. Testing in particular is a big grey area: beyond simple yaml syntax validation, how much testing can we do, and what form should it take? 

## Future possibilities

* lots of opportunity for future work with respect to interactivity models (guided prompts, curses-style "TUI" text user interfaces)
* self-updating or at least awareness of available updates
* metrics and usage reporting from the CLI itself (must be opt-out-able)
* `relay bug` to report bugs directly from the CLI