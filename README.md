 Chaos Engineering - Testing System Resilience

## Table of Contents

- [Scenario: Production Under Stress](#scenario-production-under-stress)
- [Why Chaos Engineering Matters for SRE](#why-chaos-engineering-matters-for-sre)
- [Learning Objectives](#learning-objectives)
- [Prerequisites and GitHub Repo Setup](#prerequisites-and-github-repo-setup)
- [Phase 1: Deploy Target Application](#phase-1-deploy-target-application)
- [Phase 2: Install Argo Workflows and LitmusChaos](#phase-2-install-argo-workflows-and-litmuschaos)
- [Phase 3: Basic Chaos Experiments (Workflow-Based)](#phase-3-basic-chaos-experiments-workflow-based)
- [Phase 4: Rollback Under Chaos](#phase-4-rollback-under-chaos)
- [Phase 5: Automated Stress Testing](#phase-5-automated-stress-testing)
- [Phase 6: Measuring Chaos Impact](#phase-6-measuring-chaos-impact)
- [Cleanup](#cleanup)
- [Final Thoughts](#final-thoughts)

---

## Scenario: Production Under Stress

It's Black Friday, and your e-commerce platform is under immense pressure, with traffic ten times higher than normal. To make matters worse, you need to deploy a critical security patch right in the middle of this surge. However, your production environment isn't behaving as expected. You're seeing network latency spikes disrupting service communication and pods unexpectedly restarting.

As the SRE on call, you have a critical mission. First, you must **deploy the security patch** despite the unstable environment. You also need to be prepared to **safely roll back** if the deployment fails, all while **maintaining service availability** for your customers. Throughout this process, you must also **measure the impact** of these challenging conditions on your recovery procedures. The ultimate question is this: Will your deployment and rollback procedures work when production is already under so much stress?

---

## Why Chaos Engineering Matters for SRE

In previous exercises, you mastered deployment automation and rollback procedures under **controlled and ideal conditions**. But real-world production environments are rarely ideal. This is where **Chaos Engineering** comes in. By intentionally injecting controlled failures, Chaos Engineering helps SREs validate their assumptions about how a system behaves under pressure. It's a proactive way to discover weaknesses before they lead to real outages. This practice builds confidence in your recovery procedures, allowing you to measure the true Mean Time to Recovery (MTTR) under realistic failure conditions.

---

## Learning Objectives

By completing this exercise, you will gain practical experience with modern Cloud-Native Chaos Engineering using LitmusChaos 3.x and Argo Workflows. You'll begin by **installing the complete chaos engineering stack** in your Kubernetes environment. From there, you will **define chaos experiments as workflow orchestrations** using Argo Workflows—the production standard for automated chaos testing. You will then **execute automated rollbacks** while chaos experiments actively stress your cluster. Finally, you will **measure MTTR** to quantify how instability affects your ability to recover.

**Key Technical Skills:**
- Install and configure Argo Workflows for chaos orchestration
- Deploy LitmusChaos operator and chaos experiments
- Create declarative chaos workflows using Argo CRDs
- Implement RBAC for chaos experiment execution
- Measure and analyze Mean Time to Recovery (MTTR) under failure conditions
- Version control chaos experiments as infrastructure-as-code

---

## Prerequisites and GitHub Repo Setup

### Prerequisites

- A running Kubernetes cluster (Minikube, kind, or cloud-based)
- `kubectl` configured and working
- Python 3 installed
- Basic understanding of Kubernetes deployments
- Familiarity with YAML manifests

### Create Your GitHub Repository

1. **Create a new repository:**
   - Visit https://github.com/new
   - Repository name: `sre-chaos-engineering`
   - Visibility: Public
   - Leave "Initialize with README" unchecked
   - Click **Create repository**

2. **Initialize locally:**

```bash
mkdir -p ~/Projects/sre-chaos-engineering
cd ~/Projects/sre-chaos-engineering
git init
git branch -M main
_++++++```

3. **Link to remote** (replace YOUR-USERNAME):

```bash
git remote add origin https://github.com/YOUR-USERNAME/sre-chaos-engineering.git
```

4. **Create initial structure:**

```bash
# Create directory structure
mkdir -p manifests/stable manifests/unstable manifests/chaos scripts

# Create initial README
echo "# SRE Chaos Engineering Lab" > README.md
echo "Testing deployment resilience under controlled failure conditions" >> README.md

git add README.md
git commit -m "Initial commit: chaos engineering lab setup"
git push -u origin main
```

---

## Phase 1: Deploy Target Application

**What we're building:** A three-replica nginx deployment that will serve as our "production application." We need something to break during chaos experiments.

**Why two versions?** We're creating both a stable version (works correctly) and an unstable version (intentionally broken) so we can practice rollback procedures under stress.

### Step 1: Create Application Namespace

**What this does:** Creates an isolated namespace called `chaos-app` where our application will live. This separation mirrors production practices where applications run in dedicated namespaces rather than the default namespace.

```bash
# Create dedicated namespace for the target application
kubectl create namespace chaos-app

# Verify namespace creation
kubectl get namespace chaos-app
```

**Expected output:**
```
namespace/chaos-app created
NAME        STATUS   AGE
chaos-app   Active   1s
```

**What just happened:** Kubernetes created a new namespace. Think of it as a separate "room" in your cluster where resources are isolated from other namespaces.

---

### Step 2: Create Stable Application Manifests

**What this does:** Creates three YAML files that define your application: deployment (the pods), service (networking), and configmap (web content). These files don't deploy anything yet—they're just text files on your computer.

```bash
# Create stable deployment
cat > manifests/stable/deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chaos-target-app
  namespace: chaos-app
  labels:
    app: chaos-target-app
    version: stable
spec:
  replicas: 3
  selector:
    matchLabels:
      app: chaos-target-app
  template:
    metadata:
      labels:
        app: chaos-target-app
        version: stable
    spec:
      containers:
      - name: app
        image: nginx:1.21-alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: content
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
      volumes:
      - name: content
        configMap:
          name: app-content
EOF

# Create service
cat > manifests/stable/service.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: chaos-target-service
  namespace: chaos-app
  labels:
    app: chaos-target-app
spec:
  selector:
    app: chaos-target-app
  ports:
  - port: 80
    targetPort: 80
  type: NodePort
EOF

# Create content ConfigMap
cat > manifests/stable/configmap.yaml <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-content
  namespace: chaos-app
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>Chaos Engineering Target</title>
        <style>
            body { font-family: Arial, sans-serif; text-align: center; padding: 50px; background: #e8f5e8; }
            .status { color: #2d5a3d; font-size: 1.5em; margin: 20px 0; }
        </style>
    </head>
    <body>
        <h1>Chaos Engineering Target Application</h1>
        <div class="status">Status: STABLE</div>
        <p>Timestamp: <span id="time"></span></p>
        <script>document.getElementById('time').textContent = new Date().toLocaleString();</script>
    </body>
    </html>
EOF
```

**What you just created:**
- `deployment.yaml` - Defines 3 nginx pods with health checks and resource limits
- `service.yaml` - Creates a network endpoint to access the pods
- `configmap.yaml` - Contains the HTML content the nginx pods will serve

**Key details to notice:**
- `replicas: 3` means we'll have 3 copies of the application running
- `namespace: chaos-app` places everything in our dedicated namespace
- `livenessProbe` and `readinessProbe` help Kubernetes detect if pods are healthy
- The label `app: chaos-target-app` is how chaos experiments will find these pods

---

### Step 3: Create Unstable Application (For Rollback Testing)

**What this does:** Creates manifests for a broken version of your application. The image tag `nonexistent-tag` doesn't exist, so pods will fail with `ImagePullBackOff`. This simulates a bad production deployment.

```bash
# Create unstable deployment with broken image
cat > manifests/unstable/deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chaos-target-app
  namespace: chaos-app
  labels:
    app: chaos-target-app
    version: unstable
spec:
  replicas: 3
  strategy:
    type: Recreate  # ← Changed from RollingUpdate to force all pods down
  selector:
    matchLabels:
      app: chaos-target-app
  template:
    metadata:
      labels:
        app: chaos-target-app
        version: unstable
    spec:
      containers:
      - name: app
        image: nginx:nonexistent-tag  # This will cause ImagePullBackOff
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
        volumeMounts:
        - name: content
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
      volumes:
      - name: content
        configMap:
          name: app-content-unstable
EOF

# Copy service for unstable version
cp manifests/stable/service.yaml manifests/unstable/service.yaml

# Create unstable content ConfigMap
cat > manifests/unstable/configmap.yaml <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-content-unstable
  namespace: chaos-app
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>Chaos Engineering Target - UNSTABLE</title>
    </head>
    <body>
        <h1>Chaos Engineering Target - BROKEN</h1>
    </body>
    </html>
EOF
```

**Why create a broken version?** In Phase 4, we'll deploy this broken version and measure how long it takes to roll back to the stable version—both with and without chaos attacking the cluster.

---

### Step 4: Deploy and Verify Stable Application

**What this does:** Actually deploys your application to Kubernetes. Until now, you've just been creating files. This command sends those files to the Kubernetes API server.

**Important:** Open TWO terminal windows for this section. Keep both visible.

**Terminal 1 - Observation:**
```bash
# Watch pods being created in real-time
watch kubectl get pods -n chaos-app
```

**Terminal 2 - Action:**
```bash
# Deploy stable version
kubectl apply -f manifests/stable/

# Wait for deployment to complete
kubectl rollout status deployment/chaos-target-app -n chaos-app
```

**What you'll see in Terminal 1:**
```
NAME                               READY   STATUS              RESTARTS   AGE
chaos-target-app-7d9f8c5b4-abc12   0/1     ContainerCreating   0          2s
chaos-target-app-7d9f8c5b4-def34   0/1     ContainerCreating   0          2s
chaos-target-app-7d9f8c5b4-ghi56   0/1     ContainerCreating   0          2s
```

Then after ~10 seconds:
```
NAME                               READY   STATUS    RESTARTS   AGE
chaos-target-app-7d9f8c5b4-abc12   1/1     Running   0          15s
chaos-target-app-7d9f8c5b4-def34   1/1     Running   0          15s
chaos-target-app-7d9f8c5b4-ghi56   1/1     Running   0          15s
```

**What you'll see in Terminal 2:**
```
configmap/app-content created
deployment.apps/chaos-target-app created
service/chaos-target-service created
deployment "chaos-target-app" successfully rolled out
```

**What just happened:**
1. Kubernetes created the ConfigMap with your HTML content
2. Kubernetes created the Deployment, which created a ReplicaSet, which created 3 Pods
3. Each pod pulled the nginx:1.21-alpine image from Docker Hub
4. Kubernetes started the containers and ran health checks
5. Once healthy, the pods were marked as "Ready"
6. Kubernetes created a Service to route traffic to the pods

**Verify everything is working:**
```bash
# See all resources in chaos-app namespace
kubectl get all -n chaos-app

# Test the application
kubectl get svc chaos-target-service -n chaos-app
minikube service chaos-target-service -n chaos-app --url
```

Visit the URL in your browser. You should see: "Status: STABLE"

**Press Ctrl+C in Terminal 1 to stop watching pods.**

---

## Phase 2: Install Argo Workflows and LitmusChaos

**What we're installing:** Two systems that work together:
1. **Argo Workflows** - Orchestrates multi-step chaos experiments (like a conductor leading an orchestra)
2. **LitmusChaos** - Actually executes the chaos (the musicians playing the instruments)

**Why both?** Modern chaos engineering doesn't just "break stuff randomly." It follows a workflow: install experiment → execute chaos → collect results → cleanup. Argo manages this workflow; Litmus executes the chaos.

### Step 1: Install Argo Workflows

**What this does:** Installs Argo Workflows, which will orchestrate our chaos experiments. This is similar to installing a CI/CD system like Jenkins or GitHub Actions.

**Terminal 1 - Observation:**
```bash
# Watch Argo pods being created
watch kubectl get pods -n argo
```

**Terminal 2 - Action:**
```bash
# Create namespace for Argo
kubectl create namespace argo

# Install Argo Workflows
kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/download/v3.5.11/install.yaml

# Wait for Argo to be ready (this may take 1-2 minutes)
echo "Waiting for Argo Workflows to be ready..."
kubectl wait --for=condition=Ready pods --all -n argo --timeout=300s
```

**What you'll see in Terminal 1:**

Initially:
```
NAME                                   READY   STATUS              RESTARTS   AGE
argo-server-667f6fdf45-xxxxx           0/1     ContainerCreating   0          5s
workflow-controller-5d5c84f6c8-xxxxx   0/1     ContainerCreating   0          5s
```

After 30-60 seconds:
```
NAME                                   READY   STATUS    RESTARTS   AGE
argo-server-667f6fdf45-xxxxx           1/1     Running   0          45s
workflow-controller-5d5c84f6c8-xxxxx   1/1     Running   0          45s
```

**What you'll see in Terminal 2:**
```
namespace/argo created
customresourcedefinition.apiextensions.k8s.io/clusterworkflowtemplates.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/cronworkflows.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/workfloweventbindings.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/workflows.argoproj.io created
...
pod/argo-server-667f6fdf45-xxxxx condition met
pod/workflow-controller-5d5c84f6c8-xxxxx condition met
```

**What just happened:**
1. Created `argo` namespace
2. Installed Custom Resource Definitions (CRDs) for Workflow, CronWorkflow, etc.
3. Deployed two pods: `argo-server` (web UI) and `workflow-controller` (executes workflows)
4. Argo is now watching for Workflow resources in your cluster

**Press Ctrl+C in Terminal 1 to stop watching.**

---

### Step 2: Install LitmusChaos Operator

**What this does:** Installs the LitmusChaos operator, which watches for ChaosEngine resources and executes chaos experiments.
