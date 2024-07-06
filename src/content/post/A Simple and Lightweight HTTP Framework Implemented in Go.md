---
title: "A Simple and Lightweight HTTP Framework Implemented in Go"
description: "A Simple and Lightweight HTTP Framework Implemented in Go"
publishDate: "13 Jan 2023"
tags: ["webdev", "opensource", "programming", "go"]
---

## Forward

I had been using a lot Go HTTP frameworks like Hertz, Gin and Fiber, while reading through the source code of these frameworks I tried to implement a simple HTTP framework myself.

A few days later, [PIANO](https://github.com/B1NARY-GR0UP/piano) was born. I referenced code from Hertz and Gin and implemented most of the features that an HTTP framework should have.

For example, [PIANO](https://github.com/B1NARY-GR0UP/piano) supports parameter routing, wildcard routing, and static routing, supports route grouping and middleware, and multiple forms of parameter fetching and returning. I'll introduce these features next.

## Quick Start

### Install

```shell
go get github.com/B1NARY-GR0UP/piano
```

Or you can clone it and read the code directly.

### Hello PIANO

```go
package main

import (
	"context"
	"net/http"

	"github.com/B1NARY-GR0UP/piano/core"
	"github.com/B1NARY-GR0UP/piano/core/bin"
)

func main() {
	p := bin.Default()
	p.GET("/hello", func(ctx context.Context, pk *core.PianoKey) {
		pk.String(http.StatusOK, "piano")
	})
	p.Play()
}
```

As you can see, `PIANO`'s Hello World is very simple: we register a route with the `GET` function, and then all you have to do is visit `localhost:7246/hello` in your browser or some other tool to see the value returned (which is "piano").

`7246` is the default listening port I set for `PIANO`, you can change it to your preferred listening port with the `WithHostAddr` function.

## Features

### Route

- **Static Route**

  Static route is the simplest and most common form of routing, and the one we just demonstrated in Quick Start is static route. We set up a handler to handle a fixed HTTP request URL.

  ```go
  package main
  
  import (
  	"context"
  	"net/http"
  
  	"github.com/B1NARY-GR0UP/piano/core"
  	"github.com/B1NARY-GR0UP/piano/core/bin"
  )
  
  func main() {
  	p := bin.Default()
  	// static route or common route
  	p.GET("/ping", func(ctx context.Context, pk *core.PianoKey) {
  		pk.JSON(http.StatusOK, core.M{
  			"ping": "pong",
  		})
  	})
  	p.Play()
  }
  ```

- **Param Route**

  We can set a route in the form of `:yourparam` to match the HTTP request URL, and `PIANO` will automatically parse the request URL and store the parameters as key / value pairs, which we can then retrieve using the `Param` method. For example,  if we set a route like `/param/:username`, when we send a request with URL `localhost:7246/param/lorain`, the param `username` will be assigned with value `lorain`. And we can get `username` by `Param` function.

  ```go
  package main
  
  import (
  	"context"
  	"net/http"
  
  	"github.com/B1NARY-GR0UP/piano/core"
  	"github.com/B1NARY-GR0UP/piano/core/bin"
  )
  
  func main() {
  	p := bin.Default()
  	// param route
  	p.GET("/param/:username", func(ctx context.Context, pk *core.PianoKey) {
  		pk.JSON(http.StatusOK, core.M{
  			"username": pk.Param("username"),
  		})
  	})
  	p.Play()
  }
  ```

- **Wildcard Route**

  The wildcard route matches all routes and can be set in the form `*foobar`.

  ```go
  package main
  
  import (
  	"context"
  	"net/http"
  
  	"github.com/B1NARY-GR0UP/piano/core"
  	"github.com/B1NARY-GR0UP/piano/core/bin"
  )
  
  func main() {
  	p := bin.Default()
  	// wildcard route
  	p.GET("/wild/*", func(ctx context.Context, pk *core.PianoKey) {
  		pk.JSON(http.StatusOK, core.M{
  			"route": "wildcard route",
  		})
  	})
  	p.Play()
  }
  ```

### Route Group

PIANO also implements route group, where we can group routes with the same prefix to simplify our code.

```go
package main

import (
	"context"
	"net/http"

	"github.com/B1NARY-GR0UP/piano/core"
	"github.com/B1NARY-GR0UP/piano/core/bin"
)

func main() {
	p := bin.Default()
	auth := p.GROUP("/auth")
	auth.GET("/ping", func(ctx context.Context, pk *core.PianoKey) {
		pk.String(http.StatusOK, "pong")
	})
	auth.GET("/binary", func(ctx context.Context, pk *core.PianoKey) {
		pk.String(http.StatusOK, "lorain")
	})
	p.Play()
}
```

### Parameter Acquisition

The HTTP request URL or parameters in the request body can be retrieved using methods  `Query`, `PostForm`.

- **Query**

```go
package main

import (
	"context"
	"net/http"

	"github.com/B1NARY-GR0UP/piano/core"
	"github.com/B1NARY-GR0UP/piano/core/bin"
)

func main() {
	p := bin.Default()
	p.GET("/query", func(ctx context.Context, pk *core.PianoKey) {
		pk.JSON(http.StatusOK, core.M{
			"username": pk.Query("username"),
		})
	})
	p.Play()
}
```

- **PostForm**

```go
package main

import (
	"context"
	"net/http"

	"github.com/B1NARY-GR0UP/piano/core"
	"github.com/B1NARY-GR0UP/piano/core/bin"
)

func main() {
	p := bin.Default()
	p.POST("/form", func(ctx context.Context, pk *core.PianoKey) {
		pk.JSON(http.StatusOK, core.M{
			"username": pk.PostForm("username"),
			"password": pk.PostForm("password"),
		})
	})
	p.Play()
}
```

### Other

`PIANO` has many other small features such as storing information in the request context, returning a response as JSON or string, middleware support, so I won't cover them all here.

## Design

Here we will introduce some of the core design of `PIANO`.  `PIANO` is based entirely on the Golang standard library, with only one dependency for a simple log named [inquisitor](https://github.com/B1NARY-GR0UP/inquisitor) that I implemented myself.

- `go.mod`

```go
module github.com/B1NARY-GR0UP/piano

go 1.18

require github.com/B1NARY-GR0UP/inquisitor v0.1.0
```

### Context

The design of `PIANO` context is as follows:

```go
// PianoKey play the piano with PianoKeys
type PianoKey struct {
	Request *http.Request
	Writer  http.ResponseWriter

	index    int // initialize with -1
	Params   Params
	handlers HandlersChain
	rwMutex  sync.RWMutex
	KVs      M
}
```

And after referring to the context design of several frameworks, I decided to adopt Hertz's scheme, which separates the request context from the `context.Context`, this can be well used in tracing and other scenarios through the correct management of the two different lifecycle contexts.

```go
// HandlerFunc is the core type of PIANO
type HandlerFunc func(ctx context.Context, pk *PianoKey)
```

The current context only provides some simple functionality, such as storing key-value pairs in `KVs`, but more features will be supported later.

### Route Tree

The design of the route tree uses the trie tree data structure, which inserts and searches in the form of iteration.

- **Insert**

```go
// insert into trie tree
func (t *tree) insert(path string, handlers HandlersChain) {
	if t.root == nil {
		t.root = &node{
			kind:     root,
			fragment: strSlash,
		}
	}
	currNode := t.root
	fragments := splitPath(path)
	for i, fragment := range fragments {
		child := currNode.matchChild(fragment)
		if child == nil {
			child = &node{
				kind:     matchKind(fragment),
				fragment: fragment,
				parent:   currNode,
			}
			currNode.children = append(currNode.children, child)
		}
		if i == len(fragments)-1 {
			child.handlers = handlers
		}
		currNode = child
	}
}
```

- **Search**

```go
// search matched node in trie tree, return nil when no matched
func (t *tree) search(path string, params *Params) *node {
	fragments := splitPath(path)
	var matchedNode *node
	currNode := t.root
	for i, fragment := range fragments {
		child := currNode.matchChildWithParam(fragment, params)
		if child == nil {
			return nil
		}
		if i == len(fragments)-1 {
			matchedNode = child
		}
		currNode = child
	}
	return matchedNode
}
```

It is worth mentioning that the different routing support is also implemented here.

## Future Work

- **Encapsulation protocol layer**
- **Add more middleware support**
- **Improve engine and context functionality**
- ...

## Summary

That's all for this article. Hopefully, this has given you an idea of the `PIANO` framework and how you can implement a simple version of HTTP framework yourself. I would be appreciate if you could give [PIANO](https://github.com/B1NARY-GR0UP/piano) a star.

If you have any questions, please leave them in the comments or as issues. Thanks for reading.

## Reference List

- https://github.com/B1NARY-GR0UP/piano

- https://github.com/cloudwego/hertz
- https://github.com/gin-gonic/gin

