- Feature Name: informational_commands
- Start Date: 2019-05-15
- RFC PR: (leave this empty)
- Volta Issue: (leave this empty)

# Summary
[summary]: #summary

Add an informational command, `volta list`, replacing the `volta current` command and expanding on its capabilities. The `list` command will allow users to see their default and project-specific toolchains, and will support both human-friendly output and a tool-friendly output.

# Motivation
[motivation]: #motivation

Users of Volta currently have no way to get the answers to the following questions:

- What versions of a runtime, packager, or tool do I have available already on my machine?
- What versions of a runtime, packager, or tool are currently active? And why?
- What versions of a runtime, packager, or tool are the *defaults* for the system?
- What binaries are supplied by a given package which has been installed?
- What Node version and packager are used by a given tool binary?

The only tooling available to answer *any* of these question presently is the `volta current` command, which prints exactly and only the version of Node itself which will be executed in the user’s current directory. Users wanting to answer these questions today must resort to poking through the `~/.volta` directory, and its internals are *not* public API, so any tooling a user might put in place around that would be subject to breakage between versions.

Adding a `volta list` command designed to address all of these needs allows users to get and use the information however they need. Adding it with both human- and tool-friendly output formats makes for a better user-experience: the default output will make it easy for people to *read*, and the machine-readable formats will make it easy for people to build *tools* around.

# Pedagogy
[pedagogy]: #pedagogy

We introduce a `list` command, which lets users get a description of their Volta environment: the currently-active items in their toolchain and the reason those items are currently active, and the total set of available tools on their system.

- To get the currently-active runtime, packager, and available binaries, the user can run `volta list`.

- To get *all* runtimes, packagers, and binaries currently fetched to the user's system, the user can run `volta list --all`.

- The user can also narrow the query:
    - to runtimes: `volta list node`
    - to packagers: `volta list npm` or `volta list yarn`
    - to a specific package: `volta list package <package name>`
    - to a specific tool: `volta list tool <tool name>`

These commands represent the Volta ideas of the user's *toolchain*, including *default* and *current* items within that chain.

These ideas already exist implicitly in Volta's vocabulary. Introducing `list` provides a specific place for users to see what the terms mean and how they are employed. Indeed, one of the primary benefits of the `list` command is that it provides a point where the Volta model—as distinct from e.g. the nvm or nodenv models—can be *made explicit* and thereby taught to users.

## Why `list`?

Potential options include `list`, `ls`, `tools`, `info`, `current`, or simply bare names like `node`, `yarn`, or `<package>` or `<tool>`. However, `list` is the most flexible in terms of allowing variants to let the user describe *parts* of their toolchain as desired. It is also is a very common name for this operation in other tools (see the survey below). Additionally, choosing `list` as the primary name of the command does not preclude adding aliases for common operations later.

### Survey details

A brief survey of the broader developer ecosystem indicates that `list` is by far the most common (sub)command used in CLI tools for listing installed versions of tools. The only major exception is `nodenv`, which (reasonably) seems to treat the "list" action as implicit in the user's intention, given that nodenv serves *only* to manage specific versions of Node. `nvm` uses `ls` and `ls-remote`, which are standard Unix shortenings of "list."

Survey details:

- `nvm` (and the other `*vm` tools):
    - `ls` for installed versions
    - `ls-remote` for available versions

- `nodenv` (and the other `*env` tools) 
    - `versions`: list all installed versions
    - `version`: displays the currently-active version *and* how it was set
    - `local`: list/set the version specified for a given directory tree, if any
    - `global`: list/set the version specified for a global default, if any
    - `shell`: list/set the version specified for a given shell, if any

- [`dotnet`](https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet?tabs=netcore21)
    - [`tool list`](https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-tool-list), which requires either
        - `-g, --global` -> lists “user-wide Global tools”; mutually exclusive with `--tool-path` variant
        - `--tool-path <path>` -> a custom path to look for global tools; mutually exclusive with `--global` variant

- [Chocolatey](https://chocolatey.org):
    - `list`: searches for both local and remote packages, alias for `search`
        - `-l, --lo, --localonly, --local-only`: only items on the local machine
        - `-a, --all, --allversions, --all-versions`: results from all versions
        - `--version=VALUE`: only this specific version match
        - ` -e, --exact`: only exact matches for the name
    - `info` for displaying details about a specific installed package

- [Homebrew](https://brew.sh):
    - `list`, `ls`
    - takes a variety of arguments to limit the output:
        - `--full-name`: give fully-qualified names
        - `--versions`: show version number, takes an optional list of packages
        - `--multiple`: only packages that have multiple versions installed
        - `--pinned`: only for pinned formulae (things brew won’t upgrade without forcing)
    - `info` for displaying 

- [Scoop](https://scoop.sh):
    - `list`

- apt:
    - `list`, lists `<source>/<package>,<package> <version> <arch>`
    - explicitly does not have a stable CLI

- yum:
    - `list`: List package names from repositories
        - `list available`: List all available packages
        - `list installed`: List all installed packages
        - `list all`: List installed and available packages
        - `list kernel`: List installed and available kernel packages
    - `info`: Display information about a package

# Details
[details]: #details

The `volta list` command supports four *variants*:

- **no subcommand:** `volta list` – comparable to the existing `volta current`, but expanded to show not only the user’s current runtime but also their current packager and any available tool binaries, as well as explanation of why the current values are what they are

- **`--all`:** `volta list --all` – shows every fetched runtime, packager, and tool version, along with the binaries available for each tool, and an indication of whether each is a default or is set for a project in the user’s current working directory

- **`package`:** `volta list package <package>` – shows a subset of the output from `--all`, scoped to the information for a specific package which has been fetched to the user’s inventory one or more times

- **`tool`:** `volta list tool <tool>` – shows a subset of the output from `--all`, scoped to the information for a specific tool which has been fetched to the user’s inventory one or more times; like `package` but if a package has more than one tool associated with it, only the specified tool will be shown

It also supports (initially) two *output modes*: *human-friendly* (`human`) and *machine-friendly* (`plain`).

## Information supplied by the command

The `volta list` command always prints the following information for a set of runtimes, packagers, and tools:

- name
- version
- whether it is the user's default or a project-pinned version
- for tools, the associated runtime and packager versions

## Output modes

The tool will support multiple modes (two initially), which include exactly the same information but presented in different human- or machine-friendly formats. All modes include the same information for runtimes, packagers, and tools: name, version, whether it is a default or project-specified version, and (for tools) the Node version and packager (i.e. platform).

- "human" mode, the default if the context is a user-facing terminal; also invokable with `--print=human` in any context. An indented listing of the user's current runtime, packager (if specified), and any installed binaries. See the detailed sections below for examples of the format.

- "plain" mode, the default if the context is not a user-facing terminal (e.g. when piped into another command); also invokable with `--print=plain` in any context. A simple plain text format which prints a line per runtime, packager, or tool, with space-separated output on each line.

    - runtimes:

        ```
        runtime node@<version>
        ```

    - packagers:

        ```
        packager (yarn|npm)@<version>
        ```

    - tools: 

        ```
        tool <tool name> / <package name>@<package version> [node@<version>] [<yarn|npm>@<version>] [(default|current @ <project path>)]
        ```

This RFC does not propose, but allows for the possibility of, a JSON mode (`--print=json`) or similar at a later time if that proves desirable.

## Detailed Command Output

Here we supply a worked example of each *variant* in both *output styles*.

### Assumed Configuration

Throughout, we will assume the user has the following configuration for illustrative purposes:

- Node versions installed: v12.2.0, v11.9.0, v10.15.3 (default), v8.16.0
- Yarn versions installed: v1.16.0, v1.12.3 (default)
- Tools installed:
    - ember-cli, with binary `ember`:
	    - v3.10.0 on Node v12.2.0 with built-in npm (default)
	    - v3.8.2 on Node v12.2.0 with built-in npm
    - typescript, with binaries `tsc`, `tsserver`:
	    - v3.4.5 on Node v12.2.0 with built-in npm
	    - v3.0.3 on Node v12.2.0 with built-in npm (default)
    - create-react-app, with binary `create-react-app`: v3.0.1 on Node v12.2.0 with built-in npm (default)
    - yarn-deduplicate, with binary `yarn-deduplicate`: v1.1.1 on Node v12.2.0 with built-in npm (notice that this is not a *default*; assume the user ran `volta fetch` )

They also have two projects with the following pins:

- `~/node-only/package.json`:

    ```json
    {
      "volta": {
        "node": "v8.16.0",
      }
    }
    ```

- `~/node-and-yarn/package.json`:

    ```json
    {
      "volta": {
        "node": "v12.2.0",
        "yarn": "v1.16.0"
      }
    }
    ```

### `volta list` (no flags)

#### Human

The format is:

```sh
$ volta list --human
⚡️ Currently active tools:

    Node: v8.16.0 (default)
    Yarn: v1.12.3 (default)
    Tool binaries available:
        create-react-app, ember, tsc, tsserver

See options for more detailed reports by running `volta list --help`.
```

<details><summary>Outside a project</summary>

```sh
$ volta list --human
⚡️ Currently active tools:

    Node: v8.16.0 (default)
    Yarn: v1.12.3 (default)
    Tool binaries available:
        create-react-app, ember, tsc, tsserver

See options for more detailed reports by running `volta list --help`.
```

</details>

<details><summary>In the `node-only` project</summary>

<b>Note:</b> this assumes the implementation of a fix for [volta-cli/volta#436](https://github.com/volta-cli/volta/issues/436).

```sh
$ volta list --human
⚡️ Currently active tools:

    Node: v8.16.0 (current @ ~/node-only/package.json)
    Yarn: v1.12.3 (default)
    Tool binaries available:
        create-react-app, ember, tsc, tsserver

See options for more detailed reports by running `volta list --help`.
```

</details>

<details><summary>In the <code>node-and-yarn</code> project</summary>

```sh
$ volta list --human
⚡️ Currently active tools:

    Node runtime: v12.2.0 (current @ ~/node-and-yarn/package.json)
    Packager: Yarn: v1.16.0 (current @ ~/node-and-yarn/package.json)
    Tool binaries available:
        create-react-app, ember, tsc, tsserver

See options for more detailed reports by running `volta list --help`.
```
    
</details>

#### Plain

The format is:

```sh
$ volta list --plain
runtime node@<version> (default|current @  <project path>)
packager <npm|yarn>@<version> (built-in|default|current @ <project path>)
```

<details><summary>Outside a project</summary>

```sh
$ volta list --plain
runtime node@v10.15.3 (default)
packager yarn@v1.12.3 (default)
```

</details>

<details><summary>In the `node-only` project</summary>

<b>Note:</b> this assumes the implementation of a fix for [volta-cli/volta#436](https://github.com/volta-cli/volta/issues/436).

```sh
$ volta list --plain
runtime node@v8.16.0 (~/node-only/package.json)
packager yarn@v1.12.3 (default)
```

</details>

<details><summary>In the <code>node-and-yarn</code> project</summary>

```sh
$ volta list --plain
runtime node@v12.2.0 (~/node-and-yarn/package.json)
packager yarn@v1.16.0 (~/node-and-yarn/package.json)
```
    
</details>
    
### `volta list --all`

#### Human

The basic format is:

```sh
$ volta list --all --human
⚡️ User toolchain:

    Node runtimes:
        <version> [(default|current @ <project path>)]
    
    Packagers:
        (Yarn|npm):
            <version> [(default|current @ <project path>)]
    
    Tools:
        <package name>
            <version> [(default|current @ <project path>)]
                binaries: [<binary name>]...
                platform:
                    runtime: node@<version>
                    packager: built-in npm|<npm|yarn>@<version>
```

<details><summary>Outside a project directory</summary>

```sh
$ volta list --all --human
⚡️ User toolchain:

    Node runtimes:
        v12.2.0
        v11.9.0
        v10.15.3 (default)
        v8.16.0
    
    Packagers:
        Yarn:
            v1.16.0 (default)
            v1.12.3
    
    Tools:
        create-react-app:
            v3.0.1 (default)
                binaries: create-react-app
                platform:
                    runtime: node@v12.2.0
                    packager: built-in npm
        ember-cli:
            v3.10.0 (default)
                binaries: ember
                platform:
                    runtime: node@v12.2.0
                    packager: built-in npm
            v3.8.2
                binaries: ember
                platform:
                    runtime: node@v12.2.0
                    packager: built-in npm
        typescript:
            v3.4.5
                binaries: tsc, tsserver
                platform:
                    runtime: node@v12.2.0
                    packager: built-in npm
            v3.0.3 (default)
                binaries: tsc, tsserver
                platform:
                    runtime: node@v12.2.0
                    packager: built-in npm
        yarn-deduplicate:
            v1.1.1
                binaries: yarn-deduplicate
                platform:
                    runtime: node@v12.2.0
                    packager: built-in npm
```

</details>

<details><summary>In the <code>node-only</code> project</summary>

```sh
$ volta list --all --human
⚡️ User toolchain:

    Node runtimes:
        v12.2.0
        v11.9.0
        v10.15.3 (default)
        v8.16.0 (current)
    
    Packagers:
        Yarn:
            v1.16.0 (default)
            v1.12.3
    
    Tools:
        create-react-app:
            v3.0.1 (default)
                binaries: create-react-app
                platform:
                    runtime: node@v12.2.0
                    packager: built-in npm
        ember-cli:
            v3.10.0 (default)
                binaries: ember
                platform:
                    runtime: node@v12.2.0
                    packager: built-in npm
            v3.8.2
                binaries: ember
                platform:
                    runtime: node@v12.2.0
                    packager: built-in npm
        tsc:
            v3.4.5
                binaries: tsc, tsserver
                platform:
                    runtime: node@v12.2.0
                    packager: built-in npm
            v3.0.3 (default)
                binaries: tsc, tsserver
                platform:
                    runtime: node@v12.2.0
                    packager: built-in npm
        yarn-deduplicate:
            v1.1.1
                binaries: yarn-deduplicate
                platform:
                    runtime: node@v12.2.0
                    packager: built-in npm
```

</details>

<details><summary>In the <code>node-and-yarn</code> project</summary>

```sh
$ volta list --all --human
⚡️ User toolchain:

    Node runtimes:
        v12.2.0 (current @ ~/node-and-yarn/project.json)
        v11.9.0
        v10.15.3 (default)
        v8.16.0
    
    Packagers:
        Yarn:
            v1.16.0 (default)
            v1.12.3 (current @ ~/node-and-yarn/project.json)
    
    Tools:
        create-react-app:
            v3.0.1 (default)
                binaries: create-react-app
                platform:
                    runtime: node@v12.2.0
                    packager: built-in npm
        ember-cli:
            v3.10.0 (default)
                binaries: ember
                platform:
                    runtime: node@v12.2.0
                    packager: built-in npm
            v3.8.2
                binaries: ember
                platform:
                    runtime: node@v12.2.0
                    packager: built-in npm
        tsc:
            v3.4.5
                binaries: tsc, tsserver
                platform:
                    runtime: node@v12.2.0
                    packager: built-in npm
            v3.0.3 (default)
                binaries: tsc, tsserver
                platform:
                    runtime: node@v12.2.0
                    packager: built-in npm
        yarn-deduplicate:
            v1.1.1
                binaries: yarn-deduplicate
                platform:
                    runtime: node@v12.2.0
                    packager: built-in npm
```

</details>

#### Plain

The basic format is:

```sh
$ volta list --all --plain
runtime node@<version> [(default|current @ <project path>)]
packager <packager>@<version> [(default|current @ <project path>)]
tool <tool name> / <package>@<tool version> [node@<version>] [<yarn|npm>@<version>] [(default|current @ <project path>)]
```

<details><summary>Outside a project directory</summary>

```sh
$ volta list --all --plain
runtime node@v12.2.0
runtime node@v11.9.0
runtime node@v10.15.3 (default)
runtime node@v8.16.0    
packager yarn@v1.16.0 (default)
packager yarn@v1.12.3
tool create-react-app / create-react-app@v3.0.1 node@v12.2.0 npm@built-in (default)
tool ember / ember-cli@v3.10.0 node@v12.2.0 npm@built-in (default)
tool ember / ember-cli@v3.8.2 node@v12.2.0 npm@built-in
tool tsc / typescript@v3.4.5 node@v12.2.0 npm@built-in
tool tsserver / typescript@v3.4.5 node@v12.2.0 npm@built-in
tool tsc / typescript@v3.0.3 node@v12.2.0 npm@built-in (default)
tool tsserver / typescript@v3.0.3 node@v12.2.0 npm@built-in (default)
tool yarn-deduplicate / yarn-deduplicate@v1.1.1 node@v12.2.0 npm@built-in
```

</details>

<details><summary>In the <code>node-only</code> project</summary>

```sh
$ volta list --all --plain
runtime node@v12.2.0
runtime node@v11.9.0
runtime node@v10.15.3 (default)
runtime node@v8.16.0 (current @ ~/node-only/package.json)
packager yarn@v1.16.0 (default)
packager yarn@v1.12.3
tool create-react-app / create-react-app@v3.0.1 node@v12.2.0 npm@built-in (default)
tool ember / ember-cli@v3.10.0 node@v12.2.0 npm@built-in (default)
tool ember / ember-cli@v3.8.2 node@v12.2.0 npm@built-in
tool tsc / typescript@v3.4.5 node@v12.2.0 npm@built-in
tool tsserver / typescript@v3.4.5 node@v12.2.0 npm@built-in
tool tsc / typescript@v3.0.3 node@v12.2.0 npm@built-in (default)
tool tsserver / typescript@v3.0.3 node@v12.2.0 npm@built-in (default)
tool yarn-deduplicate / yarn-deduplicate@v1.1.1 node@v12.2.0 npm@built-in
```

</details>

<details><summary>In the <code>node-and-yarn</code> project</summary>

```sh
$ volta list --all --plain
runtime node@v12.2.0 (current @ ~/node-and-yarn/project.json)
runtime node@v11.9.0
runtime node@v10.15.3 (default)
runtime node@v8.16.0
packager yarn@v1.16.0 (default)
packager yarn@v1.12.3 (current @ ~/node-and-yarn/project.json)
tool create-react-app / create-react-app@v3.0.1 node@v12.2.0 npm@built-in (default)
tool ember / ember-cli@v3.10.0 node@v12.2.0 npm@built-in (default)
tool ember / ember-cli@v3.8.2 node@v12.2.0 npm@built-in
tool tsc / typescript@v3.4.5 node@v12.2.0 npm@built-in
tool tsserver / typescript@v3.4.5 node@v12.2.0 npm@built-in
tool tsc / typescript@v3.0.3 node@v12.2.0 npm@built-in (default)
tool tsserver / typescript@v3.0.3 node@v12.2.0 npm@built-in (default)
tool yarn-deduplicate / yarn-deduplicate@v1.1.1 node@v12.2.0 npm@built-in
```

</details>

### `volta list package <package>`

List all fetched versions of a specific package, along with its associated binaries.

#### Human

The basic format is:

```sh
volta list package <package> --human

    <version> [(default|current @ <project path>)]
        binaries: [<binary name>]...
        platform:
            runtime: node@<version>
            packager: built-in npm|<npm|yarn>@<version>
```

For the TypeScript config specified in the canonical example:

```sh
volta list package typescript --human

    v3.4.5
        binaries: tsc, tsserver
        platform:
            runtime: node@v12.2.0
            packager: built-in npm

    v3.0.3 (default)
        binaries: tsc, tsserver
        platform:
            runtime: node@v12.2.0
            packager: built-in npm
```

#### Plain

The basic format is:

```sh
volta list package <package> --plain
tool <tool> / <package>@<version> node@<version> <npm|yarn>@<built-in|version> [(default|current @ <path>)]
```

For the TypeScript config specified in the canonical example:

```sh
volta list package typescript --plain
tool tsc / typescript@v3.4.5 node@12.2.0 npm@built-in
tool tsserver / typescript@v3.4.5 node@12.2.0 npm@built-in
tool tsc / typescript@v3.0.3 node@12.2.0 npm@built-in (default)
tool tsserver / typescript@v3.0.3 node@12.2.0 npm@built-in (default)
```

### `volta list tool <tool>`

#### Human

The basic format is:

```sh
volta list tool <tool> --human
⚡️ tool <tool> available from:

    <package>@<version> [(default|current @ <project path>)]
        platform:
            runtime: node@<version>
            packager: built-in npm|<npm|yarn>@<version>
```

For the TypeScript config specified in the canonical example:

```sh
volta list tool tsc --human
⚡️ tool tsc available from:

    typescript@v3.4.5
        platform:
            runtime: node@v12.2.0
            packager: built-in npm

    typescript@v3.0.3 (default)
        platform:
            runtime: node@v12.2.0
            packager: built-in npm
```

#### Plain

The basic format is:

```sh
volta list tool <tool> --plain
tool <tool> / <package>@<version> node@<version> <npm|yarn>@<built-in|version> [(default|current @ <path>)]
```

For the TypeScript config specified in the canonical example:

```sh
volta list tool tsc
tool tsc / typescript@v3.4.5 node@12.2.0 npm@built-in
tool tsc / typescript@v3.0.3 node@12.2.0 npm@built-in (default)
```

## Deprecating `volta current`

Since `volta list` subsumes (and substantially extends upon) the functionality of `volta current`, `volta current` should be deprecated when `volta list` is implemented. The deprecation should include a warning that the command will be removed in a future version and information about how to use `volta list --current <tool>` as a replacement.

<!--
This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

This section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.
-->

# Critique
[critique]: #critique

This section discusses the tradeoffs considered in this proposal. This must include an honest accounting of the drawbacks of the proposal, as well as list out all the alternatives that were considered and rationale for why this proposal was chosen over the alternatives. This section should address questions such as:

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

- Should the bare `list` command include:

    - all available package binaries?
    - a subset of package binaries once the number crosses a threshold, with instructions about how to see all?
    - no package binaries?

    And how should that be presented in `plain` vs. `human` mode? Should there be any differences between `plain` and `human` for the bare command (as in the current design)?

- How should the current version be identified? It is currently marked with `(current @ <path>)`. Should this be `(from <path>)` or some other design?

- Should the `list` command accept flags to narrow the search instead of subcommands, e.g. `volta list --node`, `volta list --yarn`, `volta list --package=typescript`, etc.?

<!-- 
- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
-->
