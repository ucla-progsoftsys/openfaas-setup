# OpenFaaS JVM Profile Artifact Demo

Local infra demo that proves pull-before-start and push-on-termination ordering for JVM profile artifacts, using OpenFaaS on Kubernetes and Redis as the artifact store.

Each function instance:
1. **Pulls** an artifact from Redis before `java` starts
2. Runs a minimal JVM HTTP handler
3. On **SIGTERM**, pushes an updated artifact to Redis before the process is killed

---

## Prerequisites

The following tools must be on your PATH: `docker`, `kubectl`, `helm`, `faas-cli`, `jq`, `redis-cli`. On macOS/Linux you also need `kind` (or `k3d`) to create a local cluster. On Windows, Docker Desktop's built-in Kubernetes is used instead (no kind needed).

### Windows (Scoop — run in PowerShell)

```powershell
# Install Scoop if you don't have it
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
irm get.scoop.sh | iex

# Install tools
scoop install helm jq redis
scoop bucket add extras
scoop install faas-cli
```

Install [Docker Desktop](https://docs.docker.com/desktop/install/windows-install/) separately and start it before proceeding. `kubectl` ships with Docker Desktop; if you need it standalone: `scoop install kubectl`.

Scoop installs to `%USERPROFILE%\scoop\shims`, which Git Bash doesn't include on PATH by default. Add it once so the tools are available in Git Bash:

```bash
echo 'export PATH="$PATH:$HOME/scoop/shims"' >> ~/.bashrc
source ~/.bashrc
```

### macOS (Homebrew)

```bash
brew install kind helm jq redis faas-cli
brew install --cask docker   # Docker Desktop
```

### Linux

```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# kind
curl -Lo kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x kind && sudo mv kind /usr/local/bin/

# helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# faas-cli
curl -sSL https://cli.openfaas.com | sudo sh

# jq and redis-cli
sudo apt-get install -y jq redis-tools   # Debian/Ubuntu
# sudo dnf install -y jq redis            # Fedora/RHEL
```

---

## Setup

### 1. Create a local cluster

**Windows:** kind fails on Windows Git Bash due to a cgroup issue. Use Docker Desktop's built-in Kubernetes instead: open Docker Desktop → Settings → Kubernetes → Enable Kubernetes → Apply & Restart. No CLI command needed.

**macOS / Linux:**

```bash
# kind
kind create cluster --name openfaas

# or k3d
k3d cluster create openfaas
```

### 2. Install OpenFaaS

```bash
kubectl apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml

helm repo add openfaas https://openfaas.github.io/faas-netes/
helm repo update
helm upgrade --install openfaas openfaas/openfaas \
  --namespace openfaas \
  --set functionNamespace=openfaas-fn \
  --set generateBasicAuth=true
```

Wait for the gateway to be ready before continuing:

```bash
kubectl rollout status deployment/gateway -n openfaas --timeout=120s
```

### 3. Deploy Redis

```bash
kubectl apply -f k8s/redis.yaml
```

### 4. Deploy the function

Start the gateway port-forward (leave it running in a separate terminal):

```bash
kubectl port-forward -n openfaas svc/gateway 8080:8080
```

Log in and deploy — run these from the repo root:

```bash
cd /path/to/openfaas-setup

PASSWORD=$(kubectl -n openfaas get secret basic-auth -o jsonpath='{.data.basic-auth-password}' | base64 --decode)
echo $PASSWORD | faas-cli login --username admin --password-stdin --gateway http://127.0.0.1:8080

faas-cli deploy -f stack.yml

# Patch the Deployment to add grace period + POD_UID downward API
kubectl patch deployment profile-fn -n openfaas-fn \
  --patch-file k8s/function-patch.yaml

# Wait for the patched pod to be ready before running the demo
kubectl rollout status deployment/profile-fn -n openfaas-fn --timeout=60s
```

`stack.yml` references a pre-built public image (`asundresh/profile-fn:latest`) so no Docker build is needed to run the demo.

**To modify the function and use your own image:**

```bash
docker build -t <your-dockerhub-username>/profile-fn:latest ./fn
docker push <your-dockerhub-username>/profile-fn:latest
```

Then update the `image:` line in `stack.yml` and redeploy. OpenFaaS CE only accepts public registry images, so the repository must be public on Docker Hub (or another public registry).

---

## Running the demo

You need two port-forwards running. The gateway one was started in step 4. Start the Redis one in another terminal:

```bash
kubectl port-forward -n openfaas svc/redis 6379:6379
```

Then run the test loop from the repo root:

```bash
TRIALS=20 bash scripts/test_loop.sh
```

The script exits `0` if every trial passes, `1` otherwise.

**Trial 1 note:** On a fresh cluster Redis has no artifact yet. The wrapper logs `ARTIFACT_MISSING`, seeds a default payload (counter=0), and the first push writes counter=1. This is expected and the script accounts for it — trial 1 passes like any other.

Expected output:

```
[...] PRE-LOOP: cycling pod to ensure trial 1 starts fresh...
[...] PRE-LOOP: waiting for replacement pod...
[...] Baseline counter=1 (read after pre-loop)

[...] ══ Trial 1/20 ══════
[...] Invoking profile-fn...
[...] pod_uid=abc123  artifact_hash=3f9a1c  runseq=2
[...] started key present: {"pod":"abc123","started_ms":...}
[...] Deleting pod profile-fn-xxxx (grace=20s)...
[...] Waiting for pod termination...
[...] terminated key present: {"pod":"abc123","terminated_ms":...}
[...] artifact counter=2 (expected 2)  last_writer=abc123
[...] Trial 1: PASS
...
════════════════════════════════════════════════════
 Results: 20 passed / 0 failed / 20 total
════════════════════════════════════════════════════
```

---

## Manual inspection

These require the Redis port-forward to be running (`kubectl port-forward -n openfaas svc/redis 6379:6379`):

```bash
# Last written artifact
redis-cli -h 127.0.0.1 -p 6379 GET artifact:profile-fn:v1

# Per-pod start record
redis-cli -h 127.0.0.1 -p 6379 GET started:<podUID>

# Per-pod termination record
redis-cli -h 127.0.0.1 -p 6379 GET terminated:<podUID>

# Run sequence counter
redis-cli -h 127.0.0.1 -p 6379 GET runseq:profile-fn:v1
```

---

## What the ordering proof looks like

| Log event | Source | Meaning |
|---|---|---|
| `PRE_PULL_DONE` | `entrypoint.sh` | Artifact on disk before JVM starts |
| `JAVA_STARTED` | `entrypoint.sh` + Java | JVM is up, artifact hash logged |
| `JAVA_SIGTERM` | Java shutdown hook | JVM received termination signal |
| `TERM_HANDLER_START` | `entrypoint.sh` | Wrapper caught SIGTERM, starting push |
| `POST_PUSH_DONE` | `entrypoint.sh` | Artifact pushed to Redis |

Each cold start reads the artifact written by the previous instance. The test script verifies `counter == prev + 1` on every trial — that monotonic increment is the chain-of-custody proof.

---

## Project layout

```
.
├── fn/
│   ├── Dockerfile          # multi-stage: Maven builder + JRE-alpine runtime
│   ├── entrypoint.sh       # wrapper: pull → start JVM → SIGTERM trap → push
│   ├── pom.xml
│   └── src/main/java/com/demo/ProfileFunction.java
├── k8s/
│   ├── redis.yaml          # Redis Deployment + Service (openfaas namespace)
│   └── function-patch.yaml # adds terminationGracePeriodSeconds + POD_UID env
├── scripts/
│   └── test_loop.sh        # N-trial invoke/kill/verify harness
└── stack.yml               # faas-cli deploy spec
```

---

## Phase 2 (next step)

Once this is stable, move the pull/push out of the app container into Kubernetes lifecycle primitives:

- **initContainer** pulls the artifact into a shared `emptyDir` at `/profiles`
- **preStop hook** or sidecar pushes the artifact on termination

See `k8s/function-patch.yaml` for the patch point — the initContainer and preStop entries go in the same `spec.template.spec` block.
