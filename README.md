# <img src="http://goa.design/img/goa-logo.svg">

# A fork of Goa v1

This is unofficial fork of Goa v1.
If you want to use the latest version of Goa, go to [goadesign/goa](https://github.com/goadesign/goa).

goa is a framework for building micro-services and REST APIs in Go using a
unique design-first approach.

[![Build Status](https://github.com/shogo82148/goa-v1/workflows/test/badge.svg?branch=master)](https://github.com/shogo82148/goa-v1/actions?query=workflow%3Atest)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](https://github.com/shogo82148/goa-v1/blob/master/LICENSE)
[![Go Reference](https://pkg.go.dev/badge/github.com/shogo82148/goa-v1.svg)](https://pkg.go.dev/github.com/shogo82148/goa-v1)

## Why Goa?

Goa takes a different approach to building micro-services. Instead of focusing
solely on helping with implementation, Goa makes it possible to describe the
*design* of an API using a simple DSL. Goa then uses that description to provide
specialized helper code to the implementation and to generate documentation, API
clients, tests, even custom artifacts.

If DSLs are not your thing then consider this: you need to document your APIs so
that clients (be it internal e.g. other services or external e.g. UIs) may
consume them. Typically this requires maintaining a completely separate document
(for example an OpenAPI specification). Making sure that the document stays
up-to-date takes a lot of effort and at the end of the day you have to write
that document - why not use a simple and clear Go DSL to do that instead?

Another aspect to consider is the need for properly designing APIs and making
sure that the design choices remain consistent across the endpoints or even
across multiple APIs. If the source code is the only place where design
decisions are kept then not only is it impossible to maintain consistency it's
also difficult to think about the design in the first place. The Goa DSL makes
it possible to think about the design explicitly and - since it's code - to
re-use design elements for consistency.

The Goa DSL allows writing self-explanatory code that describes the resources
exposed by the API and for each resource the properties and actions. Goa comes
with the `goagen` tool which runs the DSL and generates various types of
artifacts from the resulting data structures.

One of the `goagen` output is glue code that binds your code with the underlying
HTTP server. This code is specific to your API so that for example there is no
need to cast or "bind" any handler argument prior to using them. Each generated
handler has a signature that is specific to the corresponding resource action.
It's not just the parameters though, each handler also has access to specific
helper methods that generate the possible responses for that action. The DSL can
also define validations in which case the generated code takes care of
validating the incoming request parameters and payload prior to invoking the
handler.

The end result is controller code that is terse and clean, the boilerplate is
all gone. Another big benefit is the clean separation of concern between design
and implementation: on bigger projects it's often the case that API design
changes require careful review, being able to generate a new version of the
documentation without having to write a single line of implementation is a big
boon.

This idea of separating design and implementation is not new, the
excellent [Praxis](http://praxis-framework.io) framework from RightScale follows
the same pattern and was an inspiration to Goa.

## Installation

Goa v1 can be used with Go modules:

```bash
export GO111MODULE=on
go mod init <my project>
go get github.com/shogo82148/goa-v1/...@v1
```

Or without Go modules by cloning the repo first:

```bash
cd $GOPATH/src
mkdir -p github.com/goadesign
cd github.com/goadesign
git clone https://github.com/shogo82148/goa-v1
cd goa; git checkout v1
go get -v github.com/shogo82148/goa-v1/...
```

### Stable Versions

Goa follows [Semantic Versioning](http://semver.org/) which is a fancy way of saying it publishes
releases with version numbers of the form `vX.Y.Z` and makes sure that your code can upgrade to new
versions with the same `X` component without having to make changes.

Releases are tagged with the corresponding version number. There is also a branch for each major
version (only `v1` at the moment). The recommended practice is to vendor the stable branch.

Current Release: `v1.5.11`
Stable Branch: `v1`

## Teaser

### 1. Design

Create the file `$GOPATH/src/goa-adder/design/design.go` with the following content:
```go
package design

import (
        . "github.com/shogo82148/goa-v1/design"
        . "github.com/shogo82148/goa-v1/design/apidsl"
)

var _ = API("adder", func() {
        Title("The adder API")
        Description("A teaser for goa")
        Host("localhost:8080")
        Scheme("http")
})

var _ = Resource("operands", func() {
        Action("add", func() {
                Routing(GET("add/:left/:right"))
                Description("add returns the sum of the left and right parameters in the response body")
                Params(func() {
                        Param("left", Integer, "Left operand")
                        Param("right", Integer, "Right operand")
                })
                Response(OK, "text/plain")
        })

})
```
This file contains the design for an `adder` API which accepts HTTP GET requests to `/add/:x/:y`
where `:x` and `:y` are placeholders for integer values. The API returns the sum of `x` and `y` in
its body.

### 2. Implement

Now that the design is done, let's run `goagen` on the design package:
```
cd $GOPATH/src/goa-adder
goagen bootstrap -d goa-adder/design
```
This produces the following outputs:

* `main.go` and `operands.go` contain scaffolding code to help bootstrap the implementation.
  running `goagen` again does not recreate them so that it's safe to edit their content.
* an `app` package which contains glue code that binds the low level HTTP server to your
  implementation.
* a `client` package with a `Client` struct that implements a `AddOperands` function which calls
  the API with the given arguments and returns the `http.Response`.
* a `tool` directory that contains the complete source for a client CLI tool.
* a `swagger` package with implements the `GET /swagger.json` API endpoint. The response contains
  the full Swagger specificiation of the API.

### 3. Run

First let's implement the API - edit the file `operands.go` and replace the content of the `Add`
function with:
```
// Add import for strconv
import "strconv"

// Add runs the add action.
func (c *OperandsController) Add(ctx *app.AddOperandsContext) error {
        sum := ctx.Left + ctx.Right
        return ctx.OK([]byte(strconv.Itoa(sum)))
}
```
Now let's compile and run the service:
```
cd $GOPATH/src/goa-adder
go build
./goa-adder
2016/04/05 20:39:10 [INFO] mount ctrl=Operands action=Add route=GET /add/:left/:right
2016/04/05 20:39:10 [INFO] listen transport=http addr=:8080
```
Open a new console and compile the generated CLI tool:
```
cd $GOPATH/src/goa-adder/tool/adder-cli
go build
```
The tool includes contextual help:
```
./adder-cli --help
CLI client for the adder service

Usage:
  adder-cli [command]

Available Commands:
  add         add returns the sum of the left and right parameters in the response body

Flags:
      --dump               Dump HTTP request and response.
  -H, --host string        API hostname (default "localhost:8080")
  -s, --scheme string      Set the requests scheme
  -t, --timeout duration   Set the request timeout (default 20s)

Use "adder-cli [command] --help" for more information about a command.
```
To get information on how to call a specific API use:
```
./adder-cli add operands --help
Usage:
  adder-cli add operands [/add/LEFT/RIGHT] [flags]

Flags:
      --left int    Left operand
      --pp          Pretty print response body
      --right int   Right operand

Global Flags:
      --dump               Dump HTTP request and response.
  -H, --host string        API hostname (default "localhost:8080")
  -s, --scheme string      Set the requests scheme
  -t, --timeout duration   Set the request timeout (default 20s)
```
Now let's run it:
```
./adder-cli add operands /add/1/2
2016/04/05 20:43:18 [INFO] started id=HffVaGiH GET=http://localhost:8080/add/1/2
2016/04/05 20:43:18 [INFO] completed id=HffVaGiH status=200 time=1.028827ms
3⏎
```
This also works:
```
$ ./adder-cli add operands --left=1 --right=2
2016/04/25 00:08:59 [INFO] started id=ouKmwdWp GET=http://localhost:8080/add/1/2
2016/04/25 00:08:59 [INFO] completed id=ouKmwdWp status=200 time=1.097749ms
3⏎
```
The console running the service shows the request that was just handled:
```
2016/06/06 10:23:03 [INFO] started req_id=rLAtsSThLD-1 GET=/add/1/2 from=::1 ctrl=OperandsController action=Add
2016/06/06 10:23:03 [INFO] params req_id=rLAtsSThLD-1 right=2 left=1
2016/06/06 10:23:03 [INFO] completed req_id=rLAtsSThLD-1 status=200 bytes=1 time=66.25µs
```
Now let's see how robust our service is and try to use non integer values:
```
./adder-cli add operands add/1/d
2016/06/06 10:24:22 [INFO] started id=Q2u/lPUc GET=http://localhost:8080/add/1/d
2016/06/06 10:24:22 [INFO] completed id=Q2u/lPUc status=400 time=1.301083ms
error: 400: {"code":"invalid_request","status":400,"detail":"invalid value \"d\" for parameter \"right\", must be a integer"}
```
As you can see the generated code validated the incoming request against the types defined in the
design.

### 4. Document

The `swagger` directory contains the API Swagger (OpenAPI) specification in both
YAML and JSON format.

For open source projects hosted on
github [swagger.goa.design](http://swagger.goa.design) provides a free service
that renders the Swagger representation dynamically from goa design packages.
Simply set the `url` query string with the import path to the design package.
For example displaying the docs for `github.com/shogo82148/goa-v1-cellar/design` is
done by browsing to:

http://swagger.goa.design/?url=goadesign%2Fgoa-cellar%2Fdesign

Note that the above generates the swagger spec dynamically and does not require it to be present in
the Github repo.

The Swagger JSON can also easily be served from the documented service itself using a simple
[Files](http://goa.design/reference/goa/design/apidsl/#func-files-a-name-apidsl-files-a)
definition in the design. Edit the file `design/design.go` and add:

```go
var _ = Resource("swagger", func() {
        Origin("*", func() {
               Methods("GET") // Allow all origins to retrieve the Swagger JSON (CORS)
        })
        Files("/swagger.json", "swagger/swagger.json")
})
```

Re-run `goagen bootstrap -d goa-adder/design` and note the new file
`swagger.go` containing the implementation for a controller that serves the
`swagger.json` file.

Mount the newly generated controller by adding the following two lines to the `main` function in
`main.go`:

```go
cs := NewSwaggerController(service)
app.MountSwaggerController(service, cs)
```

Recompile and restart the service:

```
^C
go build
./goa-adder
2016/06/06 10:31:14 [INFO] mount ctrl=Operands action=Add route=GET /add/:left/:right
2016/06/06 10:31:14 [INFO] mount ctrl=Swagger files=swagger/swagger.json route=GET /swagger.json
2016/06/06 10:31:14 [INFO] listen transport=http addr=:8080
```

Note the new route `/swagger.json`.  Requests made to it return the Swagger specification. The
generated controller also takes care of adding the proper CORS headers so that the JSON may be
retrieved from browsers using JavaScript served from a different origin (e.g. via Swagger UI). The
client also has a new `download` action:

```
cd tool/adder-cli
go build
./adder-cli download --help
Download file with given path

Usage:
  adder-cli download [PATH] [flags]

Flags:
      --out string   Output file

Global Flags:
      --dump               Dump HTTP request and response.
  -H, --host string        API hostname (default "localhost:8080")
  -s, --scheme string      Set the requests scheme
  -t, --timeout duration   Set the request timeout (default 20s)
```

Which can be used like this to download the file `swagger.json` in the current directory:

```
./adder-cli download swagger.json
2016/06/06 10:36:24 [INFO] started file=swagger.json id=ciHL2VLt GET=http://localhost:8080/swagger.json
2016/06/06 10:36:24 [INFO] completed file=swagger.json id=ciHL2VLt status=200 time=1.013307ms
```

We now have a self-documenting API and best of all the documentation is automatically updated as the
API design changes.

## Resources

Consult the following resources to learn more about Goa.

### goa.design

[goa.design](https://goa.design) contains further information on Goa including a getting
started guide, detailed DSL documentation as well as information on how to implement a Goa service.

### Examples

The [examples](https://github.com/goadesign/examples) repo contains simple examples illustrating
basic concepts.

The [goa-cellar](https://github.com/shogo82148/goa-v1-cellar) repo contains the implementation for a
Goa service which demonstrates many aspects of the design language. It is kept up-to-date and
provides a reference for testing functionality.

## Contributing

Did you fix a bug? write docs or additional tests? or implement some new awesome functionality?
You're a rock star!! Just make sure that `make` succeeds (or that TravisCI is green) and send a PR
over.
