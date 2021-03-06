# Weeder [![Hackage version](https://img.shields.io/hackage/v/weeder.svg?label=Hackage)](https://hackage.haskell.org/package/weeder) [![Stackage version](https://www.stackage.org/package/weeder/badge/lts?label=Stackage)](https://www.stackage.org/package/weeder) [![Linux Build Status](https://img.shields.io/travis/ndmitchell/weeder.svg?label=Linux%20build)](https://travis-ci.org/ndmitchell/weeder) [![Windows Build Status](https://img.shields.io/appveyor/ci/ndmitchell/weeder.svg?label=Windows%20build)](https://ci.appveyor.com/project/ndmitchell/weeder)

The principle is to delete dead code (pulling up the weeds). To do that, run:

* GHC with `-fwarn-unused-binds -fwarn-unused-imports`, which finds unused definitions and unused imports in a module.
* [HLint](https://github.com/ndmitchell/hlint#readme), looking for "Redundant extension" hints, which finds unused extensions.
* The `weeder` tool, which detects functions that are exported internally but not available from outside this package. It also detects redundancies and missing information in your `.cabal` file.

## Running Weeder

To use `weeder` your code must have been built with [`stack`](https://www.haskellstack.org), as it piggy-backs off files that `stack` generates. If you don't normally build with `stack` a simple `stack init && weeder . --build` is likely to be sufficient to get you started.

## What does Weeder detect?

Weeder detects a bunch of weeds, including:

* You export a function `helper` from module `Foo.Bar`, but nothing else in your package uses `helper`, and `Foo.Bar` is not an `exposed-module`. Therefore, the export of `helper` is a weed. Note that `helper` itself may or may not be a weed - once it is no longer exported `-fwarn-unused-binds` will tell you if it is entirely redundant.
* Your package `depends` on another package but doesn't use anything from it - the dependency should usually be deleted. This functionality is quite like [packunused](https://hackage.haskell.org/package/packunused), but implemented quite differently.
* Your package has entries in the `other-modules` field that are either unused (and thus should be deleted), or are missing (and thus should be added). The `stack` tool warns about the latter already.
* A source file is used between two different sections in a `.cabal` file - e.g. in both the library and the executable. Usually it's better to arrange for the executable to depend on the library, but sometimes that would unnecessarily pollute the interface. Useful to be aware of, and sometimes worth fixing, but not always.
* A file has not been compiled despite being mentioned in the `.cabal` file. This situation can be because the file is unused, or the `stack` compilation was incomplete. I recommend compiling both benchmarks and tests to avoid this warning where possible - running `weeder . --build` will use a suitable command line.

Beware of conditional compilation (e.g. `CPP` and the [Cabal `flag` mechanism](https://www.haskell.org/cabal/users-guide/developing-packages.html#configurations)), as these may mean that something is currently a weed, but in different configurations it is not.

I recommend fixing the warnings relating to `other-modules` and files not being compiled first, as these may cause other warnings to disappear.

## Ignoring weeds

If you want your package to be detected as "weed free", but it has some weeds you know about but don't consider important, you can add a `.weeder.yaml` file adjacent to the `stack.yaml` with a list of exclusions. To generate an initial list of exclusions run `weeder . --yaml > .weeder.yaml`.

You may then with to generalise/simplify the `.weeder.yaml` by removing anything above or below the interesting part. As an example of the [`.weeder.yaml` file from `ghcid`](https://github.com/ndmitchell/ghcid/blob/master/.weeder.yaml):

```yaml
- message: Module reused between components
- message:
  - name: Weeds exported
  - identifier: withWaiterPoll
```

This configuration declares that I am not interested in the message about modules being reused between components (that's the way `ghcid` works, and I am aware of it). It also says that I am not concerned about `withWaiterPoll` being a weed - it's a simplified method of file change detection I use for debugging, so even though it's dead now, I sometimes do switch to it.
