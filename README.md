# A collaborative approach to Haskell development plugins

When we talk about plugins we usually mean a mechanism to extend *existing
software* with new functionality.  The new functionality can be provided by a
third-party without the need to interact with the original author of the
software.

For the purpose of this document we make additional assumptions:

- We are interested in plugins that are useful while developing Haskell
  packages.  The existing software will typically be a testing framework, a
  benchmarking tool or some other development tool.

- Using a specific plugin is a per user decision. Different users collaborating
  on the same Haskell package may choose to use different sets of plugins
  during development.

While this kind of plugins are common in other programming language
communities, they are not widely used in the Haskell community and they lack
support from Haskell build tools.

The first part of this document describes the current state of Haskell
development plugins, using custom formatters for Hspec as the motivating
example.  It then discusses limitations of the current situation and goes on to
describe a crude way on how to overcome these limitations by extending
`stack`.

The second part of this document describes a more general approach to Haskell
development plugins, first conceptually and then exploring how a concrete
implementation for `stack` might look like.
  
## Motivating example: Using a custom formatter with Hspec

This section lists the steps that are currently required to use a custom
formatter with Hspec, discusses the limitations of the current approach, and
describes a crude way on how to overcome these limitations.

### How to use a custom formatter?

When a user runs a Hspec test suite, Hspec will generate a test report.
Different users have different preferences on what information should be
included in the test report and how it should be formatted.

A Hspec user can customize the generated test report by defining a custom
formatter.  A custom formatter is typically distributed as a Haskell package.

For the purpose of this document we assume that there is a package `hspec-fancy`
that provides a custom formatter `Test.Hspec.Fancy.formatter`.  To use this
formatter a user has to do the following:


1. Add `hspec-fancy` as a dependency to `package.yaml` (or the `.cabal` file)
1. Tell `hspec-discover` to use the formatter by changing the test driver to:
    ```haskell
    {-# OPTIONS_GHC -F -pgmF hspec-discover -optF --formatter=Test.Hspec.Formatters.progress #-}
    ```

### Limitations of the current approach

With this approach we select a custom formatter by modifying project specific
files that are under revision control.  This results in the following
limitations:

- Two users collaborating on the same project can not easily choose to use
  different formatters.
- A user can not easily choose to use the same formatter for all the projects
  that she is working on.

### A crude way on how to overcome these limitations

My assumption is that overcoming these limitations will require support from
the build tool that is used to build and test the Haskell package under
development.

This section describes a crude way on how to overcome these limitations in the
context of `stack`.

From a users perspective, we want to specify a custom formatter, and we want to
do so without modifying any project specific files.  So for our running example
let's assume the user specifies that she wants to use
`Test.Hspec.Fancy.formatter` in a configuration file `~/.hspec-plugins.yaml`.
The content of that file might look like this:

```yaml
# ~/.hspec-plugins.yaml
- hspec-fancy:Test.Hspec.Fancy.formatter
```

Now, when `hspec-discover` is invoked, it can read that configuration file and
generate a suitable test driver that uses `Test.Hspec.Fancy.formatter`.

However, when `stack` tries to build the test suite it will fail due to a
missing dependency on `hspec-fancy`.

For that reason we modify `stack` to read `~/.hspec-plugins.yaml` as well and
exctract the `hspec-fancy` dependency from that file.  `stack` will then do the
following:

1. make `hspec-fancy` (and its transitive dependencies) available during
   development
1. transparently add `hspec-fancy` as a dependency to `package.yaml` (or the
   `.cabal` file) whenever it reads that file

## A more general approach to Haskell development plugins

The described solution is a collaborative approach between a *user*, zero or
more plugin *consumers* and a *build tool*.

 - The user is a person who wants to use plugins during the development of a
   Haskell package.

 - A consumer is some existing software that the user wants to extend with a
   plugin.

 - The build tool is a development tool that builds the Haskell package and
   runs its tests.  The build tool is responsible for making any plugins
   available, and for making potential consumers aware of any plugins.


### Checklist for a suitable solution

A suitable solution to the problem is a solution that meets the following
requirements.  It should

- work on a per project and per user basis
- not require modifications to any project specific files
- degrade gracefully (specifically, build tools that do not support development plugins should continue to work)
- give the build tool the final say (e.g. it should be possible for a build
  tool to ignore plugins for production builds)
- be DRY (specifically, the user should not be required to provide the same information in multiple places)

### Conceptual approach

1. The user defines lists of plugins, grouped by potential consumers, that she
   wishes to use during development.  This can e.g. be in a configuration file.
1. The build tool makes any plugins (and their transitive dependencies)
   available during development.
1. Whenever the user invokes build commands (such as `stack build`, `stack
   test`, or `stack exec`) the build tool communicates any plugins through the
   process environment to potential consumers.
1. A consumer inspects the process environment for relevant plugins and
   consumes them as desired

### A concrete implementation

The following section describes a concrete implementation of the proposed
solution in the context of `stack`.  The main purpose here is to illustrate the
solution.  Implementation details such as file names, concrete syntax of
configuration files, etc. are not meant to be final.

#### How a user requests a plugin

A user lists plugins that she wishes to use during development in
`~/.haskell-plugins.yaml` or `.haskell-plugins.yaml`.  Syntactically, this
could look like the following:

```yaml
# ~/.haskell-plugins.yaml
hspec:
- name: hspec-fancy # released package from e.g. Hackage / Stackage
  plugin: Test.Hspec.Fancy.formatter

- name: hspec-foo
  github: sol/hspec-foo # package from GitHub
  ref: master
  plugin: Test.Hspec.Foo.plugin

- ..

criterion:
- ..

yesod:
- ..
```

#### What the build tool does

`stack` reads the list of requested plugins from `~/.haskell-plugins.yaml` and
`.haskell-plugins.yaml`.  Whenever a `stack` build command is invoked `stack`
does the following two steps:

1. `stack` transparently injects any requested plugins into the dependency list
   of the Haskell package under development.  As we don't know which component
   uses the plugin we inject the plugin into every component.
1. `stack` augments the process environment to make potential consumers aware
   of any plugins.  For the example above `stack` e.g. sets the environment
   variable `HASKELL_PLUGINS_hspec` to
   `Test.Hspec.Fancy.formatter,Test.Hspec.Foo.plugin` (and
   `HASKELL_PLUGINS_criterion` and `HASKELL_PLUGINS_yesod` to their respective values).
   
Note that we only do these two steps during development.  We explicitly abstain
from doing so for production builds and when a package is installed as a
dependency of an other package.

#### How a consumer consumes plugins

How a consumer makes use of a plugin is up to that consumer and can be subject
to further exploration.

However that might look in detail, at some point some *plugin aware code* of
the consumer has to run.  That code will inspect the process environment for
any relevant plugins and consume them as desired.

Possible ways to consume a plugin could be by

- referencing the plugin during code generation (e.g. when the plugin aware
  code is a preprocessor or code generator like `markdown-unlit` or `hspec-discover`)
- referencing the plugin in a `template-haskell` splice
- loading the plugin dynamically during runtime

For the specific case of `hspec-discover`: When `hspec-discover` is invoked it
checks the process environment for `HASKELL_PLUGINS_hspec` and includes any
plugins when generating the test driver.


### FAQ

#### For what reasons does the build tool communicate plugins to potential consumers through the process environment?  Couldn't a plugin consumer read the configuration file instead?

While at first it might seem viable for a plugin consumer to directly read the
configuration file, it has the following shortcomings:

- The build tool would loose the final say over whether plugins are used or
  not.  Consequently the build tool could not disable plugins for production
  builds or when a package is installed as a dependency of an other package.
- Plugin consumers would include plugins even if the used build tool does not
  support them.  This would result in build errors with legacy build tools.
