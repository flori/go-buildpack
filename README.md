# Buildpack: Go

This is a buildpack for [Go][go].

## Getting Started

Follow the guide at <http://doc.scalingo.com/languages/go>.

There's also a hello world sample app at <https://github.com/Scalingo/sample-go-martini>.

## Example

```
$ ls -A1
.git
vendor
Procfile
web.go

$ scalingo create my-go-app
$ git push scalingo master
```

```
-----> Go app detected
-----> Installing go1.8... done
-----> Running: go install -tags paas ./...
-----> Discovering process types
       Procfile declares types -> web
        Build complete, shipping your container...
         Waiting for your application to boot...
          <-- https://my-go-app.scalingo.io -->
```

This buildpack will detect your repository as Go if you are using either:

- [govendor][govendor]
- [glide][glide]
- [GB][gb]
- [Godep][godep]

This buildpack adds a `paas` [build constraint][https://golang.org/pkg/go/build/], to enable
Scalingo-specific code. See the [App Engine build constraints
article][https://blog.golang.org/the-app-engine-sdk-and-workspaces-gopath] for more.

## govendor specifics

The [vendor.json][vendor.json] spec that govendor follows for its metadata
file allows for arbitrary, tool specific fields. This buildpack uses this
feature to track build specific bits. These bits are encoded in the following
top level json keys:

* `rootPath` (String): the root package name of the packages you are pushing to
  Scalingo. You can find this locally with `go list -e .`. There is no default for
  this and it must be specified. Recent versions of govendor automatically fill
  in this field for you. You can re-run `govendor init` after upgrading to have
  this field filled in automatically, or it will be filled the next time you use
   govendor to modify a dependency.

* `scalingo.goVersion` (String): the major version of go you would like Scalingo to
  use when compiling your code: if not specified defaults to the most recent
  supported version of Go.

* `scalingo.install` (Array of Strings): a list of the packages you want to install.
  If not specified, this defaults to `["."]`. Other common choices are:
  `["./cmd/..."]` (all packages and sub packages in the `cmd` directory) and
  `["./..."]` (all packages and sub packages of the current directory). The exact
   choice depends on the layout of your repository though. Please note that `./...`
   includes any packages in your `vendor` directory.

* `scalingo.additionalTools` (Array of Strings): a list of additional tools that
  the buildpack is aware of that you want it to install. If the tool has
  multiple versions an optional `@<version>` suffix can be specified to select
  that specific version of the tool. Otherwise the buildpack's default version
  is chosen. Currently the only supported tool is `github.com/mattes/migrate` at
  `v3.0.0` (also the default version).

Example with everything, for a project using `go1.8`, located at
`$GOPATH/src/github.com/Scalingo/sample-go-martini` and requiring a single package
spec of `./...` to install.

```json
{
    ...
    "rootPath": "github.com/Scalingo/sample-go-martini",
    "scalingo": {
        "install" : [ "./..." ],
        "goVersion": "go1.8"
         },
    ...
}
```

A tool like jq or a text editor can be used to inject these variables into
`vendor/vendor.json`.

## glide specifics

The `glide.yaml` and `glide.lock` files do not allow for arbitrary metadata, so
the buildpack relies solely on the glide command and environment variables to
control the build process.

The base package name is determined by running `glide name`.

The Go version used to compile code defaults to the latest released version of Go.
This can be overridden by the `$GOVERSION` environment variable. Setting
`$GOVERSION` to a major version will result in the buildpack using the
latest released minor version in that series. Setting `$GOVERSION` to a specific
minor Go version will pin Go to that version. Examples:

```console
$ scalingo env-set GOVERSION=go1.8   # Will use go1.8.X, Where X is that latest minor release in the 1.8 series
$ scalingo env-set GOVERSION=go1.7.5 # Pins to go1.7.5
```

`glide install` will be run to ensure that all dependencies are properly
installed. If you need the buildpack to skip the `glide install` you can set
`$GLIDE_SKIP_INSTALL` to `true`. Example:

```console
$ scalingo env-set GLIDE_SKIP_INSTALL=true
$ git push scalingo master
```

Installation defaults to `.`. This can be overridden by setting the
`$GO_INSTALL_PACKAGE_SPEC` environment variable to the package spec you want the
go tool chain to install. Example:

```console
$ scalingo env-set GO_INSTALL_PACKAGE_SPEC=./...
$ git push scalingo master
```

## Usage with other vendoring systems

If your vendor system of choice is not listed here, create `vendor/vendor.json`
with the following contents, adjusted as needed for your project's root path.

```json
{
    "comment": "For other Scalingo options see: http://doc.scalingo.com/languages/go",
    "rootPath": "github.com/yourOrg/yourRepo",
    "scalingo": {
        "sync": false
    }
}
```

## Hacking on this Buildpack

To change this buildpack, fork it on GitHub & push changes to your fork. Ensure
that tests have been added to the `test/run` script and any corresponding fixtures to
`test/fixtures/<fixture name>`.

### Tests

Requires docker.

```console
$ make test
```

## Using with cgo

The buildpack supports building with C dependencies via [cgo][cgo]. You can set
config vars to specify CGO flags to specify paths for vendored dependencies. The
literal text of `${build_dir}` will be replaced with the directory the build is
happening in. For example, if you added C headers to an `includes/` directory,
add the following config to your app: `scalingo env-set CGO_CFLAGS='-I${
build_dir}/includes'`. Note the used of `''` to ensure they are not converted to
local environment variables

## Using a development version of Go

The buildpack can install and use any specific commit of the Go compiler when
the specified go version is `devel-<short sha>`. The version can be set either
via the appropriate vendoring tools config file or via the `$GOVERSION`
environment variable. The specific sha is downloaded from Github w/o git
history. Builds may fail if GitHub is down, but the compiled go version is
cached.

When this is used the buildpack also downloads and installs the buildpack's
current default Go version for use in bootstrapping the compiler.

Build tests are NOT RUN. Go compilation failures will fail a build.

No official support is provided for unreleased versions of Go.

## Passing a symbol (and optional string) to the linker

This buildpack supports the go [linker's][go-linker] ability (`-X symbol value`)
to set the value of a string at link time. This can be done by setting
`GO_LINKER_SYMBOL` and `GO_LINKER_VALUE` in the application's config before
pushing code. If `GO_LINKER_SYMBOL` is set, but `GO_LINKER_VALUE` isn't set then
`GO_LINKER_VALUE` defaults to [`$SOURCE_VERSION`][source-version].

This can be used to embed the commit sha, or other build specific data directly
into the compiled executable.

## Testpack

This buildpack also supports the testpack API.


## Deploying

```console
$ make publish # && follow the prompts
```

### New Go version

1. Edit `files.json`, and add an entry for the new version, including the SHA,
   which can be copied from golang.org.
1. Update `data.json`, to update the `VersionExpansion` object.
1. run `ACCESS_KEY='THE KEY' SECRET_KEY='THE SECRET KEY' bin/sync-files.sh`.
   This will download everything from the bucket, plus any missing files from
   their source locations, and verify their SHAS, then upload anything missing
   from the bucket back to the s3 bucket.
1. Commit and push.

[go]: http://golang.org/
[buildpack]: http://doc.scalingo.com/buildpacks/
[go-linker]: https://golang.org/cmd/ld/
[godep]: https://github.com/tools/godep
[govendor]: https://github.com/kardianos/govendor
[gb]: https://getgb.io/
[quickstart]: http://doc.scalingo.com/languages/go/
[build-constraint]: http://golang.org/pkg/go/build/
[app-engine-build-constraints]: http://blog.golang.org/2013/01/the-app-engine-sdk-and-workspaces-gopath.html
[source-version]: http://doc.scalingo.com/app/build-environment
[cgo]: http://golang.org/cmd/cgo/
[vendor.json]: https://github.com/kardianos/vendor-spec
[gopgsqldriver]: https://github.com/jbarham/gopgsqldriver
[glide]: https://github.com/Masterminds/glide
