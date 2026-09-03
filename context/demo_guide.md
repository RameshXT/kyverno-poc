# Kyverno POC — Demo Narration Script

This is the complete walkthrough script for the demo with Deepankar.
All commands are exact. All talking points are in simple English.

---

## Opening (30 seconds)

> "Deepankar, the problem today is every microservice Helm chart carries 100+ lines of repeated boilerplate —
> secrets init container, volumes, launcher command, nodeSelector, tolerations, resource limits, health probes.
> Every team copy-pastes it. When we change a standard, we update 30 charts.
>
> What I built here with Kyverno is: a developer writes a 10-line Pod manifest with one annotation.
> Kyverno intercepts it at admission time and injects all of that governance automatically.
> The Helm chart stays clean. helmfile diff stays clean. Zero drift."

---

## Part 1: Walk Through the Repo Structure

> "Let me show you how this is organized."

```
kyverno-poc/
├── helmfile.yaml                  ← manages both releases
├── override.yaml                  ← local environment overrides
├── helm-charts/
│   ├── kyverno-policies/          ← 9 ClusterPolicies as a Helm chart
│   ├── workloads/                 ← 10 test workloads as a Helm chart
│   └── boilerplate-app/           ← existing org boilerplate (reference)
└── context/
    └── demo_guide.md              ← this file
```

---

## Part 2: Show the Boilerplate (The Problem)

> "This is what we have today for every microservice."

```bash
cat helm-charts/boilerplate-app/templates/deployment.yaml
```

Point to Deepankar:
- 128 lines for one app
- Hardcoded: initContainers, volumes, volumeMounts, nodeSelector, tolerations, resources, health probes
- Every team maintains this. When governance changes, every chart must be updated manually.

> "Now let me show what I replaced this with."

```bash
cat helm-charts/workloads/templates/01-pod-basic-injection.yaml
```

Point: **13 lines. One annotation. That is it.**

---

## Part 3: Walk Through helmfile.yaml

> "We manage everything through helmfile. Two releases — policies and workloads."

```bash
cat helmfile.yaml
```

Point:
- `kyverno-policies` release → deploys all 9 ClusterPolicies into cluster
- `workloads` release → deploys test pods into namespace `policy`
- `override.yaml` → local environment values (image, annotation key, node domain)

```bash
cat override.yaml
```

> "In staging or production, this file changes per environment. The policies stay identical."

---

## Part 4: Walk Through the 9 Policies

> "Let me show you the governance layer. 9 policies, each with one concern."

```bash
ls helm-charts/kyverno-policies/templates/
```

Explain each in one line:

| File | What it does |
|---|---|
| `01-inject-volumes` | Injects 3 shared volumes + volumeMounts into every annotated Pod |
| `02-inject-secrets` | Forces `secrets-parser` initContainer at index 0 |
| `03-inject-command` | Injects launcher script at command[0] (app-launcher or init-parser) |
| `04-inject-node-placement` | Injects nodeSelector + toleration for node pool isolation |
| `05-inject-resources` | Injects CPU/Memory requests and limits |
| `06-inject-health-probes` | Injects liveness/readiness probes (backend port 9988 or frontend port 8080) |
| `07-validate-command` | Blocks any annotated Pod missing a container command |
| `08-trigger-job-on-image-change` | Auto-generates migration Job on Deployment image update |
| `09-kyverno-job-creator-rbac` | RBAC so Kyverno background controller can create Jobs |

> "All of these fire transparently at admission time. App developers never touch them."

---

## Part 5: Show the Annotation Contract

> "The developer's only responsibility is the annotation. Deepankar, this is the contract you mentioned."

```yaml
# Minimum — opts into all base governance (volumes, secrets, node, resources)
smaitic.com/inject-secrets-poc: "my-app"

# Health probe type (optional — backend or frontend)
smaitic.com/health-config: "backend"    # port 9988, /actuator/health/*
smaitic.com/health-config: "frontend"   # port 8080, /

# Launcher script (optional — which script to inject as command[0])
smaitic.com/launcher-script: "app-launcher"   # /scripts/app_launcher.sh
smaitic.com/launcher-script: "init-parser"    # /scripts/init_parser.sh
```

> "Annotation is readable. Any engineer looking at the Pod knows exactly what governance is applied."

---

## Part 6: Live Deploy

> "Let me deploy everything now, clean."

```bash
# Step 1: Prep node
kubectl label node minikube demo.smaitic.com/role=poc-worker --overwrite
kubectl taint node minikube demo.smaitic.com/reserved-for=poc-worker:NoSchedule --overwrite

# Step 2: Namespace
kubectl create namespace policy

# Step 3: Deploy policies
helmfile apply -l name=kyverno-policies
```

Then verify:
```bash
kubectl get clusterpolicy
```

Expected — 8 policies, all `READY: True`:
```
NAME                          ADMISSION   BACKGROUND   READY
inject-command-prefix         true        false        True
inject-health-probes          true        false        True
inject-node-placement         true        false        True
inject-resources              true        false        True
inject-secrets-parser         true        false        True
inject-volumes                true        false        True
trigger-job-on-image-change   true        false        True
validate-command-defined      true        false        True
```

> "background: false means Kyverno only fires at admission time, not background scans.
> In high-frequency CronJob environments this avoids unnecessary cluster churn."

```bash
# Step 4: Deploy workloads
helmfile apply -l name=workloads
```

Watch pods come up:
```bash
kubectl get pods -n policy
```

---

## Part 7: Live Proofs

### Proof 1 — Full mutation on basic pod

> "This pod was created from a 13-line manifest. Let me show what Kyverno injected."

```bash
kubectl get pod demo-01-basic-injection -n policy -o yaml | yq '.spec'
```

Point to each injected field:
- `initContainers[0].name: secrets-parser` — injected, not in original
- `containers[0].command[0]: /scripts/app_launcher.sh` — prepended by Kyverno
- `nodeSelector: demo.smaitic.com/role: poc-worker` — injected
- `tolerations: demo.smaitic.com/reserved-for` — injected
- `resources.requests/limits` — injected
- `volumes: secrets-store-inline, shared-memory, init-scripts` — injected

---

### Proof 2 — secrets-parser forced to index 0

> "This pod already had a custom init container defined. Kyverno still forced secrets-parser to position 0."

```bash
kubectl get pod demo-02-existing-init-ordering -n policy \
  -o jsonpath='{.spec.initContainers[*].name}'
```

Expected: `secrets-parser custom-app-init`

---

### Proof 3 — Unannotated pod gets no injection, stays Pending

> "No annotation — no policies fire. The pod gets no toleration, so it can't schedule on the tainted node."

```bash
kubectl get pod demo-03-plain-unannotated -n policy
```

Expected: `Pending`

---

### Proof 4 — Health probe injection by annotation

> "Backend pod gets actuator endpoints. Frontend pod gets port 8080 slash root. Purely annotation-driven."

```bash
kubectl get pod demo-04-backend-probes -n policy \
  -o jsonpath='{.spec.containers[0].livenessProbe}' | python3 -m json.tool

kubectl get pod demo-05-frontend-probes -n policy \
  -o jsonpath='{.spec.containers[0].livenessProbe}' | python3 -m json.tool
```

---

### Proof 5 — Validation rule blocks bad pods

> "Any annotated pod without a command is rejected at the gate."

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: demo-invalid-pod
  namespace: policy
  annotations:
    smaitic.com/inject-secrets-poc: "demo-invalid-pod"
spec:
  containers:
    - name: app
      image: busybox:latest
EOF
```

Expected: admission webhook denied

---

### Proof 6 — Image change triggers migration Job

> "When a Deployment image changes, Kyverno auto-generates a migration Job. No Helm hooks needed."

```bash
kubectl set image deployment/poc-app poc-app=busybox:1.37 -n policy
```

Wait 3 seconds:
```bash
kubectl get jobs -n policy
```

Expected: `poc-app-migration-<resourceVersion>` Job created automatically.

---

### Proof 7 — helmfile diff is clean (zero noise)

> "Kyverno mutates at pod admission time only. The CronJob object in etcd is never touched.
> So helmfile diff sees zero changes — no boilerplate noise in GitOps pipelines."

```bash
helmfile diff
```

Expected: no changes reported.

---

## Part 8: Side-by-Side Comparison

> "Deepankar, this is the final proof. Same rendered spec. Completely different source."

```bash
# Boilerplate Helm — 128 lines
helm template boilerplate-app helm-charts/boilerplate-app > output/live-helm.yaml

# Kyverno mutated pod — from 13-line manifest
kubectl get pod demo-01-basic-injection -n policy -o yaml > output/live-kyverno.yaml
```

Open both files and compare `spec` section field by field.

Key matches:
- `initContainers[0].name: secrets-parser` ✅
- `command[0]: /scripts/app_launcher.sh` ✅
- `nodeSelector key: demo.smaitic.com/role` ✅
- `tolerations key: demo.smaitic.com/reserved-for` ✅
- `resources.requests: cpu 100m, memory 256Mi` ✅
- `volumes: secrets-store-inline, shared-memory, init-scripts` ✅

> "90% Helm template reduction. Identical runtime spec. Zero developer overhead."

---

## Part 9: Teardown

```bash
helmfile destroy
kubectl delete namespace policy --ignore-not-found
kubectl delete clusterpolicy --all --ignore-not-found
kubectl delete clusterrole kyverno-job-creator --ignore-not-found
kubectl delete clusterrolebinding kyverno-job-creator-binding --ignore-not-found
kubectl taint node minikube demo.smaitic.com/reserved-for=poc-worker:NoSchedule- || true
kubectl label node minikube demo.smaitic.com/role- || true
```

---

## Anticipated Questions & Answers

**Q: What happens if Kyverno is down?**
> Depends on failurePolicy — `Fail` means pods are blocked (safe), `Ignore` means pods are admitted without mutation (available). We need to decide this before production. (Task 13)

**Q: Why background: false?**
> Avoids continuous background audit scanning. In high-frequency CronJob environments (every 1-3 minutes), background scans create unnecessary CPU and etcd churn. Mutation fires only at admission time.

**Q: Does this work for Deployments and CronJobs too?**
> Yes. Kyverno intercepts the Pod created by any parent resource — Deployment, CronJob, Job. The parent object in etcd is never mutated, so helmfile diff stays clean.

**Q: How do we roll this out to staging?**
> Install Kyverno via its official Helm chart. Deploy the kyverno-policies chart. Teams add the annotation to their workloads. No other change needed.

**Q: What about teams who don't want injection?**
> Simple — don't add the annotation. Unannotated pods are completely untouched. Opt-in model.

---

## Quick Reference Commands Cheat Sheet (Demo Shortcuts)

### 1. Boilerplate vs Kyverno Spec Diff (Requirement R8 Proof)
```bash
# Save boilerplate Helm rendered output (128 lines)
helm template boilerplate-app helm-charts/boilerplate-app > output/live-helm.yaml

# Save Kyverno live mutated Pod spec
kubectl get pod demo-01-basic-injection -n policy -o yaml > output/live-kyverno.yaml

# Compare side-by-side in VS Code
code --diff output/live-helm.yaml output/live-kyverno.yaml
```

### 2. Deploy Everything (Policies + Workloads)
```bash
# Deploy policies
helmfile apply -l name=kyverno-policies

# Deploy workloads
helmfile apply -l name=workloads
```

### 3. Key Verification Commands
```bash
# Verify 8 ClusterPolicies active
kubectl get clusterpolicy

# Verify all Pods running in namespace 'policy'
kubectl get pods -n policy

# Inspect basic mutated pod spec
kubectl get pod demo-01-basic-injection -n policy -o yaml | yq '.spec'

# Verify init container ordering (secrets-parser at index 0)
kubectl get pod demo-02-existing-init-ordering -n policy -o jsonpath='{.spec.initContainers[*].name}'

# Verify zero Helm diff noise (Requirement R10)
helmfile diff
```

### 4. Trigger Image Change Migration Job (Requirement R9)
```bash
# Trigger live Deployment image update
kubectl set image deployment/poc-app poc-app=busybox:1.37 -n policy

# Verify auto-generated migration Job
kubectl get jobs,pods -n policy
```

### 5. Full Environment Reset / Teardown
```bash
helmfile destroy && \
kubectl delete namespace policy --ignore-not-found && \
kubectl delete clusterpolicy --all --ignore-not-found && \
kubectl delete clusterrole kyverno-job-creator --ignore-not-found && \
kubectl delete clusterrolebinding kyverno-job-creator-binding --ignore-not-found && \
kubectl taint node minikube demo.smaitic.com/reserved-for=poc-worker:NoSchedule- || true && \
kubectl label node minikube demo.smaitic.com/role- || true
```

