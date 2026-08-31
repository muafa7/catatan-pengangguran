# Jenkins Scripted Pipeline

Today I worked on a Jenkins CI pipeline for a Python project using a scripted pipeline.

The Python project itself was already provided, so the interesting part for me was mostly the Jenkinsfile and the environment where Jenkins actually runs the commands.

## The Pipeline

```groovy
node {
    deleteDir()

    stage('Checkout') {
        checkout scm
    }

    stage('Setup') {
        sh 'python3 -m venv .venv'
        sh '.venv/bin/pip install pytest pyinstaller'
    }

    stage('Build') {
        sh '.venv/bin/python -m py_compile sources/add2vals.py sources/calc.py'
    }

    stage('Test') {
        sh 'mkdir -p test-reports'
        sh '.venv/bin/python -m pytest --verbose --junitxml=test-reports/results.xml sources/test_calc.py'
        junit 'test-reports/results.xml'
    }

    stage('Deliver') {
        sh '.venv/bin/python -m PyInstaller --onefile sources/add2vals.py'
        archiveArtifacts artifacts: 'dist/add2vals'
    }
}
```

The flow is basically:

```text
Checkout
→ Setup
→ Build
→ Test
→ Deliver
```

One thing that became clearer to me is that Jenkins mostly orchestrates the workflow.

For example, pytest generates the test report:

```groovy
sh '.venv/bin/python -m pytest --verbose --junitxml=test-reports/results.xml sources/test_calc.py'
```

Then Jenkins reads that report:

```groovy
junit 'test-reports/results.xml'
```

The same thing happens with the artifact.

PyInstaller creates the executable, then Jenkins preserves it with:

```groovy
archiveArtifacts artifacts: 'dist/add2vals'
```

So Jenkins is not really doing the compilation or testing itself. It coordinates the tools that do the actual work.

## The Environment Problem

This was the part that made me think more about how Jenkins actually executes a pipeline.

Previously, I worked on a React pipeline that used:

```groovy
agent {
    docker {
        image 'node:16-buster-slim'
        args '-p 3000:3000'
    }
}
```

In that setup, the pipeline commands were executed inside a Node.js Docker container.

So even if the Jenkins container itself did not have Node.js installed, commands such as:

```groovy
sh 'npm install'
```

could still work because Node and npm existed inside the Docker agent.

My Python pipeline was different.

I wrote:

```groovy
node {
    ...
    sh 'python3 -m venv .venv'
}
```

There was no separate Python Docker environment declared there.

That meant `python3` had to exist in the environment where Jenkins was actually executing the job.

My existing Jenkins container did not have Python installed, so I created another image based on it:

```dockerfile
FROM myjenkins-blueocean:lts

USER root

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        python3 \
        python3-venv \
        python3-dev \
        binutils \
        build-essential \
        libpython3.13 \
    && rm -rf /var/lib/apt/lists/*

USER jenkins
```

Instead of rebuilding my Jenkins setup from scratch, I extended the image I already had and added the tools required by the pipeline.

## What Clicked

Before this, I was mostly thinking:

> Jenkins runs my pipeline.

Now I think about it more like:

```text
Jenkinsfile
    ↓
Jenkins orchestrates the workflow
    ↓
some execution environment runs the commands
    ↓
that environment needs the required tools
```

The important part is not whether Jenkins itself has Python, Node.js, Java, or anything else.

The important part is that those tools exist in the environment where the job is actually running.

That also made me understand why using Docker agents can be cleaner.

Instead of making one Jenkins image that slowly contains:

```text
Python
Node.js
Java
Maven
Go
...
```

each project can use an environment that matches its own requirements.

For example:

```text
Jenkins
├── Node.js container
├── Python container
├── Java container
└── ...
```

The Jenkins server stays focused on orchestration, while the build environment provides the project-specific tooling.

For this exercise, extending my Jenkins image worked fine and was useful for learning.

For a larger CI setup, I would probably prefer keeping Jenkins relatively clean and using disposable build environments for each workload.

## Scripted Pipeline

This was also my first time intentionally using a scripted pipeline.

Previously, I was more familiar with declarative pipelines:

```text
pipeline
└── stages
    └── stage
        └── steps
```

With the scripted pipeline, I started directly with:

```groovy
node {
    ...
}
```

and defined the stages inside it.

For this pipeline, I don't think scripted pipeline was necessarily better. The workflow was simple enough that declarative pipeline would probably be easier to read.

What I learned is that scripted pipeline feels more like programming the flow of the pipeline, while declarative pipeline gives me a more predefined structure.

The biggest lesson today was not really scripted versus declarative.

It was understanding the separation between **Jenkins as the orchestrator** and **the environment that actually executes the build**.
