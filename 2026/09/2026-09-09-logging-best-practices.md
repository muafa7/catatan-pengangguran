# Logging Best Practices

Today I learned some best practices for writing useful logs.

Logging is not just about writing as much information as possible.

Good logs should help me understand what happened without creating too much noise.

---

## Best Practices

A good log should be useful when I need to investigate something.

That means logs should have enough context, but they should not contain unnecessary information.

A useful mental model is:

```text
good logging
=
useful information
-
unnecessary noise
```

The goal is to make logs easy to read and easy to search.

---

## Timestamps

Logs should include timestamps.

For example:

```text
2026-09-09T10:15:32Z level=INFO msg="request completed"
```

Without a timestamp, it becomes difficult to know when something happened.

Timestamps are especially important when I need to compare events between different services.

For distributed systems, using a consistent timezone such as UTC also makes logs easier to compare.

---

## Minimal Logging

More logs do not always mean better logs.

If I log too much, important information can get buried.

For example, logging every tiny internal step may create a lot of noise:

```text
starting function
checking variable
variable exists
calling another function
function returned
```

Instead, I should log events that are actually useful for debugging or monitoring.

The idea is:

```text
log what is useful,
not everything that happens
```

---

## Redundant Logs

I should avoid logging the same event multiple times.

For example:

```text
service: database query failed
repository: database query failed
handler: database query failed
```

This can make one error look like three different errors.

It also creates unnecessary log volume.

Instead, the error can be passed upward and logged at the place where there is enough context to understand what happened.

---

## One Log Per Event

One event should usually produce one useful log entry.

For example, instead of:

```text
request started
user found
processing request
request finished
```

I can sometimes produce one structured log:

```text
level=INFO
msg="request completed"
method=GET
path=/users/42
status=200
duration_ms=35
```

This gives me the important information about the event without creating several separate logs.

It also makes searching and analyzing logs easier.

---

## What Clicked

The biggest thing I learned today is that good logging is not about logging everything.

It is about logging the right information.

My mental model is:

```text
timestamps
→ tell me when something happened

minimal logging
→ reduce unnecessary noise

avoid redundant logs
→ prevent the same problem from appearing multiple times

one log per event
→ keep related information together
```

A good log should help me answer:

```text
What happened?

When did it happen?

Where did it happen?

What context do I need to investigate it?
```

If the log cannot help answer those questions, it might not be useful.
