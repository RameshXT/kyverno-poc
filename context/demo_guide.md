# Kyverno POC - Demo Script & Execution Guide

This guide provides a minimal, step-by-step walkthrough script and exact commands for presenting the Kyverno POC to Deepankar and the leadership team.

---

## Demo Pitch (30-Second Talking Point)
> *"Today, our Helm charts contain over 100+ lines of hardcoded boilerplate per microservice (initContainers, secrets parser, volumes, nodeSelectors, resources). This creates template clutter and noisy `helmfile diff` logs.*
> 
> *With Kyverno, app developers write clean, 10-line Pod manifests. Kyverno intercepts Pod creation in-flight at admission time, injects required governance settings, and leaves parent Helm releases 100% clean with **zero diff noise**."*

---

## Phase 1: Environment Setup

### 1. Label and Taint Minikube Node
Simulate production node pools and taints:
```bash
kubectl label node minikube demo.smaitic.com/role=poc-worker --overwrite
kubectl taint node minikube demo.smaitic.com/reserved-for=poc-worker:NoSchedule --overwrite
```

### 2. Create Target Namespace
```bash
kubectl create namespace policy
```

---

## Phase 2: Deploy Governance Policies

Deploy all Kyverno `ClusterPolicy` rules via Helmfile:
```bash
helmfile apply -l name=kyverno-policies
```

Verify all 7 policies are active and ready:
```bash
kubectl get clusterpolicy
```
*Key Point to Highlight: Policies are managed as a standard Helm release (`kyverno-policies`).*

> **Q&A Tip (If asked why `BACKGROUND` is `false`)**:  
> *"We set `background: false` to avoid continuous background audit scanning. In high-frequency CronJob environments, background scans create unnecessary DB and CPU churn. `background: false` ensures mutation runs live at admission time only."*

---

## Phase 3: Deploy Application Workloads

Deploy application workloads via Helmfile:
```bash
helmfile apply -l name=workloads
```

Check deployed resources in namespace `policy`:
```bash
kubectl get pods,deployments,cronjobs -n policy
```

---

## Phase 4: Live Proofs & Feature Demonstrations

### Proof 1: Position 0 Init Container Insertion
Show that `secrets-parser` was forced to index `0` ahead of existing init containers:
```bash
kubectl get pod demo-02-existing-init-ordering -n policy -o jsonpath='{.spec.initContainers[*].name}'
```
*Expected Output: `secrets-parser custom-app-init`*

### Proof 2: Full In-Flight Pod Mutation
Inspect all injected fields on the basic test pod:
```bash
kubectl get pod demo-01-basic-injection -n policy -o yaml
```
*Highlight to Deepankar*:
- `spec.initContainers[0]`: `secrets-parser`
- `spec.containers[0].command[0]`: `/scripts/app_launcher.sh`
- `spec.nodeSelector`: `demo.smaitic.com/role: poc-worker`
- `spec.tolerations`: `key: demo.smaitic.com/reserved-for`
- `spec.containers[0].resources`: `requests` (100m/256Mi) & `limits` (500m/512Mi)
- `spec.volumes`: `secrets-store-inline`, `shared-memory`, `init-scripts`

### Proof 3: Node Taint Isolation
Show that unannotated pods are repelled by node taints:
```bash
kubectl get pod demo-03-plain-unannotated -n policy
```
*Expected Status: `Pending` (unannotated pod lacks injected toleration).*

### Proof 4: Automated Migration Job Generation on Image Change
Update Deployment image tag:
```bash
kubectl set image deployment/poc-app poc-app=busybox:1.37 -n policy
```
Wait 3 seconds, then inspect generated Job:
```bash
kubectl get jobs -n policy
```
*Highlight to Deepankar: Kyverno auto-generated `poc-app-migration-<resourceVersion>` with zero Helm hooks and zero immutable field errors.*

### Proof 5: Admission Validation Rule (Deny Invalid Workload)
Try applying an annotated pod missing a command:
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
*Expected Result: `admission webhook "validate-command-defined" denied the request`.*

---

## Phase 5: Zero Helm Diff Noise Verification

Run `helmfile diff` to demonstrate zero drift reported on CronJob / Deployment objects:
```bash
helmfile diff
```
*Highlight to Deepankar: Pods are mutated in-flight at admission time. The CronJob object in etcd remains unmutated, so `helmfile diff` reports ZERO changes.*

---

## Phase 6: Side-by-Side Spec Comparison

Show saved comparison files in `output/`:
- Traditional Helm output: `output/helm-vizkit.yaml` (~130 lines of template boilerplate)
- Kyverno mutated output: `output/kyverno-vizkit.yaml` (Produced from 10-line Pod manifest)

---

## Phase 7: Complete Teardown & Reset

```bash
helmfile destroy
kubectl delete namespace policy --ignore-not-found
kubectl delete clusterpolicy --all --ignore-not-found
kubectl delete clusterrole kyverno-job-creator --ignore-not-found
kubectl delete clusterrolebinding kyverno-job-creator-binding --ignore-not-found
kubectl taint node minikube demo.smaitic.com/reserved-for=poc-worker:NoSchedule- || true
kubectl label node minikube demo.smaitic.com/role- || true
```
