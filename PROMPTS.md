# PROMPTS.md — Capstone

## Transfer from Go to Node.js

The previous challenges all used a Go binary. Switching to Node.js
changed several things: no compilation step, dependencies live in
node_modules not a single binary, and the runtime needs to be present
in the container.

I asked:

> "In the Go challenges the Dockerfile copied a single static binary
> into scratch or busybox. Node.js needs the runtime and node_modules
> — what is the right Dockerfile pattern for a Node app?"

The key difference: Go produces a self-contained binary. Node.js needs
the interpreter plus dependencies at runtime. The multi-stage pattern
still applies — builder stage installs all dependencies including dev,
final stage copies only node_modules and source.

## Unit test stage

The Go challenges had no test stage. Adding one for Node.js was new.
I asked:

> "The test file uses node:test which is built into Node 18+. Does
> the test runner exit non-zero if a test fails, so Jenkins will
> fail the build?"

Confirmed: node --test exits with a non-zero code on failure which
is what Jenkins needs to fail the pipeline stage.

## Friction moment — node_modules in Docker

First attempt at the Dockerfile copied node_modules from the workspace
into the image directly. This works locally but breaks in CI because
the Jenkins machine may have a different architecture than the container.
Native modules compiled on the host would fail inside the container.

Fix: use a builder stage that runs npm install inside the container,
then copy the result to the final stage. This ensures node_modules
are built for the container's architecture not the host's.

## Kubernetes — Deployment vs bare Pod

In Challenge 4 the manifest used a bare Pod. For the capstone I used
a Deployment because it handles restarts and rolling updates. I asked:

> "The challenge 4 solution used kind: Pod. For the capstone should
> I use a Deployment instead and what changes in the pipeline?"

The pipeline change: kubectl wait needs -l app=myapp instead of
pod/myapp by name, because a Deployment creates pods with generated
names. The Deployment also means I do not need kubectl delete before
apply — Deployment handles rolling updates natively.

## Verification step

After the pipeline ran I verified with:
    kubectl get pods -l app=myapp
    kubectl get svc myapp

Then curled from inside the cluster using a temporary busybox pod
to confirm the app was actually serving traffic, not just that the
pod was in Running state.
