# PROMPTS.md

## Background

The previous challenges were all Go — a compiled language that produces a single static binary you can copy anywhere. Switching to Node.js for the capstone changed almost everything about the deploy story: no compilation step, the runtime has to exist on the target machine, and `node_modules` needs to travel with the app.

## Prompt 1 — Dockerfile approach

My first instinct was to copy the Dockerfile from the Go challenges and just swap the binary for `index.js`. That obviously wouldn't work — Node needs the interpreter. I asked:

> "In the Go challenges the Dockerfile just copied a binary into a minimal image. Node.js needs the runtime and node_modules at runtime — what's the right pattern here, and does multi-stage still make sense?"

The answer was yes, multi-stage still applies but differently. Stage 1 runs `npm install` inside the container so the modules are compiled for the right architecture. Stage 2 copies only what's needed — `node_modules` and `index.js`. This matters because if you run `npm install` on the Jenkins host and then copy the modules, native addons would be compiled for the host OS/arch, not the container's.

## Friction moment 1 — sudo in Jenkins

The install stage was failing with:
sudo: a terminal is required to read the password

Jenkins runs as the `jenkins` user which doesn't have passwordless sudo by default. In the Go challenges this wasn't an issue because Go was already installed. I had to add:

```bash
echo "jenkins ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/jenkins
```

on the jenkins machine before the pipeline could install Node.js. This was the first thing that blocked me and took a run to figure out — the error message is clear but not obvious if you haven't hit it before.

## Friction moment 2 — node_modules on target

The target deploy kept failing because scp couldn't write into `/opt/myapp/node_modules` — the directory didn't exist yet. The fix was to SSH in first and create it before copying:

```bash
ssh laborant@target "sudo mkdir -p /opt/myapp/node_modules && sudo chown -R laborant:laborant /opt/myapp"
```

then scp the contents with `node_modules/.` instead of `node_modules` to copy into the existing directory rather than trying to rename it.

## Prompt 2 — Kubernetes: Deployment vs bare Pod

In Challenge 4 I used a bare Pod. I wasn't sure whether to do the same here or switch to a Deployment, so I asked:

> "Challenge 4 used kind: Pod with kubectl delete before apply. For the capstone should I use a Deployment instead, and what changes in how I wait for it to be ready in the pipeline?"

The key difference: with a Deployment you use `kubectl rollout status deployment/myapp` instead of `kubectl wait --for=condition=Ready pod/myapp`, because the pod name is generated (e.g. `myapp-7b8cc5455-rwqkw`) and you can't reference it directly. I ended up using both — a Deployment + Service for the proper Kubernetes setup, and a bare Pod named `myapp` to satisfy the iximiuz checker which looks for that exact name.

## Verification step

After the pipeline went green I checked each target manually before calling it done:

```bash
# kubernetes tab
kubectl get pods
# confirmed Running, not just Pending

POD_IP=$(kubectl get pod -l app=myapp -o jsonpath='{.items[0].status.podIP}')
curl http://$POD_IP:4444/
# confirmed JSON response, not connection refused
```

The pod showing `Running` in `kubectl get pods` doesn't mean the app is actually serving — it just means the container started. Curling the pod IP directly confirmed the app was up inside the cluster.

## Transfer summary

The pipeline shape stayed the same as the Go challenges — install, test, build image, push, deploy. What changed was everything underneath: no static binary means node_modules has to go everywhere the app goes, the Dockerfile needs a real runtime, and the systemd unit needs `WorkingDirectory` set because Node resolves `require()` relative to the current directory. Re-deriving each step for Node.js rather than copy-pasting from Go made the differences obvious.
