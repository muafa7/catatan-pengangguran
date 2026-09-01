# From CI to Continuous Deployment with Jenkins

Yesterday I spent most of my time understanding what Jenkins actually does in a pipeline.

The thing that clicked was that Jenkins is mainly the orchestrator. It tells other tools what to do, while the actual build environment still needs to provide things like Python, Node.js, Docker, and so on.

Today felt like the next step from that.

Instead of stopping at:

```text
Checkout
→ Build
→ Test
→ Deliver
```

I started learning about taking the pipeline further into continuous deployment:

```text
Checkout
→ Build
→ Test
→ Deliver
→ Deploy
```

## Jenkins Side

I could still practice the Jenkins parts normally.

What became clearer is the difference between having Jenkins produce a successful artifact and having Jenkins continue the workflow by actually deploying it somewhere.

Before, my pipeline basically ended once the artifact existed.

Now the idea is that a successful build can become the input for the next step:

```text
code change
    ↓
Jenkins
    ↓
build + test
    ↓
artifact
    ↓
deployment
```

So Jenkins is still doing what I understood yesterday: orchestrating.

The difference is that the workflow now extends beyond CI and into actually releasing the application.

I think seeing this right after yesterday's exercise helped a lot because continuous deployment no longer feels like a completely separate concept. It's just another stage in the same pipeline.

## AWS... Mostly Reading Today

The second half moved into AWS: CodeDeploy, Infrastructure as Code, and AWS SAM.

Unfortunately, I couldn't really practice those parts.

AWS could validate my debit card, but when it tried to make the small verification payment, the transaction kept getting rejected by the card.

So the issue wasn't that AWS couldn't recognize or validate the card details. The payment itself just wouldn't go through.

Because of that, I couldn't properly set up the AWS side and had to go through those modules mostly by reading.

From the theory, my current understanding is roughly:

```text
Jenkins
→ decides when the deployment happens

CodeDeploy
→ helps perform the application deployment

Infrastructure as Code
→ describes the infrastructure instead of creating it manually

SAM
→ provides a higher-level way to describe serverless AWS applications
```

I don't want to pretend I properly understand the AWS part yet, though.

The concepts make sense at a high level, but this is exactly the kind of topic that I need to actually use before I feel comfortable saying I learned it.

Reading about deployment and actually configuring it are very different things.

So I'll probably need to come back to the AWS part once I figure out why the card keeps rejecting that verification payment.

## What Clicked

The useful connection for me today was really between yesterday and today.

Yesterday:

```text
Jenkins orchestrates the build workflow.
```

Today:

```text
That workflow doesn't have to stop after the build.
```

It can keep going into deployment.

So I'm starting to see CI/CD less as separate concepts and more as one continuous path:

```text
code
→ build
→ test
→ package
→ deploy
```

I got to practice the Jenkins side of that today.

The AWS side is still mostly theory for now.

But at least the overall picture is starting to make more sense.