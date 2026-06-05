# CS411 Capstone — Node.js CI/CD Pipeline

Node.js Express app deployed via Jenkins to three targets: a Linux VM (systemd), a Docker container, and a Kubernetes cluster.

## Repository Structure

```text
.
├── index.js                  # Express application
├── index.test.js             # Unit tests (node:test)
├── package.json              # Project dependencies
├── package-lock.json         # Dependency lock file
├── Dockerfile                # Multi-stage Docker build
├── Jenkinsfile               # CI/CD pipeline definition
├── PROMPTS.md                # Assignment prompts/documentation
│
├── k8s/                      # Kubernetes manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   └── pod.yaml
│
└── systemd/                  # Linux service configuration
    └── myapp.service
```

## Prerequisites (iximiuz playground)

You need four tabs open: jenkins, target, docker, kubernetes.

**On the jenkins tab — one-time setup:**
```bash
# Give Jenkins passwordless sudo
echo "jenkins ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/jenkins

# Generate SSH key for jenkins user
sudo -u jenkins ssh-keygen -t ed25519 -f /var/lib/jenkins/.ssh/id_ed25519 -N ""
sudo cat /var/lib/jenkins/.ssh/id_ed25519.pub
```

**On the target tab — add the key:**
```bash
sudo bash -c 'echo "YOUR_PUBLIC_KEY" >> /home/laborant/.ssh/authorized_keys && chmod 600 /home/laborant/.ssh/authorized_keys'

# Install Node.js
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt-get install -y nodejs

# Create systemd user and directory
sudo useradd -r -s /bin/false myapp 2>/dev/null || true
sudo mkdir -p /opt/myapp
sudo chown myapp:myapp /opt/myapp

# Install the unit file
sudo cp systemd/myapp.service /etc/systemd/system/myapp.service
sudo systemctl daemon-reload
sudo systemctl enable myapp
```

**On the docker tab — add the key:**
```bash
sudo bash -c 'echo "YOUR_PUBLIC_KEY" >> /home/laborant/.ssh/authorized_keys && chmod 600 /home/laborant/.ssh/authorized_keys'
```

**On the kubernetes tab — create the service account and token:**
```bash
kubectl create serviceaccount jenkins-robot 2>/dev/null || true
kubectl create rolebinding jenkins-robot-binding --clusterrole=cluster-admin --serviceaccount=default:jenkins-robot 2>/dev/null || true
kubectl create token jenkins-robot
```
Copy the token (starts with `eyJ...`).

## Jenkins setup

1. Go to Jenkins → Manage Jenkins → Credentials → Global → Add Credentials
   - Kind: **Secret text**
   - Secret: paste the token from above
   - ID: `k8s-token`

2. Run the **Seed-Remote** job and paste this repo URL

3. Open the generated job and trigger a build

## What the pipeline does

| Stage | What happens |
|---|---|
| Install Dependencies | Installs Node.js 24 and runs `npm install` |
| Test | Runs `node --test` — fails the build if any test fails |
| Docker Build and Push | Builds multi-stage image, pushes to `ttl.sh` |
| Deploy to Target | Copies `index.js` + `node_modules` via scp, restarts systemd service |
| Deploy to Docker | SSH into docker VM, pulls image, runs container on port 4444 |
| Deploy to Kubernetes | Applies Deployment + Service + bare Pod, waits for ready |
| Health Check | Curls the pod IP from inside the cluster to confirm the app is serving |

## Verifying each target

**Target VM:**
```bash
# on the target tab
curl http://localhost:4444/
```

**Docker VM:**
```bash
# on the docker tab
curl http://localhost:4444/
```

**Kubernetes:**
```bash
# on the kubernetes tab
kubectl get pods
POD_IP=$(kubectl get pod -l app=myapp -o jsonpath='{.items[0].status.podIP}')
curl http://$POD_IP:4444/
```

Expected response from all three:
```json
{"name":"Hello","description":"World","url":"<host>"}
```

## Local run (no pipeline)

```bash
npm install
node index.js
# in another terminal
curl http://localhost:4444/
```

Run tests:
```bash
node --test
```
