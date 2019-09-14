---
layout: post
title: "Go patterns for web services (by Mat Ryer's talk)"
date: 2019-09-14 18:55:39 +0200
categories: engineering
tags: [go, golang, programming]
---

Taken out of [Mat Ryer's talk](https://www.youtube.com/watch?v=rWBSMsLG8po) to have a brief overview here.

## Tiny `main` abstraction

The Go way of handling errors is to return error object as a last value. `main` function does not allow
that approach, but we could have a tiny wrapper around:

```go
func main() {
  if err := run(); err != nil {
    fmt.Fprintf(os.Stderr, "%s\n", err)
    os.Exit(1)
  }
}

func run() error {
  db, dbtidy, err := setupDatabase()
  if err !== nil {
    return errors.Wrap(err, "setup database")
  }
  defer dbtidy()
  // ...
}
```

## The `server` struct

A struct of `server` type represents the component and dependencies go into that type.
Makes it clear what server needs in order to do its job. Also, removes temptation to have these
in some kind of global state.

```go
type server struct {
  db *someDb
  router *someRouter
  email EmailSender
}
```

## Make `server` an `http.Handler`

```go
func (s *server) ServeHTTP (w http.ResponseWriter, r *http.Request) {
  s.router.ServeHttp(w, r)
}
```

This turns `server` into `http.Handler` and allows `server` to be used where `http.Handler` can be used (e.g. `http.ListenAndServe`).

Important: there should be no logic - just pass execution to the `router` so the flow is explicit.

## One place for all routes (routes.go)

```go
package main

func (s *server) routes() {
  s.router.Get("/api", s.serveApi())
  s.router.Get("/about", s.serveAbout())
  // ...
}
```

## Handlers hang off the `server`

Each handler is a method on `server`:

```go
func (s *server) handleSomething() http.HandlerFunc {
  // some logic here
}
```

- access to dependencies via \*s
- be careful with data races as other handlers also have access to \*s.

## Naming handler methods

Starts with `handle`, then subject, then action, e.g.:

- `handleTaskCreate`
- `handleAuthLogin`

## Returning the handler

Returning the handler rather than being a handler itself:

```go
func (s *server) handleSomething() http.HandlerFunc {
  thing := prepareThing()

  return func (rw http.ResponseWriter, r *http.Request) {
    // `thing` can be used here
  }
}
```

Allows to put some startup logic inside.

## Server struct gets too big? Split them.

```go
// people.go
type serverPeople struct {
  db *myDb
  emailSender EmailSender
}

// comments.go
type serverComments struct {
  db *myDb
}
```

## Middleware are methods on server

```go
func (s *server) adminOnly(h http.HandlerFunc) http.HandlerFunc {
  return func (rw http.ResponseWriter, r *http.Request) {
    if !currentUser(r).isAdmin {
      http.NotFound(rw, r)
      return
    }
    h(w, r)
  }
}
```

## Wire middleware in routes.go

```go
package main

func (s *server) routes() {
  s.router.Get("/api", s.serveApi())
  s.router.Get("/admin", s.adminOnly(s.serveAdminIndex()))
}
```

So `routes.go` is a high level map of the service.

## Setup handlers lazily with sync.Once

```go
func (s *server) handleSomething(file string...) http.HandlerFunc {
  var (
    init sync.Once
    tpl *template.Template
    tplerror error
  )

  return func (rw http.ResponseWriter, r *http.Request) {
    init.Do(func() {
      tpl, tplerror = template.ParseFiles(files...)
    })
    // ...
  }
}
```

## Testing

### Use `net/http/httptest`

### Our `server` struct is testable!

```go
func TestHandleAbout(t *testing.T) {
  is := is.New(t)
  db, cleanup := connectToTestDatabase()
  defer cleanup()
  srv := &server{
    db: db
  }
  r := httptest.NewRequest("GET", "/about", nil)
  rw := httptest.NewRecorder()
  srv.serveHTTP(r, rw)
  is.Equal(w.StatusCode, http.StatusOK)
}

```
