# Validating YouTube URLs Without the YouTube Data API

Today I worked on improving YouTube URL validation in a CMS.

I made this feature after noticing that the existing YouTube video field did not properly verify the URL entered by the admin.

That bothered me because this wasn't only about preventing broken YouTube videos.

It could also become a security problem.

The field is supposed to accept a **YouTube video URL**, but if I don't actually validate the hostname and URL structure, technically someone could put something completely different there.

For example:

```text
https://malicious-example.com/file.exe
```

or:

```text
https://malicious-example.com/download/malware
```

or even something that tries to look like YouTube:

```text
https://malicious-example.com/youtube.com/watch?v=NOvCNw0-xaw
```

If the application simply trusts the value because it contains something like:

```text
youtube.com
```

then that input could potentially be stored in the CMS and later rendered somewhere on the website.

Depending on how that value is used by the frontend, it could become:

- a clickable link,
- a redirect destination,
- an iframe source,
- a media source,
- or another URL exposed to website visitors.

So the real problem wasn't just:

> "What if the YouTube video doesn't exist?"

It was also:

> "Why does a field that is supposed to contain only YouTube videos accept arbitrary URLs at all?"

That is the part I wanted to fix.

---

## The Problem

The existing behavior was roughly:

```text
Admin enters URL
        ↓
Extract something that looks like a video ID
        ↓
Generate thumbnail
        ↓
Save
```

There wasn't a proper trust boundary between:

```text
user input
```

and:

```text
trusted YouTube content
```

The application needed to verify several things before treating the value as a YouTube video.

It should confirm:

- the value is actually a valid URL,
- the hostname really belongs to YouTube,
- the URL points to an individual video,
- the video ID has a valid structure,
- the video actually exists,
- the video is available,
- and the video can be embedded.

The most important part from a security perspective is the hostname validation.

I don't want something like this:

```text
https://malicious-example.com/youtube.com/watch?v=NOvCNw0-xaw
```

to ever be accepted as a YouTube URL.

---

## Why `includes("youtube.com")` Is Dangerous

An implementation like this is tempting:

```ts
if (url.includes("youtube.com")) {
  // accept it
}
```

But this does not validate the domain.

For example:

```text
https://malicious-example.com/youtube.com/watch?v=NOvCNw0-xaw
```

contains:

```text
youtube.com
```

So a simple string check could consider it valid.

But the actual hostname is:

```text
malicious-example.com
```

That's a completely different website.

This is why URL validation should use the browser's URL parser:

```ts
const parsedUrl = new URL(value);
```

Then explicitly inspect:

```ts
parsedUrl.hostname
```

For this CMS, only these hosts should be trusted:

```text
youtube.com
www.youtube.com
m.youtube.com
youtu.be
```

For example:

```ts
const allowedHosts = new Set([
  "youtube.com",
  "www.youtube.com",
  "m.youtube.com",
  "youtu.be",
]);

const parsedUrl = new URL(value);

if (!allowedHosts.has(parsedUrl.hostname.toLowerCase())) {
  throw new Error("Only YouTube URLs are allowed");
}
```

Now:

```text
https://www.youtube.com/watch?v=NOvCNw0-xaw
```

is allowed.

But:

```text
https://malicious-example.com/youtube.com/watch?v=NOvCNw0-xaw
```

is rejected.

And:

```text
https://malicious-example.com/file.exe
```

is obviously rejected too.

---

## The Security Boundary I Actually Want

Instead of:

```text
Admin Input
    ↓
Trust It
    ↓
Store / Display
```

I want:

```text
Admin Input
    ↓
Parse URL
    ↓
Check Allowed Hostname
    ↓
Check Supported YouTube URL Structure
    ↓
Extract Video ID
    ↓
Validate Video ID
    ↓
Verify Video With YouTube
    ↓
Trusted YouTube Video
    ↓
Store / Display
```

The important idea here is:

**Input should not become trusted content just because an admin entered it.**

An admin panel is still an input surface.

The CMS should enforce what kind of data belongs in each field.

If a field says:

```text
YouTube Video URL
```

then arbitrary URLs should not be accepted.

---

## One Important Detail

The actual severity depends on how the application uses the saved URL.

If the application only extracts the video ID and then constructs its own trusted URL such as:

```text
https://www.youtube.com/embed/VIDEO_ID
```

then an arbitrary malicious URL has much less opportunity to reach the visitor directly.

But if the original CMS value is later used directly as something like:

```tsx
<a href={videoUrl}>
```

or:

```tsx
<iframe src={videoUrl}>
```

or:

```tsx
window.location.href = videoUrl;
```

then strict URL validation becomes much more important.

So besides improving validation, I also want to make sure the application follows this principle:

```text
Store/receive URL
      ↓
Extract trusted video ID
      ↓
Use video ID
      ↓
Construct known YouTube URLs ourselves
```

rather than repeatedly trusting the original raw URL.

---

## My Goal

The final validation flow should therefore be:

```text
Admin enters URL
        ↓
Parse URL
        ↓
Is hostname explicitly allowed?
       / \
     No   Yes
     ↓     ↓
 Reject  Extract Video ID
              ↓
        Is ID structurally valid?
             / \
           No   Yes
           ↓     ↓
         Reject Verify with YouTube
                    ↓
                 Success?
                  /   \
                No     Yes
                ↓       ↓
             Reject   Valid
                        ↓
                 Show Thumbnail
                        ↓
                Allow Save/Publish
```

So this feature is doing two things:

1. **Security / input restriction**
   - Don't allow arbitrary external URLs where only YouTube should be accepted.

2. **Content validation**
   - Don't treat a syntactically valid YouTube URL as a valid video until YouTube itself can load it.

That is a much better trust model than simply extracting whatever looks like a video ID and hoping the input is safe.
