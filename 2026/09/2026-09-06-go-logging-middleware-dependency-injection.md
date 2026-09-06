# Go Logging, Request Middleware, and Dependency Injection

Today I learned more about logging in Go.

At first, logging looked like a small change:

```go
fmt.Println("server started")
```

becomes:

```go
log.Println("server started")
```

But the exercises gradually turned this into something much more interesting.

I went through several stages:

```text
fmt
 ↓
package-level log
 ↓
custom log.Logger
 ↓
request logging middleware
 ↓
dependency-injected loggers
 ↓
separate access and application logs
```

The biggest thing that clicked is that logging is not just about printing text.

It is part of the application's infrastructure.

Where logs are written, what metadata they contain, which component owns the logger, and how the logger gets into that component all affect how maintainable and observable the application is.

---

# `fmt` Is Not a Logger

I have used things like this many times:

```go
fmt.Println("starting server")
fmt.Printf("listening on port %d\n", port)
fmt.Fprintf(os.Stderr, "something failed: %v\n", err)
```

These functions are perfectly fine for normal program output.

But application logs are different from normal output.

A log usually needs additional behavior such as:

* timestamps
* prefixes
* configurable destinations
* consistent formatting
* separation from normal program output
* eventually, log levels or structured fields

Go's standard library already provides the `log` package for this.

For example:

```go
log.Println("starting server")
```

produces something similar to:

```text
2026/09/06 20:00:00 starting server
```

The timestamp is automatically added.

Similarly:

```go
log.Printf("listening on port %d", port)
```

does not need:

```go
\n
```

because the logger handles the newline itself.

So one small rule I want to remember is:

```text
fmt
→ normal program output / formatting

log
→ operational information about the program
```

This distinction matters especially for servers.

A CLI application might intentionally print something to `STDOUT` because that output is the result the user asked for.

A server log is mainly information about what the application itself is doing.

---

# `STDOUT` vs `STDERR`

Another thing that became clearer today is why logs are commonly written to `STDERR`.

Unix programs have separate output streams:

```text
STDOUT
→ normal program output

STDERR
→ errors, diagnostics, logs
```

Keeping them separate makes the program easier to compose with other tools.

For example, if a program's actual output is being piped somewhere:

```sh
myprogram | another-program
```

I usually do not want debug logs mixed into the data being sent through the pipe.

For application logging, `STDERR` is therefore usually a better default.

This also explained this command from the exercises:

```sh
go run . 2>&1 | sh -c 'trap "" INT; tee linko.out.log'
```

The important part is:

```sh
2>&1
```

File descriptor `1` is `STDOUT`.

File descriptor `2` is `STDERR`.

So:

```text
2>&1
```

means:

```text
send STDERR to the same place as STDOUT
```

Then:

```sh
tee linko.out.log
```

can display the combined output in the terminal while also saving it to a file.

Conceptually:

```text
Go program
   |
   ├── STDOUT ──┐
   |            |
   └── STDERR ──┤
                |
                v
               tee
              /   \
             v     v
         terminal  linko.out.log
```

---

# Moving from Package-Level `log` to `log.Logger`

The first improvement was replacing calls such as:

```go
log.Printf(...)
```

with a logger object.

A logger can be created with:

```go
logger := log.New(
    os.Stderr,
    "DEBUG: ",
    log.LstdFlags,
)
```

There are three important pieces here.

```text
os.Stderr
```

controls where logs are written.

```text
"DEBUG: "
```

is the prefix added to each message.

```text
log.LstdFlags
```

adds Go's standard date and time information.

So:

```go
logger.Printf("Linko is running")
```

might produce:

```text
DEBUG: 2026/09/06 20:00:00 Linko is running
```

This was my first glimpse of why a logger object is more useful than directly calling package-level functions.

Instead of configuring every logging call separately:

```text
component
   |
   ├── decide destination
   ├── decide prefix
   ├── decide timestamp
   └── print message
```

I configure the logger once:

```text
logger
├── output = STDERR
├── prefix = DEBUG:
└── flags = date + time

          |
          v

many logging calls
```

The logging calls only need to care about the message.

---

# Why `log.Logger` Is Better Than Scattered Logging

A `log.Logger` gives me one place to configure logging behavior.

For example, changing:

```go
log.New(os.Stderr, "DEBUG: ", log.LstdFlags)
```

to another destination changes all logging that uses that logger.

That means the application code does not need to know whether logs eventually go to:

```text
terminal
file
test buffer
external logging system
```

The component just receives something capable of logging.

That separation becomes especially useful in tests.

Instead of writing test logs into my actual terminal, I could create a logger backed by a buffer.

Conceptually:

```text
Production
component → logger → STDERR/file

Test
component → logger → memory buffer
```

Same component.

Different dependency.

---

# Logging HTTP Requests with Middleware

The next lesson connected logging with HTTP middleware.

For a web server, it is useful to know things like:

```text
GET /
GET /api/stats
POST /admin/shutdown
```

I could manually put logging code inside every handler:

```go
func handlerSomething(w http.ResponseWriter, r *http.Request) {
    logger.Printf(...)
    // handler logic
}
```

But that would repeat the same concern everywhere.

Logging a request is not really the responsibility of each individual handler.

Middleware gives me a cleaner place to put cross-cutting behavior like this.

The request logger looked roughly like:

```go
func requestLogger(logger *log.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            next.ServeHTTP(w, r)

            logger.Printf(
                "Served request: %s %s",
                r.Method,
                r.URL.Path,
            )
        })
    }
}
```

The important part is:

```go
next.ServeHTTP(w, r)
```

The middleware lets the actual handler process the request.

Then it logs:

```go
logger.Printf(
    "Served request: %s %s",
    r.Method,
    r.URL.Path,
)
```

For example:

```text
Served request: GET /
Served request: GET /api/stats
```

Even something like an unauthorized request can still be useful to record.

For example:

```text
GET /api/stats
→ 401 Unauthorized
```

The request was still served by the application, so it should still appear in the access logs.

---

# My Middleware Mental Model

Middleware makes more sense to me when I think of it as wrapping another handler.

Without middleware:

```text
Request
   |
   v
Handler
   |
   v
Response
```

With request logging:

```text
Request
   |
   v
Request Logger Middleware
   |
   v
Handler
   |
   v
Request Logger Middleware
   |
   ├── log method/path
   |
   v
Response
```

Or in code:

```text
requestLogger(logger)(
    mux
)
```

The logger does not replace the mux.

It wraps it.

This is useful because the same pattern can eventually be used for things like:

```text
authentication
request logging
metrics
panic recovery
tracing
CORS
rate limiting
```

Those behaviors can live around the handlers instead of being duplicated inside each handler.

---

# Global Logger

One intermediate step was creating a global logger:

```go
var logger = log.New(
    os.Stderr,
    "DEBUG: ",
    log.LstdFlags,
)
```

This definitely improved consistency compared with scattered `log.Printf` calls.

Every file could use:

```go
logger.Printf(...)
```

and get the same output configuration.

But there is still a problem:

```text
global logger
     |
     ├── server
     ├── handlers
     ├── auth
     └── store
```

Everything depends on shared global state.

That makes the dependency invisible.

If I look at a function like:

```go
func something() {
    logger.Printf(...)
}
```

the function signature does not tell me that it depends on a logger.

The dependency exists, but it is hidden somewhere outside the function.

That becomes especially annoying when testing.

---

# Dependency Injection

This introduced dependency injection.

The name sounds much more complicated than the idea actually is.

Dependency injection basically means:

> If something needs another thing, pass that dependency to it.

Instead of:

```go
var logger *log.Logger

func doSomething() {
    logger.Println("hello")
}
```

I can do something like:

```go
func doSomething(logger *log.Logger) {
    logger.Println("hello")
}
```

Now the dependency is explicit.

The function tells me:

```text
I need a logger in order to do my job.
```

For larger components, the logger can be stored on a struct.

For example:

```go
type server struct {
    logger *log.Logger
}
```

Then the constructor receives it:

```go
func newServer(logger *log.Logger) *server {
    return &server{
        logger: logger,
    }
}
```

And server methods use:

```go
s.logger.Printf(...)
```

The dependency path becomes visible:

```text
run()
  |
  | creates logger
  v
newServer(logger)
  |
  v
server
  |
  v
server methods
```

The server does not decide how logging should be configured.

The code that constructs the server decides.

That separation is important.

---

# Injecting a Logger into the Store

The same idea applied to the storage layer.

Instead of the store package reaching out to some global logger:

```text
Store
  |
  └── global logger
```

the logger becomes part of the store's dependencies:

```go
type Store struct {
    logger *log.Logger
}
```

and something like:

```go
func New(logger *log.Logger) *Store {
    return &Store{
        logger: logger,
    }
}
```

Then the dependency graph becomes:

```text
                 run()
                   |
          creates dependencies
             /           \
            v             v
       server logger   store logger
            |             |
            v             v
         server          Store
```

This is a much clearer architecture.

`run()` becomes the place where the application is assembled.

The components themselves mostly just receive what they need.

---

# Different Components Can Use Different Loggers

This was probably the most useful consequence of dependency injection.

Once the logger is no longer global, different parts of the application can receive different loggers.

The exercise created two.

An access logger:

```text
destination: linko.access.log
prefix:      INFO:
```

and a standard application logger:

```text
destination: STDERR
prefix:      DEBUG:
```

Conceptually:

```text
                         Application
                             |
                +------------+------------+
                |                         |
                v                         v
             Server                     Store
                |                         |
                v                         v
        Access Logger              Standard Logger
                |                         |
                v                         v
       linko.access.log                 STDERR
           INFO: ...                  DEBUG: ...
```

This is much better than one giant stream containing every kind of message.

---

# Access Logs vs Application Logs

I now think of the two types of logging differently.

## Access Logs

Access logs describe traffic entering the server.

For example:

```text
INFO: 2026/09/06 20:00:00 Served request: GET /
INFO: 2026/09/06 20:00:03 Served request: GET /api/stats
```

They answer questions such as:

```text
What requests reached the server?
Which paths were requested?
Which HTTP methods were used?
When did requests happen?
```

So the server/request middleware uses the access logger.

---

## Application Logs

Application logs describe what the application itself is doing.

For example:

```text
DEBUG: 2026/09/06 20:01:00 Linko is shutting down
```

Or potentially:

```text
database operation failed
configuration loaded
background task started
cache refreshed
```

These are more useful for understanding internal application behavior.

So the store and shutdown logic use the standard logger.

The distinction is roughly:

```text
Access log
→ what came into the application

Application log
→ what happened inside the application
```

---

# Writing Access Logs to a File

Instead of using `STDERR`, an access logger can write directly to a file.

Conceptually:

```go
file, err := os.OpenFile(
    "linko.access.log",
    os.O_CREATE|os.O_WRONLY|os.O_APPEND,
    0666,
)
if err != nil {
    return err
}
defer file.Close()

accessLogger := log.New(
    file,
    "INFO: ",
    log.LstdFlags,
)
```

Then:

```go
accessLogger.Printf(
    "Served request: %s %s",
    r.Method,
    r.URL.Path,
)
```

goes into:

```text
linko.access.log
```

instead of the terminal.

The important architectural idea is that the server does not need to know this.

From the server's point of view it simply has:

```go
*log.Logger
```

It does not care whether that logger writes to:

```text
STDERR
file
buffer
network
/dev/null
```

The destination is configured outside the server.

That is another example of separating policy from behavior.

---

# Why Dependency Injection Helps Testing

Testing is where the advantage becomes clearer.

With a global:

```text
Test A ──┐
         ├── global logger
Test B ──┘
```

Both tests share the same mutable dependency.

If one test changes the logger configuration, another test can potentially be affected.

With dependency injection:

```text
Test A → logger A → component A

Test B → logger B → component B
```

Each test can own its own dependencies.

For example, a test could create:

```go
var buf bytes.Buffer

logger := log.New(
    &buf,
    "",
    0,
)
```

Run some code and then inspect:

```go
buf.String()
```

The production code does not need to change just because the logger destination changed.

This is the part of dependency injection that makes the idea feel practical rather than theoretical.

---

# One Interesting Progression

The exercises intentionally went through something that was temporarily "better" before replacing it again.

First:

```text
fmt.Println
```

became:

```text
log.Println
```

Then:

```text
log.Println
```

became:

```text
logger.Println
```

using a global logger.

Then the global logger itself was removed and replaced with injected loggers.

At first this can look like rewriting the same thing repeatedly.

But each stage fixes a different problem.

```text
fmt
 |
 | problem: not purpose-built for application logging
 v
log package
 |
 | problem: limited centralized configuration
 v
log.Logger
 |
 | problem: global shared dependency
 v
dependency-injected log.Logger
 |
 | benefit: explicit, configurable, testable dependencies
 v
different loggers for different responsibilities
```

That progression made the architectural reason easier to see.

---

# A Better Mental Model for `run()`

Previously I mostly thought of `run()` as:

```text
start application
```

Now I can also think of it as the place where dependencies are assembled.

Something like:

```text
run()
 |
 ├── open access log file
 |
 ├── create access logger
 |
 ├── create standard logger
 |
 ├── create Store(standard logger)
 |
 ├── create Server(access logger, store, ...)
 |
 └── start server
```

This is sometimes called the application's **composition root**.

I do not need to memorize that term yet.

The more important idea is:

```text
Create dependencies near the top.

Pass them downward.

Do not make lower-level components secretly create or find them.
```

---

# What Clicked

Before today, my mental model for logging was basically:

```text
something happened
      |
      v
print a message
```

Now it is closer to:

```text
                        Application
                             |
            +----------------+----------------+
            |                                 |
            v                                 v
      HTTP Requests                    Internal Behavior
            |                                 |
            v                                 v
     request middleware                 Store / shutdown
            |                                 |
            v                                 v
      access logger                     standard logger
            |                                 |
            v                                 v
   linko.access.log                        STDERR
       INFO: ...                         DEBUG: ...
```

The logger is a dependency.

The component using the logger should not necessarily decide:

```text
where logs go
what prefix they use
what timestamp format they use
```

Those decisions can happen when the application is assembled.

Another thing that clicked is that middleware is a natural place for behavior that applies across many HTTP handlers.

Instead of:

```text
handler A → logging code
handler B → logging code
handler C → logging code
handler D → logging code
```

I can have:

```text
               request logger
                     |
        +------------+------------+
        |            |            |
        v            v            v
    handler A    handler B    handler C
```

And dependency injection connects the whole thing.

The middleware receives a logger.

The server receives a logger.

The store receives a logger.

None of them need to reach into global state.

So the main lesson today ended up being larger than logging:

```text
Dependencies should be explicit.

Create them at the edge/top of the application.

Pass them to the components that need them.
```

Logging was just a very practical way to learn that idea.

---

# Things I Want to Remember

```text
fmt.Println / fmt.Printf
→ normal output and formatting

log.Println / log.Printf
→ basic application logging

log.New(...)
→ create a configurable logger

os.Stderr
→ common destination for application/debug logs

log.LstdFlags
→ standard date + time metadata

http middleware
→ wrap handlers with shared behavior

next.ServeHTTP(w, r)
→ continue the HTTP handler chain

dependency injection
→ pass dependencies instead of hiding them in globals

access logger
→ records incoming HTTP traffic

application logger
→ records internal application behavior
```

And probably the most important progression:

```text
Printing
   ↓
Logging
   ↓
Configurable logging
   ↓
Request logging
   ↓
Injected logging
   ↓
Separate logs for separate responsibilities
```

Today started with replacing `fmt.Printf`.

It ended up teaching me quite a bit about application architecture.
