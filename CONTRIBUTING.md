# Contributing

To make contributing more convenient we have a vagrant setup, [read more here](#vagrant)

To build and install `terraform-cdk` locally you need to install:

- Node version 12.16+
- Go 1.16+
- dotnet (v3.1.0)
- mvn
- pipenv

Alternatively you can work on the CDK from within a docker container with the image `docker.mirror.hashicorp.services/hashicorp/jsii-terraform`, e.g.:

```shell
$ docker run -it --rm -w=/home -v (pwd):/home docker.mirror.hashicorp.services/hashicorp/jsii-terraform
```

or through [Visual Studio Code Remote - Containers](https://code.visualstudio.com/docs/remote/containers).

## Vagrant  

1. Install [vagrant](https://www.vagrantup.com/docs/installation)
2. Boot your vagrant machine: `vagrant up <linux|windows>`
3. SSH into machine: `vagrant ssh <linux|windows>`
4. Follow Linux or Windows instructions below

### Linux

```sh
cd terraform-cdk
# follow instructions below from here on
```

### Windows

Due to yarn's usage of symlinks we need to manually sync back and forth between the host machine and the windows VM. We have a special command in place that can be used:

```bat
<!-- This gives you access to the installed binaries like node or python -->
refreshenv

<!-- For a one time copy from your host system to the vm run -->
.\terraform-cdk-origin\tools\vagrant\windows-copy-to-vm.bat

<!-- In development you might want to sync files more often, run this and abort it with ctrl+C -->
.\terraform-cdk-origin\tools\vagrant\windows-sync-to-vm.bat

<!-- Go to the working environment and follow instructions below -->
cd .\terraform-cdk

<!-- For a one time copy from your vm to the host system run -->
.\terraform-cdk-origin\tools\vagrant\windows-copy-to-host.bat

```

## Getting Started

Clone the repository:

```shell
$ git clone https://github.com/hashicorp/terraform-cdk.git
```

To compile the `terraform-cdk` binary for your local machine:

```shell
$ yarn install
$ yarn build
```

## Examples

We have a few top level script commands which are executed with Lerna to make the handling of examples easier:

```
yarn examples:reinstall // -> reinstall dependencies in Python examples
yarn examples:build // -> fetch providers for examples and build them
yarn examples:synth // -> synth all examples
yarn examples:integration // -> run all of the above
```

For this work, each example needs a `package.json` with at least a minmal config like this:

```json
{
  "name": "@examples/[LANGUAGE]-[EXAMPLE_NAME]",
  "version": "0.0.0",
  "license": "MPL-2.0",
  "scripts": {
    "reinstall": "rm Pipfile.lock && pipenv --rm && pipenv install", // Python only
    "build": "cdktf get",
    "synth": "cdktf synth"
  }
}
```

Lerna is filtering for the `@examples/` prefix in the `name` field.

## Development

For development, you'd likely want to run:

```shell
$ yarn watch
```

This will watch for changes for the packages `cdktf` and `cdktf-cli`.

## Tests

If you just want to run the tests:

```shell
$ yarn test
```

To run integration tests, package and run integration tests.

```shell
$ yarn package
$ yarn integration
```

## Local Usage

### Monorepo Examples

The easiest way to use this locally is using one of the [examples](./examples). They are setup as part of the monorepo and reference the local packages.

#### Typescript

All Typescript [examples](./examples/typescript) leverage yarn workspaces to directly reference symlinked packages. If you don't have `./node_modules/.bin` in your `$PATH`, you can use `$(yarn bin)/cdktf` to use the symlinked CLI.

#### Python

For Python [examples](./examples/python), packages are referenced from `./dist`, there's no symlinking possible for live code updates. You'll have to explictly run `yarn package` to create new packages to be referenced in the Pipefile.

#### Java

For Java [examples](./examples/java), packages are referenced from `./dist`, there's no symlinking possible for live code updates. You'll have to explictly run `yarn package` to create new packages to be referenced in the pom.

#### C#

For C# [examples](./examples/csharp), packages are referenced from `./dist`, there's no symlinking possible for live code updates. You'll have to explictly run `yarn package` to create new packages to be referenced in the project.

### Outside of this Monorepo

If you want to use the libraries and cli from the repo for local development, you can make use of `yarn link`.

### Setup

Unfortunately, there's an [issue](https://github.com/yarnpkg/yarn/issues/891) with globally linked binaries. This requires you to run the following:

```
yarn config set prefix $(npm config get prefix)
```

If you'd want this permanently, you can add this line to your profile settings (`~/.bashrc`, `~/.zshrc`, or `~/.profile`, etc.)

### Create link

Let's link `cdktf` and `cdktf-cli`, run the following the repository root folder:

```shell
$ yarn link-packages
$ cdktf --version
0.0.0
```

When the version equals `0.0.0` everything worked as expected. If you see another version, try uninstalling `cdktf-cli` with `npm` or `yarn`.

### Build & Package

```shell
$ yarn build && yarn package
$ export CDKTF_DIST=$(pwd)/dist
```

### Create local project

```shell
$ mkdir ~/my-local-cdktf-example
$ cd ~/my-local-cdktf-example
$ cdktf init --template typescript --local
```

Please note, that this will reference the built packages in `$CDKTF_DIST`. This means, it will reflect code changes only after repeating `yarn build && yarn package` and running an explicit `yarn install` again.

Reference the previously [linked](#create-link) `cdktf` package in our newly created project:

```shell
$ cd ~/my-local-cdktf-example
$ yarn link "cdktf"
```

From here on both, the `cli` and the `cdktf` packages are linked and changes will be reflected immediatlely.

## Rebasing contributions against main

PRs in this repo are merged using the [`rebase`](https://git-scm.com/docs/git-rebase) method. This keeps
the git history clean by adding the PR commits to the most recent end of the commit history. It also has
the benefit of keeping all the relevant commits for a given PR together, rather than spread throughout the
git history based on when the commits were first created.

If the changes in your PR do not conflict with any of the existing code in the project, then Github supports
automatic rebasing when the PR is accepted into the code. However, if there are conflicts (there will be
a warning on the PR that reads "This branch cannot be rebased due to conflicts"), you will need to manually
rebase the branch on main, fixing any conflicts along the way before the code can be merged.

## Feature Flags

Sometimes we want to introduce new breaking behavior because we believe this is
the correct default behavior for the CDK for Terraform. The problem of course is that breaking
changes are only allowed in major versions and those are rare.

To address this need, we have a feature flags pattern/mechanism. It allows us to
introduce new breaking behavior which is disabled by default (so existing
projects will not be affected) but enabled automatically for new projects
created through `cdktf init`.

The pattern is simple:

1. Define a new const under
   [cdktf/lib/features.ts](https://github.com/hashicorp/terraform-cdk/blob/main/packages/cdktf/lib/features.ts)
   with the name of the context key that **enables** this new feature (for
   example, `EXCLUDE_STACK_ID_FROM_LOGICAL_IDS`).
2. Use `node.tryGetContext(ENABLE_XXX)` to check if this feature is enabled
   in your code. If it is not defined, revert to the legacy behavior.
3. Add your feature flag to the `FUTURE_FLAGS` map in
   [cdktf/lib/features.ts](https://github.com/hashicorp/terraform-cdk/blob/main/packages/cdktf/lib/features.ts).
   This map is inserted to generated `cdktf.json` files for new projects created
   through `cdktf init`.
4. In your PR title (which goes into CHANGELOG), add a `(under feature flag)` suffix. e.g:

   ```
   fix(core): top level constructs should omit stack id from name (under feature flag)
   ```

5. Under `BREAKING CHANGES` in your commit message describe this new behavior:

   ```
   BREAKING CHANGE: top level resource names for new projects created through "cdktf init"
   will omit the stack id from their name. This is enabled through the flag
   `excludeStackIdFromLogicalIds` in newly generated `cdktf.json` files.
   ```

In the next major version of the
CDKTF we will either remove the
legacy behavior or flip the logic for all these features and then
reset the `FEATURE_FLAGS` map for the next cycle.
