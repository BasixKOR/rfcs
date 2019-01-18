- Feature Name: toolchain
- Start Date: 2018-12-07
- RFC PR: 
- Notion Issue: 

# Summary
[summary]: #summary

This RFC describes a design for the Notion's central unifying design concept: the **toolchain**.

# Motivation
[motivation]: #motivation

The Notion toolchain is designed with several goals:

- **Reproducible project tooling.** With this design, Notion should make it convenient and reliable for all contributors on a project to get the same exact version of the Node runtime, the package manager, and any package binaries configured for use by `package.json`.
- **Reconciling the use of npm as a software distribution platform with best practices.** It's considered a best practice for project development to avoid ever installing global packages. But npm is a popular and convenient platform for distributing command-line tools. The toolchain design reconciles this tension by isolating _user tools_, which are installed for personal use, making them invisible to JS project scripts, and distinguishing them from _project tools_.
- **Install and forget.** This design solves the problem of _tool bitrot_, where a tool stops working because of Node upgrades. It also avoids the need to reinstall global tools every time a new Node version is provisioned. With Notion, once you install a user tool and get it working, it keeps working unless you deliberately decide to change or uninstall it.
- **Low cognitive overhead via statelessness.** This design allows Notion users to generally avoid thinking at all about the version of Node that's currently installed, instead relying on saved configuration to ensure that tools and projects have already declaratively specified their Node platform version.

## User stories

To demonstrate this, here are three representative user stories.

### Project tools: `tsc`

A project maintainer selects a version of TypeScript in `package.json`:

```js
  ...
  "dependencies": {
      "typescript": "^3.2"
  },
  "platform": {
      "node": "10.10.0"
  }
  ...
```

and the lockfile pins TypeScript to version 3.2.2.

Users can call the TypeScript compiler directly as long as they've installed it:
```
notion install typescript
```
Afterwards, running `tsc` from within the project directory runs version 3.2.2 of the TypeScript compiler, using Node 10.10.0 as the runtime.

### User tools: `surge`

An end user of the [surge.sh](https://surge.sh) service installs their CLI tool that is deployed as an npm package:
```
notion install surge
```
Assuming the `surge` tool selects Node 11.4.0 in its `"platform"` manifest (it's OK if not; more details below), the CLI tool is installed in the user toolchain and pinned to Node 11.4.0.

From this point on, unless the user changes their toolchain, running
```
surge
```
from the command-line always runs this tool using Node 11.4.0.

### User tools in projects: `pexx`

The [project-explorer](https://sdras.github.io/project-explorer-site/) tool is a useful CLI tool that lets you inspect JS projects, but is not itself a tool that a project would use in its scripts.

A user can install this tool to their toolchain:
```
notion install project-explorer
```
Assuming the `project-explorer` manifest selects Node 8.12.0 in its manifest, the tool is installed pinned to that version of the Node runtime.

When the user runs
```
pexx myproject
```
the `pexx` binary runs with Node 8.12.0---even if `myproject` specifies a different version of Node in its `"platform"` spec.

# Pedagogy
[pedagogy]: #pedagogy

This section lists the set of concepts that users may encounter using Notion. The first two, **tools** and the **toolchain**, are central to using Notion. The latter two, **shims** and **platforms**, are lower-level primitives that may be helpful for more implementation-oriented users.

## Tools

There are three types of tools:

- **Node runtime:** The version of Node itself, which in particular dictates which `node` binary gets invoked.
- **Package manager:** A version of npm or a version of Yarn. (In the future we may want to add support for other package managers such as pnpm or tink.)
- **Package binary:** An executable published and distributed as part of an npm package.

## Toolchain

The toolchain is a set of tools. The user adds to their toolchain with `notion install` and removes with `notion uninstall`.

Each tool installed in the toolchain has a default version and platform, which can be overridden when the tool is invoked in a project that has a dependency on that tool.

## Primitive: Shims

Notion shims intercept calls to tools and redirect execution to the right executable based on the environment and current directory.

Users may already be familiar with shims, since these are commonly used by other version managers.

## Primitive: Platforms

A **platform spec** is a complete description of a version of the Node platform:

- An exact version of the Node runtime.
- An optional exact version of npm.
- An optional exact version of Yarn.

A **platform image** is an immutable instantiation of a platform spec on disk. It can be thought of analogously to a container image, but for the Node platform as opposed to an operating system.

# Details
[details]: #details

## Toolchain

A toolchain is a pair of a platform image and a set of pinned package binaries.

## Platform specs

In the manifest, an omitted npm version defaults to the version bundled with the specified Node runtime. An omitted Yarn version defaults to Yarn being unavailable, meaning that invoking `yarn` will produce an error. Both of these can be explicitly set to `null`, which means the specified tool is unavailable.

## Platform images

The definition of a platform image is a pair consisting of a Node version and a Yarn version.

A Node version is a pair consisting of an exact Node runtime version, and either an exact npm version or `null`.

A Yarn version is either an exact Yarn version or `null`.

## Pinning

The `"platform"` section of `package.json` selects the platform image associated with a package. (**Note:** This is a change from earlier versions of Notion, which used the key `"toolchain"`, since the toolchain consists of _both_ the platform image _and_ the pinned package binaries -- see above.)

Users can pin the platform by manually editing the `package.json` `"platform"` section or via [`notion pin`](https://github.com/notion-cli/rfcs/pull/24).

## Installation

The `notion install` command installs a tool to the user's toolchain.

When installing a version of the Node runtime, Notion also installs the default version of npm bundled with that verison of Node. This can be overridden with a subsequent `notion install npm` command.

When installing a package binary to the user toolchain, Notion checks the package for a `"platform"` key to pin the user tool to a specific platform image. If there is no `"platform"` key in the package manifest, it defaults to the current platform image configured in the user toolchain. If there is no current platform image, `notion install` fails with an error message suggesting the user choose at least a Node runtime version.

### Overriding the associated platform

When used for a package tool, `notion install` accepts an optional `--node` parameter for overriding the package's specified platform version:

```
notion install surge --node=latest
```

## Uninstallation

The `notion uninstall` command uninstalls a tool from the user's toolchain.

## Updating

Taken together, `notion uninstall` and `notion install` can be used to update a tool. Note that as the tool evolves over time, its authors may update its platform to newer Node versions. So as users upgrade their tools they will automatically get platform upgrades for the tool. But they also continue to be assured they are getting a version of the platform that the tool was tested with.

The `notion update` command is a shorthand command for doing the same thing:
```
notion update surge
```

# Critique
[critique]: #critique

It's natural to question whether pinning for reproducibility is worth the cost of extra fetching. An alternative approach would be to allow projects to specify less precise version requirements (such as the ranges typically expressed in the `"engines"` field of `package.json`) and assume most differences will be benign. However, behavioral divergences between versions of Node do happen and are tricky bugs to nail down. Putting in extra work up front to ensure that these divergences cannot happen, by construction should pay dividends when scaled across the Node ecosystem. And over time, we can investigate optimization techniques to save time and disk space for fetching multiple similar versions.

Theoretically, it might make more sense to put the `"platform"` section in a lockfile. But since there isn't a standardized single lockfile format for JS, and those formats aren't extensible, and we don't want to impose a whole new file to add to JS projects, using the package manifest seemed like the least imposition on users.

Another reasonable criticism is that pinning the Node version for tools in the user toolchain means that users will not automatically benefit from performance and security improvements in Node. There are a couple of reasons this is outweighed by the benefits of pinning. First, as described above, updating tools will typically get platform updates. Second, users can still override the default with the `--node` parameter.

# Unresolved questions
[unresolved]: #unresolved-questions

- What about the (rarer) cases of user tools that want to work with the current project’s toolchain choice instead of a statically-bound choice? For example, tools that wrap the Node REPL. Maybe a special keyword for the platform like `"node": "user"`? We'd need to think through the semantics.
- Should we consider some kind of update notification mechanism to inform you when your user tools are out of date? Probably at least something similar to `brew outdated` so users can explicitly ask for the information.
- What kinds of performance optimizations can we offer to make stateless platform images super fast?
- What about scenarios like CI and testing matrices, where users want to be able to specify different platform configurations without having to change the configuration file? Perhaps workflows similar to `ember try`?
