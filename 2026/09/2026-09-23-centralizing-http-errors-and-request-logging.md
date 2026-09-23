# Centralizing HTTP Errors and Request Logging

Today I cleaned up how HTTP errors and request logging work in a Go service.

The main idea was simple:

> A failed HTTP request should have one path for returning the error and one place responsible for logging it.

Previously, some handlers returned an HTTP error and also logged the same failure themselves.

Conceptually, the flow looked something like this:

```go
logger.Error("failed to validate password", "error", err)

http.Error(
    w,
    "failed to validate password",
    http.StatusInternalServerError,
)
```

This works, but it creates two separate representations of the same failure.

The handler logs the error, then the request logger logs the completed request separately.

That can result in logs like:

```text
ERROR failed to validate password ...
ERROR Served request ...
```

Both lines describe the same request failure.

The change was to make `httpError` the single path for failed HTTP responses.

## `httpError` Now Accepts an Error

Instead of passing a string:

```go
httpError(
    w,
    r,
    http.StatusInternalServerError,
    "failed to validate password",
)
```

the caller passes an `error`:

```go
httpError(
    w,
    r,
    http.StatusInternalServerError,
    errors.New("failed to validate password"),
)
```

When there is an underlying error, it can be wrapped:

```go
httpError(
    w,
    r,
    http.StatusInternalServerError,
    fmt.Errorf("failed to validate password: %w", err),
)
```

This is useful because the HTTP response still gets the readable error message:

```go
err.Error()
```

while the original error remains attached to the error chain.

So internally we don't lose useful information just because the client receives a simple message.

The wrapped cause can still be inspected later for things such as:

- stack traces
- error attributes
- error classification
- debugging information

## Store the Error in the Request Context

`httpError` now has two responsibilities.

First, it stores the error in the request's logging context.

Conceptually:

```go
logContext.Error = err
```

Then it sends the HTTP response:

```go
http.Error(w, err.Error(), status)
```

The important part is that both operations use the **same error object**.

There is no separate logging string that can accidentally become inconsistent with the HTTP response.

## Let the Request Logger Log It Once

The request logger already runs after the handler finishes.

So instead of logging errors throughout individual handlers, the request logger checks whether an error was attached to the request context.

Conceptually:

```go
handler.ServeHTTP(w, r)

attrs := []any{
    "method", r.Method,
    "path", r.URL.Path,
    "status", status,
}

if logContext.Error != nil {
    attrs = append(attrs, "error", logContext.Error)
}

logger.Info("Served request", attrs...)
```

Now a failed request produces one useful log entry:

```text
Served request
method=POST
path=/...
status=500
error="failed to validate password: ..."
```

Instead of having one log from the handler and another from the request middleware.

## Request Context Can Collect More Than Errors

Another thing that clicked is that the request logging context is useful for information discovered during the request.

For example, during password validation the username may already be known:

```go
logContext.Username = username
```

If password checking later fails:

```go
httpError(
    w,
    r,
    http.StatusUnauthorized,
    errors.New("invalid username or password"),
)
```

the final request log can contain both:

```text
Served request
username=someone
status=401
error="invalid username or password"
```

The handler doesn't need to create its own error log just to preserve that context.

It only enriches the request context.

The request logger handles the actual logging.

## What Changed

The old explicit `logger.Error` calls were removed from places such as:

- password validation
- URL lookup
- URL listing

Those failures now go through `httpError`.

A password-check failure still adds the username to the request log context before returning the error.

This keeps useful request-specific information without creating duplicate log entries.

## What Stayed the Same

An important part of this change is that it is mainly an observability refactor.

The application's external HTTP behavior does not change.

The same failures still return:

- the same HTTP status codes
- the same error messages

The difference is what happens internally.

Instead of converting everything into strings early, errors remain errors for longer.

That preserves their causes and gives the logging layer more information to work with.

## The Mental Model

The pattern I ended up with is roughly:

```text
handler
   │
   ├── add useful request context
   │
   └── error
         │
         ▼
     httpError
         │
         ├── store error in request log context
         │
         └── write HTTP response
                 │
                 ▼
           handler returns
                 │
                 ▼
          requestLogger
                 │
                 └── log "Served request"
                     + request metadata
                     + error
                     + extra context
```

Each layer has a clearer responsibility.

The handler handles application logic.

`httpError` converts an application failure into an HTTP failure.

The request log context carries information collected during the request.

`requestLogger` produces the final request log.

## What I Learned

The interesting part wasn't really removing a few `logger.Error` calls.

It was realizing that **logging an error and returning an error are often two parts of the same request lifecycle**.

If every handler logs failures independently, logging becomes scattered throughout the application.

It also becomes easy to:

- log the same failure multiple times
- forget useful request metadata
- use inconsistent error messages
- lose the original error by converting it to a string
- make request logs harder to follow

Centralizing the flow means the handler only needs to report the failure.

The request logger decides how that failure should appear in the logs.

And when there is an underlying cause, using `%w` keeps the original error available instead of flattening everything into text too early.

A useful rule to remember:

> Pass errors as errors for as long as possible. Turn them into strings only at the boundary where a string is actually required.

For HTTP responses, that boundary is the response body.

For logging and debugging, keeping the actual error object around gives much more information.
