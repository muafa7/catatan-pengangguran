# HTTP Request & Response Logging

Today I learned about adding more **context to HTTP request logs** in Go.

A log message by itself might be correct, but without enough context it can be difficult to understand what actually happened.

For example:

> "I wish the ring had never come to me."
> – Frodo Baggins

The quote makes sense when we know what happened before it.

But imagine Frodo saying this when he first meets Gandalf in the Shire instead of saying:

> "You're late!"

Gandalf would probably be confused.

What ring?

There is not enough context.

The same idea applies to application logs.

If my server only logs:

```text
Served request
```

I know that a request was served, but I don't really know what happened.

It becomes much more useful when the log contains additional information such as:

```text
duration=12ms
request_body_bytes=128
response_status=200
response_body_bytes=512
```

Now the log gives me some context about the request and response.

---

## HTTP Request Context

For an HTTP server, there are many pieces of information that could be useful to include in logs.

For example:

- Number of bytes in the request body
- Authenticated user information
- Response status code
- Number of bytes in the response body
- Response duration
- Request `User-Agent`
- Request `Content-Type`
- Relevant cookies
- Response `Content-Type`

This doesn't mean I should log everything.

The important thing is to add information when it becomes useful.

Logging too much can make logs noisy and can also be dangerous if sensitive information accidentally gets logged.

---

# Logging Response Duration

The first thing I learned to track is the **response duration**.

The idea is simple.

Record when the request starts:

```go
start := time.Now()
```

Then let the next HTTP handler process the request:

```go
next.ServeHTTP(w, r)
```

After the handler finishes, calculate how much time has passed:

```go
time.Since(start)
```

So the middleware can look like this:

```go
func requestLogger(logger *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			start := time.Now()

			next.ServeHTTP(w, r)

			logger.Info("Served request",
				slog.Duration("duration", time.Since(start)),
			)
		})
	}
}
```

The important part is that:

```go
next.ServeHTTP(w, r)
```

runs before:

```go
time.Since(start)
```

So the flow is basically:

```text
Request arrives
      |
      v
start := time.Now()
      |
      v
next.ServeHTTP()
      |
      v
Handler finishes
      |
      v
time.Since(start)
```

This tells me how long the server took to process the request.

For example:

```text
duration=2.5ms
```

or maybe:

```text
duration=4.2s
```

If requests suddenly become slow, this information can help me notice the problem from the logs.

---

# Logging Request Body Size

The next thing I learned is how to track how many bytes are actually read from the request body.

Normally the request body is available from:

```go
r.Body
```

The type of `r.Body` implements:

```go
io.ReadCloser
```

Instead of changing how the handler reads the request body, I can create a wrapper around the existing `io.ReadCloser`.

The wrapper is called:

```go
spyReadCloser
```

For example:

```go
type spyReadCloser struct {
	io.ReadCloser
	bytesRead int
}
```

There are two things inside it:

```go
io.ReadCloser
```

and:

```go
bytesRead int
```

The original `ReadCloser` still does the actual work.

The additional `bytesRead` field is just there so I can count how many bytes have been read.

---

## Intercepting Read()

The wrapper implements its own `Read()` method:

```go
func (r *spyReadCloser) Read(p []byte) (int, error) {
	n, err := r.ReadCloser.Read(p)
	r.bytesRead += n
	return n, err
}
```

The important line is:

```go
n, err := r.ReadCloser.Read(p)
```

I am still calling the original request body's `Read()` method.

So the wrapper doesn't replace the actual reading behavior.

It only observes it.

After the original body is read, Go gives me:

```go
n
```

which represents how many bytes were read.

Then I add that number to:

```go
r.bytesRead
```

using:

```go
r.bytesRead += n
```

So if the body is read several times:

```text
Read #1 -> 100 bytes
Read #2 -> 200 bytes
Read #3 -> 50 bytes
```

the wrapper keeps accumulating them:

```text
bytesRead = 350
```

---

# Replacing r.Body With the Spy

Inside the middleware, I create the wrapper:

```go
spyReader := &spyReadCloser{
	ReadCloser: r.Body,
}
```

Then replace:

```go
r.Body
```

with:

```go
spyReader
```

So:

```go
r.Body = spyReader
```

Now when the next handler reads:

```go
r.Body
```

it is actually reading through my `spyReadCloser`.

The handler doesn't need to know that.

From the handler's perspective, `r.Body` is still an `io.ReadCloser`.

The flow becomes:

```text
Handler
   |
   v
r.Body.Read()
   |
   v
spyReadCloser.Read()
   |
   +---- count bytes
   |
   v
original r.Body.Read()
```

After the handler finishes:

```go
next.ServeHTTP(w, r)
```

I can inspect:

```go
spyReader.bytesRead
```

and add it to the log:

```go
slog.Int("request_body_bytes", spyReader.bytesRead)
```

For example:

```go
func requestLogger(logger *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			spyReader := &spyReadCloser{
				ReadCloser: r.Body,
			}

			r.Body = spyReader

			next.ServeHTTP(w, r)

			logger.Info("Served request",
				slog.Int("request_body_bytes", spyReader.bytesRead),
			)
		})
	}
}
```

---

# Logging Response Metadata

The response is a little different.

Go gives the handler:

```go
http.ResponseWriter
```

For example:

```go
func handler(w http.ResponseWriter, r *http.Request) {
	// ...
}
```

The problem is that after the handler finishes, the default `http.ResponseWriter` doesn't give me an easy way to ask:

```text
What status code did you send?
```

or:

```text
How many bytes did you write?
```

There isn't something like:

```go
w.StatusCode()
```

that I can simply call afterward.

But `http.ResponseWriter` is an **interface**.

That means I can create my own implementation that wraps the original `ResponseWriter`.

This is similar to what I did with the request body.

---

# spyResponseWriter

I can create:

```go
type spyResponseWriter struct {
	http.ResponseWriter
	bytesWritten int
	statusCode   int
}
```

It contains the original:

```go
http.ResponseWriter
```

and two additional fields:

```go
bytesWritten int
statusCode   int
```

The original `ResponseWriter` still handles the real HTTP response.

My wrapper just watches what happens.

---

# Tracking Response Body Bytes

When a handler wants to send response data, it might call:

```go
w.Write(data)
```

So I can intercept `Write()`:

```go
func (w *spyResponseWriter) Write(p []byte) (int, error) {
	if w.statusCode == 0 {
		w.statusCode = http.StatusOK
	}

	n, err := w.ResponseWriter.Write(p)
	w.bytesWritten += n

	return n, err
}
```

The actual response is still written using:

```go
w.ResponseWriter.Write(p)
```

So the wrapper doesn't prevent the response from being sent.

It simply gets the number of bytes that were successfully written:

```go
n
```

and stores them:

```go
w.bytesWritten += n
```

For example:

```text
Write #1 -> 100 bytes
Write #2 -> 300 bytes
Write #3 -> 50 bytes
```

will result in:

```text
bytesWritten = 450
```

---

# Why statusCode Defaults to 200

There is an interesting detail inside `Write()`:

```go
if w.statusCode == 0 {
	w.statusCode = http.StatusOK
}
```

A Go HTTP handler doesn't always explicitly call:

```go
w.WriteHeader(http.StatusOK)
```

For example, this is valid:

```go
func handler(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello"))
}
```

There is no:

```go
w.WriteHeader(200)
```

but Go automatically sends:

```text
200 OK
```

when the response body starts being written.

So if `Write()` happens and I haven't seen a status code yet:

```go
w.statusCode == 0
```

I know the response is effectively:

```go
http.StatusOK
```

which is:

```text
200
```

That's why the wrapper does:

```go
if w.statusCode == 0 {
	w.statusCode = http.StatusOK
}
```

---

# Tracking Explicit Status Codes

A handler can also explicitly send a status code:

```go
w.WriteHeader(http.StatusNotFound)
```

which means:

```text
404
```

To capture this, the wrapper also implements:

```go
func (w *spyResponseWriter) WriteHeader(statusCode int) {
	w.statusCode = statusCode
	w.ResponseWriter.WriteHeader(statusCode)
}
```

So when the handler calls:

```go
w.WriteHeader(404)
```

my wrapper first stores:

```go
w.statusCode = 404
```

then forwards the real operation:

```go
w.ResponseWriter.WriteHeader(404)
```

The response still works normally.

I just get to remember what status code was sent.

---

# Using spyResponseWriter

Inside the middleware, instead of directly passing:

```go
w
```

to:

```go
next.ServeHTTP()
```

I first wrap it:

```go
spyWriter := &spyResponseWriter{
	ResponseWriter: w,
}
```

Then:

```go
next.ServeHTTP(spyWriter, r)
```

Now the handler thinks it is using a normal:

```go
http.ResponseWriter
```

but it is actually using:

```go
spyResponseWriter
```

The flow looks something like:

```text
Handler
   |
   v
spyResponseWriter
   |
   +---- track status code
   |
   +---- track bytes written
   |
   v
original http.ResponseWriter
   |
   v
HTTP response
```

After:

```go
next.ServeHTTP(spyWriter, r)
```

finishes, I can access:

```go
spyWriter.statusCode
```

and:

```go
spyWriter.bytesWritten
```

Then log them:

```go
slog.Int("response_status", spyWriter.statusCode),
slog.Int("response_body_bytes", spyWriter.bytesWritten),
```

---

# Putting Everything Together

Now I can combine all three ideas:

1. Measure request duration
2. Count request body bytes
3. Capture response status and response body bytes

First, the request body spy:

```go
type spyReadCloser struct {
	io.ReadCloser
	bytesRead int
}

func (r *spyReadCloser) Read(p []byte) (int, error) {
	n, err := r.ReadCloser.Read(p)
	r.bytesRead += n
	return n, err
}
```

Then the response writer spy:

```go
type spyResponseWriter struct {
	http.ResponseWriter
	bytesWritten int
	statusCode   int
}

func (w *spyResponseWriter) Write(p []byte) (int, error) {
	if w.statusCode == 0 {
		w.statusCode = http.StatusOK
	}

	n, err := w.ResponseWriter.Write(p)
	w.bytesWritten += n

	return n, err
}

func (w *spyResponseWriter) WriteHeader(statusCode int) {
	w.statusCode = statusCode
	w.ResponseWriter.WriteHeader(statusCode)
}
```

Then use both inside the middleware:

```go
func requestLogger(logger *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			start := time.Now()

			spyReader := &spyReadCloser{
				ReadCloser: r.Body,
			}

			r.Body = spyReader

			spyWriter := &spyResponseWriter{
				ResponseWriter: w,
			}

			next.ServeHTTP(spyWriter, r)

			logger.Info("Served request",
				slog.Duration(
					"duration",
					time.Since(start),
				),
				slog.Int(
					"request_body_bytes",
					spyReader.bytesRead,
				),
				slog.Int(
					"response_status",
					spyWriter.statusCode,
				),
				slog.Int(
					"response_body_bytes",
					spyWriter.bytesWritten,
				),
			)
		})
	}
}
```

---

# Full Request Flow

The complete flow now makes more sense to me:

```text
HTTP Request
     |
     v
requestLogger
     |
     +---- start := time.Now()
     |
     +---- wrap r.Body
     |        |
     |        v
     |   spyReadCloser
     |
     +---- wrap ResponseWriter
     |        |
     |        v
     |   spyResponseWriter
     |
     v
next.ServeHTTP()
     |
     +---- Handler reads request body
     |        |
     |        v
     |   count request bytes
     |
     +---- Handler calls WriteHeader()
     |        |
     |        v
     |   capture status code
     |
     +---- Handler writes response
     |        |
     |        v
     |   count response bytes
     |
     v
Handler finishes
     |
     +---- calculate duration
     |
     +---- read request bytes
     |
     +---- read response status
     |
     +---- read response bytes
     |
     v
logger.Info("Served request")
```

So instead of just:

```text
Served request
```

I can have something closer to:

```text
Served request
duration=5.2ms
request_body_bytes=124
response_status=200
response_body_bytes=512
```

That gives me much more useful information when debugging or monitoring the server.

---

# The Bigger Thing I Learned

The most interesting part of this lesson for me is actually not the logging itself.

It is the idea of **wrapping an interface to observe what happens without changing the actual business logic**.

For the request body:

```go
io.ReadCloser
```

becomes:

```text
Handler
   |
   v
spyReadCloser
   |
   v
original io.ReadCloser
```

For the response:

```go
http.ResponseWriter
```

becomes:

```text
Handler
   |
   v
spyResponseWriter
   |
   v
original http.ResponseWriter
```

The wrapper does some extra work and then delegates the actual operation to the original object.

For example:

```go
func (r *spyReadCloser) Read(p []byte) (int, error) {
	n, err := r.ReadCloser.Read(p)

	// extra behavior
	r.bytesRead += n

	return n, err
}
```

The important part is:

```go
r.ReadCloser.Read(p)
```

The real implementation is still doing the work.

The wrapper is just observing it.

The same thing happens here:

```go
func (w *spyResponseWriter) Write(p []byte) (int, error) {
	n, err := w.ResponseWriter.Write(p)

	// extra behavior
	w.bytesWritten += n

	return n, err
}
```

Again, the original:

```go
w.ResponseWriter.Write(p)
```

still handles the real response.

---

# Why This Pattern Is Useful

This makes me understand why interfaces are powerful in Go.

If my code depends on an interface, I can insert another implementation between the caller and the real object.

Something like:

```text
Caller
  |
  v
Wrapper
  |
  +---- observe
  +---- measure
  +---- log
  +---- modify if needed
  |
  v
Real Implementation
```

This kind of pattern could be useful for more than HTTP request logging.

For example:

```text
Logging
Metrics
Tracing
Auditing
Debugging
Counting operations
Measuring execution time
Monitoring external calls
```

The business logic doesn't necessarily need to know about any of it.

---

# What I Want to Remember

The main thing I want to remember from today's lesson:

**Useful logs need context.**

This:

```text
Served request
```

isn't very useful by itself.

But this:

```text
Served request
duration=5ms
request_body_bytes=250
response_status=201
response_body_bytes=480
```

already tells me much more about what happened.

And the interesting Go technique behind it is:

**wrap existing interfaces, intercept the operations I care about, record information, then delegate the real operation to the original implementation.**

For the request:

```go
r.Body = spyReader
```

For the response:

```go
next.ServeHTTP(spyWriter, r)
```

The handler can continue working normally.

The middleware gets the information it needs.

And the actual request/response behavior stays mostly unchanged.

That's the part that clicked for me today.
