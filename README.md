# From Bazel

This repo demonstrates how to bootstrap a development environment for Frontend
Web programming assuming you have Bazel installed on your machine.

You don't need to install any frontend tooling like Node.js, npm, yarn, etc.

This illustrates a typical workflow for a backend engineer who already uses
Bazel to build code such as a Java or C++ backend, and wants to add some
frontend code to their build. Such an engineer might work at a company where the
corporate IT department manages the image for developer machines and doesn't
give developers administrator rights on their machine.

## Prerequisites

Bazel 0.17 or greater installed.

## Bootstrapping a frontend build

Most frontend tooling runs on the Node.js runtime. We'll need to use
`rules_nodejs` to get this toolchain.

This example also assumes you'd like to develop in a typed superset of
JavaScript, called TypeScript.

Add this to your `WORKSPACE` file (or create an empty one if starting from
scratch):

```python
http_archive(
    name = "build_bazel_rules_typescript",
    url = "https://github.com/bazelbuild/rules_typescript/archive/0.19.1.zip",
    strip_prefix = "rules_typescript-0.19.1",
)

# Fetch our Bazel dependencies that aren't distributed on npm
load("@build_bazel_rules_typescript//:package.bzl", "rules_typescript_dependencies")
rules_typescript_dependencies()

# Setup the NodeJS toolchain
load("@build_bazel_rules_nodejs//:defs.bzl", "node_repositories", "yarn_install")
node_repositories()

# Setup TypeScript toolchain
load("@build_bazel_rules_typescript//:defs.bzl", "ts_setup_workspace")
ts_setup_workspace()
```

Let's assume we are in a repository where the frontend code will live in a
subdirectory. So create a `frontend` directory and `cd` into there.

Now we can run a package manager to fetch the frontend tooling like the
TypeScript compiler. Either `npm` or `yarn` are typically used for this purpose.
We'll use `yarn` in this example.

The package manager expects a file called `package.json` which specifies the
packages and their versions. To create such a file, we'll just call the `init`
command of the package manager:

```sh
$ bazel run -- @nodejs//:bin/yarn init -y
```

> On Windows, the target is `@nodejs//:bin/yarn.cmd`

Now we need to add a dependency on the TypeScript compiler, and its Bazel
support package.

```sh
$ bazel run -- @nodejs//:bin/yarn add typescript @bazel/typescript
```

> Again, on Windows the target is `@nodejs//:bin/yarn.cmd`

## Teaching Bazel to find the frontend dependencies

The previous step created a `node_modules` directory in your project. That's
useful so your editor can use the matching version of TypeScript as Bazel will.

However, we'll let Bazel manage its own copy of the dependencies. To do that,
add to your `WORKSPACE`:

```python
yarn_install(
    name = "npm",
    package_json = "//frontend:package.json",
    yarn_lock = "//frontend:yarn.lock",
)
```

> We named this rule "npm" because the repository of frontend dependencies is
> named "npm" and is found at http://npmjs.com.

That rule references the `package.json` file we created with `yarn init` and
also a `yarn.lock` file which pins the versions of our transitive dependencies
so that everyone gets the same build results.

Since we've referenced those files in the `frontend` package, we'll also need
a `BUILD.bazel` file in that directory, which could be empty.

## Try running TypeScript

Now we can run the TypeScript compiler manually to verify that it works:

```sh
$ bazel build @npm//:typescript/tsc
$ ../bazel-bin/external/npm/typescript/tsc
... some output
```

> Note, we don't use `bazel run` here because it sets the current working
> directory to the Bazel runfiles by default, and in this case we want to work
> in our current directory.

In order to compile TypeScript code, we need a `tsconfig.json` configuration
file for the compiler. We can use the same `init` trick as with `yarn` above:

```sh
$ ../bazel-bin/external/npm/typescript/tsc --init
```

## First ts_library rule

Bazel will run the TypeScript compiler as needed on library rules whose inputs
have changed since the last build.

First create a simple TypeScript application, `frontend/app.ts`

```typescript
const el: HTMLDivElement = document.createElement('div');
el.innerText = 'Hello, TypeScript';
el.className = 'ts1';
document.body.appendChild(el);
```

> Your editor should give you help if you type this code by hand,
> since TypeScript supplies the API for the `document` variable.

Now edit your `frontend/BUILD.bazel` file to contain

```python
load("@build_bazel_rules_typescript//:defs.bzl", "ts_library", "ts_devserver")
ts_library(name = "app", srcs = [":app.ts"], tsconfig = ":tsconfig.json")
```

> We could have skipped the `tsconfig` attribute on `ts_library` if our config
> was found in the default location, which is to have the `tsconfig.json` file
> in the Workspace root, or else to add an `alias` rule to the root
> `BUILD.bazel` file like
>
> ```
> alias(name = "tsconfig.json", actual = "//frontend:tsconfig.json")
> ```

Finally, we can build the code:

```sh
$ bazel build :app
Target //frontend:app up-to-date:
  bazel-bin/frontend/app.d.ts
```

## Serve the app to a browser

> 😢 This step is currently broken on Windows 😢

For scalability, we use an optimized devserver written in Go. This is compiled
from source, thanks to Bazel's ability to work in many languages.

First you'll need a few lines added to `WORKSPACE`:

```python
load("@io_bazel_rules_go//go:def.bzl", "go_rules_dependencies", "go_register_toolchains")
go_rules_dependencies()
go_register_toolchains()
```

Then add this to your `frontend/BUILD.bazel` file:

```python
ts_devserver(name = "devserver", deps = [":app"])
```

You can now run the server with

```
$ bazel run :devserver
Server listening on http://...
```

Click the link that's printed there, and you should see "Hello, TypeScript"
appear in the browser.
