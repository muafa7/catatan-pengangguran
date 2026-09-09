# Structured Logging with Go `slog`

Yesterday I learned about structured logging in Go using the `log/slog` package.

Before this, I mostly thought of logging as printing some text when something happens.

For example:

```go
log.Println("user logged in")
```

That works, but structured logging adds more context in a format that is easier for both humans and tools to understand.

---

## Structured Logging

Instead of putting everything inside one message:

```text
user 42 logged in from 192.168.1.10
```

Structured logging separates the message from the data:

```text
message="user logged in"
user_id=42
ip=192.168.1.10
```

The important thing that clicked for me is that logs are not only text for developers to read.

They can also be treated like data.

That makes them easier to search, filter, and process later.

---

## Go's `slog` Package

Go provides structured logging through the standard library package:

```go
"log/slog"
```

A simple log looks like:

```go
slog.Info("server started")
```

Instead of manually formatting every log message, `slog` gives a consistent way to attach information to an event.

---

## Log Levels

Logs can have different levels depending on how important the event is.

The main ones are:

```text
DEBUG
INFO
WARN
ERROR
```

My mental model is:

```text
DEBUG
→ useful while investigating what the program is doing

INFO
→ normal application events

WARN
→ something unexpected happened, but the application can continue

ERROR
→ something failed and probably needs attention
```

For example:

```go
slog.Debug("connecting to database")
slog.Info("server started")
slog.Warn("request is taking too long")
slog.Error("database connection failed")
```

This is useful because not every event deserves the same amount of attention.

---

## More Log Levels

Sometimes the default levels are not enough.

`slog` also allows custom log levels.

This can be useful if an application needs more detailed control over which logs should appear.

But for most applications, `DEBUG`, `INFO`, `WARN`, and `ERROR` are usually enough.

---

## Key-Value Pairs

One of the most useful parts of structured logging is adding context as key-value pairs.

For example:

```go
slog.Info(
    "user logged in",
    "user_id", 42,
    "ip", "192.168.1.10",
)
```

Instead of putting the values inside the message itself, they become separate fields.

That means a logging system could later search for something like:

```text
user_id = 42
```

without needing to understand the sentence inside the log message.

---

## Output Formats

`slog` can output logs in different formats.

One option is text:

```text
time=... level=INFO msg="user logged in" user_id=42
```

Another option is JSON:

```json
{
  "time": "...",
  "level": "INFO",
  "msg": "user logged in",
  "user_id": 42
}
```

Text output is nice when I am reading logs directly in the terminal.

JSON makes more sense when logs are sent to systems that need to search or process them.

My mental model is:

```text
Human reading terminal
        ↓
     Text logs

Logging / monitoring system
        ↓
     JSON logs
```

---

## What Clicked

The biggest thing I understood is that logging is more than printing messages.

A useful log contains both:

```text
what happened
+
context about what happened
```

So instead of:

```go
slog.Info("request completed")
```

This is more useful:

```go
slog.Info(
    "request completed",
    "method", "GET",
    "path", "/users",
    "status", 200,
    "duration_ms", 32,
)
```

The message tells me what happened.

The structured fields give me enough context to investigate it later.

The mental model I want to remember is:

```text
Logs are events.

Log levels describe their importance.

Key-value pairs provide context.

Output formats decide how those events are represented.
```

Structured logging feels less like `print()` debugging and more like something designed for running and monitoring a real application.
