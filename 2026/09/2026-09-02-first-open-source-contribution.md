# My First Open Source Contribution

Today I opened my first pull request to an open source project.

The actual code change is very small.

Like, *two lines of implementation change* small.

But I'm still really excited about it because this is my first time contributing to open source.

The issue I worked on was [nakama#374](https://github.com/ahmadrosid/nakama/issues/374), specifically part (a), which was about Telegram formatting corrupting certain dollar-sign replacement sequences.

## The Problem

The Telegram formatter temporarily replaces some content with placeholders and restores it later.

The relevant code looked like this:

```ts
result = result.replace(`@@TCURL${index}@@`, urls[index]!);
```

and:

```ts
result = result.replace(`@@TCTOKEN${index}@@`, blocks[index]!);
```

At first glance, it looks completely normal.

The problem is how JavaScript's `String.replace()` works.

When the replacement value is passed directly as a string, some dollar-sign sequences have special meanings.

For example:

```text
$&
$`
$'
```

Those sequences aren't always treated as literal text.

So if a URL or protected Telegram content contains one of them, `String.replace()` can interpret it as replacement syntax and corrupt the original value.

## Reproducing It First

Instead of immediately changing the implementation, I added a regression test first.

For the bare URL case:

```ts
test("preserves dollar replacement sequences in bare URLs", () => {
  const url = "https://example.com/$&/path";

  expect(stripMarkdownForTelegram(url)).toBe(url);
});
```

Then I ran:

```bash
bun test apps/platform/telegram/src/format.test.ts
```

and it failed.

That was probably one of my favorite parts of the whole process.

The fix itself looked obvious once I understood the issue, but having a failing test first meant I wasn't just changing code based on an assumption.

I could actually reproduce the bug.

I did the same thing for fenced code blocks.

Before the fix, content containing `$&` could end up looking something like:

```text
<pre><code>@@TCTOKEN0@@amp;</code></pre>
```

instead of preserving the original content.

So now I had a clear before and after.

## The Fix

The actual fix was tiny.

Instead of passing the replacement value directly:

```ts
result.replace(token, value);
```

I changed it to use a replacement callback:

```ts
result.replace(token, () => value);
```

So the two implementation changes became:

```ts
result = result.replace(`@@TCURL${index}@@`, () => urls[index]!);
```

and:

```ts
result = result.replace(`@@TCTOKEN${index}@@`, () => blocks[index]!);
```

The callback makes JavaScript treat the returned value literally instead of interpreting dollar-sign replacement patterns.

After that, the tests passed.

I also added a regression test for fenced code blocks covering the replacement sequences mentioned in the issue:

```ts
test("preserves dollar replacement sequences in fenced code blocks", () => {
  expect(renderTelegramRichText("```text\n$& $` $'\n```")).toBe(
    "<pre><code>$&amp; $` $'</code></pre>"
  );
});
```

The final diff was:

```text
2 files changed
14 insertions(+)
2 deletions(-)
```

That's it.

Two implementation lines.

## What Felt Different

I've been working professionally for around three years, so working in a codebase I didn't write isn't new to me.

That's not what made this feel different.

The difference is that this is open source.

The issue is public.

The discussion is public.

The pull request is public.

The CI results are public.

And the people reviewing the change don't know me or have any context beyond what I put into the issue, code, tests, commit, and PR.

That made me think about the contribution differently.

I paid more attention to whether the scope was actually mine to work on, whether somebody else had already changed related code, how other contributors structured their branches and commits, whether the bug could be reproduced properly, and whether the diff was small enough for a maintainer to understand quickly.

None of those things are completely foreign to normal professional development.

But doing all of it in public, in a project I don't own, felt different.

## Opening the PR

After the tests passed and I reviewed the final diff, I committed it as:

```text
fix(telegram): preserve dollar replacement sequences
```

Then I opened the pull request.

Issue #374 contains several separate findings, and my PR only addresses part (a), so I intentionally didn't use:

```text
Closes #374
```

Instead I wrote:

```text
Addresses part (a) of #374.
```

It's a small detail, but I liked learning that too.

My change shouldn't imply that the entire issue is solved when I'm only responsible for one part of it.

At this point the PR is open and some CI workflows are waiting for maintainer approval.

So now I wait.

Maybe everything passes.

Maybe a maintainer asks me to change something.

Maybe I misunderstood part of the project and have to revise the PR.

And honestly, I'm excited about that too.

I want to experience the review part, not just the coding part.

## What I Learned

Before today, open source contribution looked simpler in my head:

```text
find issue
-> fix issue
-> open PR
```

Now I see more of the process around it:

```text
understand the issue
-> check the current state of the code
-> see what other contributors already changed
-> understand the project's conventions
-> reproduce the bug
-> add a regression test
-> make the smallest fix
-> run the tests
-> inspect the diff
-> commit
-> open the PR
-> wait for CI and review
```

The actual code change was probably the smallest part.

And I think that's what made today interesting.

## First One

I know this isn't a huge contribution.

I changed two `String.replace()` calls and added tests.

That's it.

But this was my first time contributing in public to a project I don't own, where the issue, discussion, PR, CI, and review are all visible.

That made the process feel very different from normal professional development.

I'm genuinely excited about it.

Maybe in the future this kind of contribution will feel normal.

But this one is the first.

Tiny diff.

Big milestone for me.