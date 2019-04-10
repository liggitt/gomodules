- [$GOPATH: the old way](#gopath-the-old-way)
  - [Building](#building)
    - [What happens](#what-happens)
    - [Benefits](#benefits)
    - [Potential problems](#potential-problems)
    - [Questions/Answers](#questionsanswers)
  - [Distribution](#distribution)
    - [What happens](#what-happens-1)
    - [Benefits](#benefits-1)
    - [Potential problems](#potential-problems-1)
- [go modules: the new way](#go-modules-the-new-way)
  - [Defining a module](#defining-a-module)
    - [What happens](#what-happens-2)
    - [Benefits](#benefits-2)
  - [Resolving dependencies automatically](#resolving-dependencies-automatically)
    - [What happens](#what-happens-3)
    - [Benefits](#benefits-3)
  - [Distribution](#distribution-1)
    - [What happens](#what-happens-4)
    - [Benefits](#benefits-4)
  - [Replacing resolved module versions or source](#replacing-resolved-module-versions-or-source)
    - [Caveats](#caveats)
  - [Major versions](#major-versions)
    - [Benefits of semantic import versioning](#benefits-of-semantic-import-versioning)
    - [Caveats of semantic import versioning](#caveats-of-semantic-import-versioning)
  - [Tips](#tips)
    - [Getting started](#getting-started)
    - [Querying](#querying)
    - [Cleaning up](#cleaning-up)
    - [Low-level go.mod manipulation](#low-level-gomod-manipulation)
    - [Local module cache](#local-module-cache)
    - [Module sources](#module-sources)
  - [Additional Reading](#additional-reading)

## $GOPATH: the old way

### Building

Source was located under `$GOPATH/src`, whether we liked it or not.

```sh
go get github.com/liggitt/a
go get github.com/liggitt/b
```

```sh
cat $GOPATH/src/github.com/liggitt/a/make-cake.go
```

> ```go
> package main
> 
> import (
> 	"github.com/liggitt/a/helpers"
> 	"github.com/liggitt/b"
> )
> 
> func main() {
> 	b.Cake()
> 	helpers.PrintSuccess()
> }
> ```

```sh
cat $GOPATH/src/github.com/liggitt/a/helpers/helpers.go
```

> ```go
> package helpers
> 
> import "fmt"
> 
> func PrintSuccess() {
> 	fmt.Println("cake!")
> }
> ```

```sh
cat $GOPATH/src/github.com/liggitt/b/cake.go
```

> ```go
> package b
> 
> func Cake() {
> 	// TODO: implement. the cake is a lie.
> }
> ```

```sh
cd $GOPATH/src/github.com/liggitt/a
go run .
```

> ```sh
> cake!
> ```

#### What happens

1. go gathers dependencies of my main package (e.g. `go list -deps .`)
2. my main package imports `github.com/liggitt/a/helpers` and `github.com/liggitt/b`
3. go searches for those import paths in the following places, in order:
   1. `github.com/liggitt/a/vendor/<import-path>`
   2. `$GOPATH/src/<import-path>`
   3. `$GOROOT/src/<import-path>`
4. in this case it finds them in `$GOPATH`, builds, and runs

#### Benefits

* Hermetic (if you fully control all the content in `$GOPATH`)
* No network access
* Allows local development

#### Potential problems

* What if I need multiple things in my `$GOPATH` at different code levels?
  * workaround: have a separate `$GOPATH` per project
  * workaround: place a copy of all code inside a vendor dir in each project
* What if I don't want to structure my directories like $GOPATH requires?
  * you don't always get what you want
* What if source in `$GOPATH` accidentally drifts from the authoritative source?
  * whatever is in `$GOPATH` gets used, for better or worse:
    * can be bad if you weren't expecting it
    * can be good if you're intentionally developing multiple components at once

#### Questions/Answers

**Q: What is the full import path of `$GOPATH/src/github.com/liggitt/b`?**

A: The relative path under `$GOPATH/src`, so `github.com/liggitt/b`

**Q: What version of `github.com/liggitt/b` does `github.com/liggitt/a` use?**

A: Whatever is sitting in `$GOPATH/src/github.com/liggitt/b`

**Q: What version of `github.com/liggitt/b` does `github.com/liggitt/a` prefer to use?**

A: What's a version?

### Distribution

```sh
go get github.com/liggitt/a
```

#### What happens

* go resolves the version-control location for the specified import path
* go downloads the source to `$GOPATH/src/<import-path>`
* go also resolves and downloads transitive dependencies to `$GOPATH/src/<import-path>`

#### Benefits

* Simple distribution for simple things

#### Potential problems

* No versioning, you always get master
* No versioning of dependencies, you always get master of those as well
* Import path is coupled to location
* Random things get dumped into `$GOPATH`

## go modules: the new way

Note: if you want to work through this demo yourself, all the commands should work as described.
If you want something prepared in advance, the results of walking through this exercise are at
https://github.com/liggitt/a/tree/demo

### Defining a module

Instead of being defined by a path relative to `$GOPATH`, modules are just a tree
of Go source files with a `go.mod` file in the tree's root directory.
The tree can be located anywhere.

Let's remove our projects from `$GOPATH`:

```sh
rm -fr $GOPATH/src/github.com/liggitt/{a,b}
```

And try to build them outside our `$GOPATH`:

```sh
mkdir -p $HOME/tmp/modules/can/be/anywhere
cd $HOME/tmp/modules/can/be/anywhere
git clone https://github.com/liggitt/a.git
cd a
go run .
```

We see the problems we expect:
> ```
> make-cake.go:4:2: cannot find package "github.com/liggitt/a/helpers" in any of:
> 	/Users/liggitt/.gvm/gos/go1.12.1/src/github.com/liggitt/a/helpers (from $GOROOT)
> 	/Users/liggitt/go/src/github.com/liggitt/a/helpers (from $GOPATH)
> make-cake.go:5:2: cannot find package "github.com/liggitt/b" in any of:
> 	/Users/liggitt/.gvm/gos/go1.12.1/src/github.com/liggitt/b (from $GOROOT)
> 	/Users/liggitt/go/src/github.com/liggitt/b (from $GOPATH)
> ```

Without a relative path to `$GOPATH`, go has no way of knowing our `helpers` subpackage is `github.com/liggitt/a/helpers`.
It also doesn't have a way to find `github.com/liggitt/b`.

Let's turn our package into a go module:
```sh
go mod init github.com/liggitt/a
```

> ```
> go: creating new go.mod: module github.com/liggitt/a
> ```

```sh
cat go.mod
```

> ```
> module github.com/liggitt/a
>
> go 1.12
> ```

Commit the initial version of our go.mod file:
```sh
git add . && git commit -m "initial go.mod file"
```

#### What happens

From `go help go.mod`:
* The `module` verb defines the module path
* The `go` verb sets the expected language version

Now when we run, go can figure out our `helpers` package is `github.com/liggitt/a/helpers`
by finding the closest parent dir containing a `go.mod` file, looking at the name of the module it defines,
then appending the relative path to the `helpers` directory to get the full import path.

#### Benefits

* Allows developing outside of `$GOPATH`

### Resolving dependencies automatically

Since we can no longer assume all go source is located under `$GOPATH`, how does go find source for dependencies?
`go run` (and `go build`, `go list`, etc) will now fetch dependencies automatically from their canonical locations
(the same place `go get` would fetch them) at run time, if needed:

```sh
go run .
```

> ```sh
> go: finding github.com/liggitt/b v1.0.0
> go: downloading github.com/liggitt/b v1.0.0
> go: extracting github.com/liggitt/b v1.0.0
> cake!
> ```

#### What happens

That did a few things:

1. It noticed a dependency that wasn't represented in our `go.mod` file, so it resolved and added a `require` directive for it (`v1.0.0` happened to be the latest version):

   ```sh
   git diff go.mod
   ```

   > ```diff
   > diff --git a/go.mod b/go.mod
   > index 34c6e02..9869e5e 100644
   > --- a/go.mod
   > +++ b/go.mod
   > @@ -1,3 +1,5 @@
   >  module github.com/liggitt/a
   >  
   >  go 1.12
   > +
   > +require github.com/liggitt/b v1.0.0
   > ```
   
   From `go help go.mod`:
   * The `require` verb requires a particular module at a given version or later (editors note: the "or later" will be important)
   
   The syntax of a `require` directive is `require <module> <version>`.
   
   You can also group multiple `require` directives into a block, just like imports:
   ```
   require (
   	example.com/thing1 v2.3.4
   	example.com/thing2 v1.2.3
   )
   ```
   
   You can specify any resolveable tag, branch name, or SHA as a `require` version,
   and it will be canonicalized the next time the module graph is computed,
   and the `go.mod` file automatically updated.

   You can also run `go get <module>@<version>`, and go will resolve and add a 
   `require` directive for the specified module and version to your `go.mod` file:

   ```sh
   go get github.com/liggitt/b@v1.0.0
   ```

   Commit the updated go.mod file:
   ```sh
   git add . && git commit -m "require github.com/liggitt/b@v1.0.0"
   ```

2. `go run` also downloaded the new dependency to a local module cache:

   ```sh
   find $GOPATH/pkg/mod/cache/download
   ```
   
   > ```sh
   > /Users/liggitt/go/pkg/mod/cache/download
   > /Users/liggitt/go/pkg/mod/cache/download/github.com
   > /Users/liggitt/go/pkg/mod/cache/download/github.com/liggitt
   > /Users/liggitt/go/pkg/mod/cache/download/github.com/liggitt/b
   > /Users/liggitt/go/pkg/mod/cache/download/github.com/liggitt/b/@v
   > /Users/liggitt/go/pkg/mod/cache/download/github.com/liggitt/b/@v/v1.0.0.mod
   > /Users/liggitt/go/pkg/mod/cache/download/github.com/liggitt/b/@v/list.lock
   > /Users/liggitt/go/pkg/mod/cache/download/github.com/liggitt/b/@v/v1.0.0.zip
   > /Users/liggitt/go/pkg/mod/cache/download/github.com/liggitt/b/@v/v1.0.0.lock
   > /Users/liggitt/go/pkg/mod/cache/download/github.com/liggitt/b/@v/v1.0.0.ziphash
   > /Users/liggitt/go/pkg/mod/cache/download/github.com/liggitt/b/@v/v1.0.0.info
   > /Users/liggitt/go/pkg/mod/cache/download/github.com/liggitt/b/@v/list
   > ```

   This cache is maintained automatically, but has a few associated commands:
   
   To clean the module cache:
   ```sh
   go clean -modcache
   ```

   To force downloading dependencies:
   ```sh
   go mod download
   ```
  
   You can prevent downloading modules from the network by setting the environment variable `GOPROXY` to `off`.
   If dependencies are not present in the module cache, and network requests are disabled, operations will fail:
   ```sh
   go run .
   ```
   
   > ```sh
   > cake!
   > ```
   
   ```sh
   GOPROXY=off go run .
   ```
   
   > ```sh
   > cake!
   > ```
   
   ```sh
   go clean -modcache
   GOPROXY=off go run .
   ```
   
   > ```sh
   > go: github.com/liggitt/b@v1.0.0: module lookup disabled by GOPROXY=off
   > go: error loading module requirements
   > ```

   You can also choose to copy all required dependency packages into a local vendor directory:
   ```sh
   go mod vendor
   find vendor
   ```

   > ```sh
   > vendor
   > vendor/github.com
   > vendor/github.com/liggitt
   > vendor/github.com/liggitt/b
   > vendor/github.com/liggitt/b/cake.go
   > vendor/modules.txt
   > ```

   Then run with `-mod=vendor` to tell go to use the local vendor directory when resolving dependency source:

   ```sh
   go clean -modcache
   GOPROXY=off GOFLAGS=-mod=vendor go run .
   ```
   
   > ```sh
   > cake!
   > ```

   Clean up the vendor directory before continuing:
   ```sh
   rm -fr vendor
   ```

#### Benefits

* Dependencies don't drift from canonical source (the module cache computes checksums, which it compares with checksums in the optional `go.sum` file of the main module, and complains about differences)
* There is (some) control over what versions of dependencies are used
* There are built-in tools for caching and vendoring dependencies, and ensuring a hermetic build with no network access

### Distribution

Someone who wanted to fetch a particular version of your module could do this:
```sh
go get github.com/liggitt/a@v1.0.0
```

#### What happens

* go resolves the version-control location for the specified import path, and resolves the specified version
* go downloads the dependency to the module cache
* go adds a `require` directive to the current module's `go.mod` file recording the specified version

#### Benefits

* Simple distribution for simple things
* `go get` is version-aware (both for the requested module, and for its transitive dependencies)
* No stomping of versions in a single `$GOPATH` location when developing multiple modules

### Replacing resolved module versions or source

Let's see what happens when multiple modules in a build require different versions of the same module.

First, let's add a new dependency:

```sh
go get github.com/liggitt/c@v1.0.0
```

> ```diff
> git diff go.mod
> diff --git a/go.mod b/go.mod
> index 9869e5e..16d2163 100644
> --- a/go.mod
> +++ b/go.mod
> @@ -2,4 +2,7 @@ module github.com/liggitt/a
>  
>  go 1.12
>  
> -require github.com/liggitt/b v1.0.0
> +require (
> +       github.com/liggitt/b v1.0.0
> +       github.com/liggitt/c v1.0.0 // indirect
> +)
> ```

The `// indirect` decoration is added because we are not actually using any packages from the module yet.

Now change our `github.com/liggitt/a/make-cake.go` file to use the new module:

```diff
 package main
 
 import (
+       "fmt"
        "github.com/liggitt/a/helpers"
        "github.com/liggitt/b"
+       "github.com/liggitt/c"
 )
 
 func main() {
        b.Cake()
+       fmt.Println(c.CakeOrDeath())
        helpers.PrintSuccess()
 }
```

```sh
go run .
```

> ```sh
> cake, please
> cake!
> ```

Since we are now using a package from the module, the `// indirect`
decoration was removed from `go.mod` when `go run` computed the module graph.

Commit the changes:
```sh
git add . && git commit -m "require github.com/liggitt/c@v1.0.0"
```

Now add and use another dependency in `github.com/liggitt/a/make-cake.go`:

```diff
@@ -5,10 +5,12 @@ import (
        "github.com/liggitt/a/helpers"
        "github.com/liggitt/b"
        "github.com/liggitt/c"
+       "github.com/liggitt/d"
 )
 
 func main() {
        b.Cake()
        fmt.Println(c.CakeOrDeath())
+       fmt.Println(d.Ingredients())
        helpers.PrintSuccess()
 }
```

```sh
go get github.com/liggitt/d@v1.0.0
go run .
```

> ```
> sorry, all out of cake... your choice is 'or death'
> [eggs flour sugar butter]
> cake!
> ```

Wait... why did the output from `github.com/liggitt/c#CakeOrDeath()` change?

```sh
git diff go.mod
```

> ```diff
> diff --git a/go.mod b/go.mod
> index c6e8ae8..7e155cf 100644
> --- a/go.mod
> +++ b/go.mod
> @@ -4,5 +4,6 @@ go 1.12
>  
>  require (
>         github.com/liggitt/b v1.0.0
> -       github.com/liggitt/c v1.0.0
> +       github.com/liggitt/c v1.1.0
> +       github.com/liggitt/d v1.0.0
>  )
> ```

Our version of `github.com/liggitt/c` changed from `v1.0.0` to `v1.1.0`.
To see why, we can inspect our module dependency graph by running `go mod graph`:

```sh
go mod graph
```

> ```
> github.com/liggitt/a github.com/liggitt/b@v1.0.0
> github.com/liggitt/a github.com/liggitt/c@v1.1.0
> github.com/liggitt/a github.com/liggitt/d@v1.0.0
> github.com/liggitt/d@v1.0.0 github.com/liggitt/c@v1.1.0
> ```

Here we see that `github.com/liggitt/d@v1.0.0` requires `github.com/liggitt/c@v1.1.0`.

If we go look at [that module definition](https://github.com/liggitt/d/blob/v1.0.0/go.mod), that version is indeed required:

```
module github.com/liggitt/d

go 1.12

require github.com/liggitt/c v1.1.0
```

If multiple modules are involved in a build, and they require different versions
of the same module, the maximum required version of the module is selected,
and the `go.mod` file of the main module is updated to reflect that version.

From `go help go.mod`:
> The go command automatically updates go.mod each time it uses the
> module graph, to make sure go.mod always accurately reflects reality
> and is properly formatted.
> 
> The update removes redundant or misleading requirements.
> For example, if A v1.0.0 itself requires B v1.2.0 and C v1.0.0,
> then go.mod's requirement of B v1.0.0 is misleading (superseded by
> A's need for v1.2.0), and its requirement of C v1.0.0 is redundant
> (implied by A's need for the same version), so both will be removed.
> If module M contains packages that directly import packages from B or
> C, then the requirements will be kept but updated to the actual
> versions being used.
> 
> Because the module graph defines the meaning of import statements, any
> commands that load packages also use and therefore update go.mod,
> including go build, go get, go install, go list, go test, go mod graph,
> go mod tidy, and go mod why.

This auto-updating of the go.mod file to reflect reality is alternately helpful and maddening.
See https://github.com/golang/go/issues/29452 for discussion about this.

Whatever the reason, we really want to use `github.com/liggitt/c@v1.0.0` when running our build.
Fortunately, go modules allow the main module (the one where the go commands are run) to override
the selected version of a module. This is done with a `replace` directive in the `go.mod` file.

```sh
go mod edit -replace github.com/liggitt/c=github.com/liggitt/c@v1.0.0
go run .
```

> ```
> cake, please
> [eggs flour sugar butter]
> cake!
> ```

We're back to the behavior we wanted, let's see what that added to our `go.mod` file:

```sh
git diff go.mod
```

> ```diff
> diff --git a/go.mod b/go.mod
> index c6e8ae8..8aae820 100644
> --- a/go.mod
> +++ b/go.mod
> @@ -4,5 +4,8 @@ go 1.12
>  
>  require (
>         github.com/liggitt/b v1.0.0
> -       github.com/liggitt/c v1.0.0
> +       github.com/liggitt/c v1.1.0
> +       github.com/liggitt/d v1.0.0
>  )
> +
> +replace github.com/liggitt/c => github.com/liggitt/c v1.0.0
> ```

The maximum version still appears as a `require` directive,
but we are now forcing the version we want to be selected with a `replace` directive.

Commit the changes:
```sh
git add . && git commit -m "require github.com/liggitt/d v1.0.0, pin github.com/liggitt/c v1.0.0"
```

`replace` directives have many different uses, not just pinning versions. From `go help go.mod`:
* The `replace` verb replaces a module version with a different module version

Possible applications:

* pin a dependency to a specific version:

   ```sh
   go mod edit -replace example.com/some/thing=example.com/some/thing@v1.0.0
   ```
   produces this in `go.mod`:
   ```
   replace example.com/some/thing => example.com/some/thing v1.0.0
   ```

* change the remote source for a dependency (useful for building/testing with forks):

   ```sh
   go mod edit -replace example.com/some/thing@v1.0.0=example.com/my/fork@v1.0.0
   ```
   produces this in `go.mod`:
   ```
   replace example.com/some/thing v1.0.0 => example.com/my/fork v1.0.0
   ```

* use a local source for a dependency (useful for developing multiple modules locally):

   ```sh
   go mod edit -replace example.com/some/thing=../local/path/to/source
   ```
   produces this in `go.mod`:
   ```
   replace some/thing => ../local/path/to/source
   ```

#### Caveats

* `replace` directives apply only in the main module's `go.mod` and are ignored in dependencies,
so they are not effective in modules intended to be used as libraries by other modules.
* Modules you depend on via local path `replace` directives must also be published at their canonical
locations in order for components that use your module to be able to resolve them
* Changing the remote source for a dependency (for example, to point to a fork),
must point to a drop-in-compatible location. No import rewriting is performed.

### Major versions

What happens if a breaking change is made to a struct type, interface, or method signature,
and two modules in the same build depend on incompatible versions?

go modules allow module publishers to provide different major versions of a module,
by adding a major version suffix to the module import path. This results in a completely 
new import tree, which can be used alongside the old tree.
This is called [semantic import versioning](https://github.com/golang/go/wiki/Modules#semantic-import-versioning).

Let's make a [breaking change](https://github.com/liggitt/c/commit/6bfb5f89f40e7e6b10ce18771af841bf682044eb#diff-a6413ab6421e2f42b61888f9ac5b2db1) to `github.com/liggitt/c`:

```diff
-func CakeOrDeath() string {
+func CakeOrDeath(preference string) string {
+       if preference == "death" {
+               return "death it is"
+       }
        return "sorry, all out of cake... your choice is 'or death'"
 }
```

We also change the module name in `go.mod` file to append a major version suffix:

```diff
-module github.com/liggitt/c
+module github.com/liggitt/c/v2
```

Now we can tag that as [`v2.0.0`](https://github.com/liggitt/c/releases/tag/v2.0.0).

Versions prior to v2.0.0 are special-cased by go modules and don't need a major version suffix,
so callers that want to use v0.x or v1.x don't need to do anything special.

Callers that want to use v2.0.0+ of a module that uses semantic import versioning
have to [rewrite their go imports and change the required version of the module](https://github.com/liggitt/d/commit/c3bdd9d5b06e8387886616e30753140afdd39184) to add the versioned suffix:

```diff
diff --git a/go.mod b/go.mod
index 7d67056..fa289cd 100644
--- a/go.mod
+++ b/go.mod
@@ -2,4 +2,4 @@ module github.com/liggitt/d
 
 go 1.12
 
-require github.com/liggitt/c v1.1.0
+require github.com/liggitt/c/v2 v2.0.0
diff --git a/ingredients.go b/ingredients.go
index 1053743..c2bcfd5 100644
--- a/ingredients.go
+++ b/ingredients.go
@@ -3,7 +3,7 @@ package d
 import (
 	"strings"
 
-	"github.com/liggitt/c"
+	c "github.com/liggitt/c/v2"
 )
 
 func Ingredients() []string {
@@ -11,5 +11,5 @@ func Ingredients() []string {
 }
 
 func CakeAvailable() bool {
-	return !strings.Contains(c.CakeOrDeath(), "death")
+	return !strings.Contains(c.CakeOrDeath("cake"), "death")
 }
```

Because `github.com/liggitt/c` made that incompatible change in what is effectively a different package
(`github.com/liggitt/c/v2`), we can upgrade to a version of `github.com/liggitt/d` that uses that new version,
and our existing use of `github.com/liggitt/c` is unaffected:

```sh
go get github.com/liggitt/d@v1.1.0
go run .
```

> ```
> cake, please
> [eggs flour sugar butter]
> cake!
> ```

We can see both versions are in use:

```sh
go mod graph
```

> ```
> github.com/liggitt/a github.com/liggitt/b@v1.0.0
> github.com/liggitt/a github.com/liggitt/c@v1.1.0
> github.com/liggitt/a github.com/liggitt/d@v1.1.0
> github.com/liggitt/d@v1.1.0 github.com/liggitt/c/v2@v2.0.0
> ```

Commit the updated go.mod file:
```sh
git add . && git commit -m "require github.com/liggitt/d v1.1.0"
```

As we transition through multiple versions of dependencies, the checksum file (`go.sum`)
can accumulate checksums for dependencies we no longer use. To prune unused `require` directives
and unused checksums, running `go mod tidy` is recommended before publishing a module:

```sh
go mod tidy
git diff
```

```diff
diff --git a/go.sum b/go.sum
index fa373b9..8262cc2 100644
--- a/go.sum
+++ b/go.sum
@@ -2,11 +2,7 @@ github.com/liggitt/b v1.0.0 h1:b2PD0maWheor82stUPtMmVt15kSIal//bKvvXGW6m/c=
 github.com/liggitt/b v1.0.0/go.mod h1:ELIy9WS4GN+KTnXJ4EeReLowuaddQ7tnfsXV6BxuPiw=
 github.com/liggitt/c v1.0.0 h1:jLi0imaDl5DUIPX63+zI6vB5w2UF6wDa8F7bLMBDYko=
 github.com/liggitt/c v1.0.0/go.mod h1:F/PN0EPwqEcaXbrAc1E8sqvkXqQjvWH2whyWJoSkgA0=
-github.com/liggitt/c v1.1.0 h1:Uc7ExZIjGtvzmUjpGvpaSEWG2xHEQ/mJbK7sScoCYEQ=
-github.com/liggitt/c v1.1.0/go.mod h1:F/PN0EPwqEcaXbrAc1E8sqvkXqQjvWH2whyWJoSkgA0=
 github.com/liggitt/c/v2 v2.0.0 h1:7ITR3NLf81hJyIaFoUPUUl1QrFm7lnOjoH2v+fqsZf0=
 github.com/liggitt/c/v2 v2.0.0/go.mod h1:KRLLWBp7oLaMdqk8sLfRgalmCgDMNvQ4TQH5UrZevtI=
-github.com/liggitt/d v1.0.0 h1:7CL2m7s17MyF3C2R2Y7jCMt8Czp/Ats2DClddZoN+88=
-github.com/liggitt/d v1.0.0/go.mod h1:4GgUKOe83Va2bM/CVugCwU5aIM0V9Y5NpL1bg1rKZGE=
 github.com/liggitt/d v1.1.0 h1:MlVbFLjh4c5OZtN94h6Gj8x6BSpqXKi4epUIgSUdnFE=
 github.com/liggitt/d v1.1.0/go.mod h1:kLQGcLlWWIApPBG+JqB8mWJamzof4jHZ2nIhzfaullk=
```

The checksums for the unused versions of `github.com/liggitt/c` and `github.com/liggitt/d` are removed.

Commit the updated go.mod file:
```sh
git add . && git commit -m "go mod tidy"
```

#### Benefits of semantic import versioning

* Allows side-by-side use of different major versions in the same build
* Allows a module to make use of other major versions of itself if it wants to (`github.com/liggitt/c/v2` could use `github.com/liggitt/c`)

#### Caveats of semantic import versioning

* It's not magic... it behaves like an unrelated import path, so all callers have to change their imports to pick up the new version
* Even internal imports in the versioned module have to use the version-qualified import paths
* Deeply nested modules with lots of internal imports are painful to increment

### Tips

#### Getting started

Opt in to using go modules:
```sh
export GO111MODULE=on
```

Create a new module:
```
go mod init <name>
```

#### Querying

New commands (or module-aware versions of existing commands) for extracting module and dependency info:
```
go mod graph
go mod why <import-path>
go list -m all
go list -m -json all
go mod edit -json
```

Be aware that all of these (except `go mod edit -json`) have the potential to 
modify the `go.mod` file in the process of computing the module graph.

#### Cleaning up

Tidy a go.mod file (resolve references, remove unused replace directives, sort, etc),
and populate the `go.sum` file with checksums for the selected module's versions:
```sh
go mod tidy
```

#### Low-level go.mod manipulation

These commands all perform direct manipulation of the go.mod file.
They do not resolve the module graph, so they do not consult the 
network or module cache, or modify required versions.
All the invocations which modify the `go.mod` file also format it (`-fmt` is implied).

Dump to JSON:
```sh
go mod edit -json
```

Add a require directive:
```sh
go mod edit -require github.com/liggitt/a@v1.0.0
```

Remove a require directive:
```sh
go mod edit -droprequire github.com/liggitt/a
```

Add a replace directive to pin versions:
```sh
go mod edit -replace github.com/liggitt/c=github.com/liggitt/c@v1.0.0
```

Add a replace directive to use a forked version:
```sh
go mod edit -replace github.com/liggitt/c=github.com/example/c-fork@v1.0.0
```

Add a replace directive to use a local path:
```sh
go mod edit -replace github.com/liggitt/c=../path/to/source
```

Remove a replace directive:
```sh
go mod edit -dropreplace github.com/liggitt/c
```

Format the `go.mod` file:
```sh
go mod edit -fmt
```

See `go help mod edit` for details

#### Local module cache

Ensure the local module cache has all required dependencies:
```sh
go mod download
```

Clear the local module cache:
```sh
go clean -modcache
```

#### Module sources

By default, missing modules are searched/fetched using the network.
This can happen in response to go commands that were previously local-only,
like `go list`.

To prevent go operations from hitting the network (and fail if they need to):
```sh
export GOPROXY=off
```

To use a directory instead of the network, do `GOPROXY=file:///...`.
The module cache maintained by go is in the expected structure to be used this way:
```sh
export GOPROXY=file://$GOPATH/pkg/mod/cache/download
```

You can also point GOPROXY at a network location of a module proxy.

See `go help goproxy` for details

### Additional Reading

* `go help` topics:

   ```sh
   go help modules
   go help go.mod
   go help mod
   go help module-get
   go help goproxy
   ```

* https://github.com/golang/go/wiki/Modules, especially:
  * https://github.com/golang/go/wiki/Modules#semantic-import-versioning
  * https://github.com/golang/go/wiki/Modules#how-to-prepare-for-a-release
* https://golang.org/cmd/go/#hdr-The_go_mod_file
* https://golang.org/cmd/go/#hdr-Maintaining_module_requirements
* https://golang.org/cmd/go/#hdr-Module_compatibility_and_semantic_versioning
* https://blog.golang.org/using-go-modules
* https://blog.golang.org/modules2019
* [Open golang issues related to modules](https://github.com/golang/go/issues?q=is%3Aopen+is%3Aissue+label%3Amodules)
