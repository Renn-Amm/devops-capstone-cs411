# DevOps Capstone — Node.js App

## What this is
Node.js Express app serving JSON on port 4444, built and deployed
via Jenkins pipeline to a Kubernetes cluster.

## How to run the pipeline
1. Connect this repo to Jenkins via the Seed-Remote job
2. Add a k8s-token credential in Jenkins (Secret text)
3. Trigger the pipeline — it installs deps, runs tests, builds
   the Docker image, pushes to ttl.sh, and deploys to Kubernetes

## How to verify
After the pipeline goes green:
    kubectl get pods -l app=myapp
    kubectl get svc myapp

To curl the app from inside the cluster:
    POD_IP=$(kubectl get pod -l app=myapp -o jsonpath='{.items[0].status.podIP}')
    curl http://$POD_IP:4444/

Expected response:
    {"name":"Hello","description":"World","url":"<host>"}

## Local run
    npm install
    node index.js
    curl http://localhost:4444/
