# kyverno-policies

Helm chart that packages all Kyverno admission policies for this project.
Deploy this chart and Kyverno will automatically enforce the rules below on every pod that enters the cluster.

---

## How it works

When a Deployment, StatefulSet, or CronJob is created, Kyverno intercepts it **before** it reaches the cluster.
Depending on the annotations on the pod template, Kyverno will:
- **Mutate**: add or patch fields automatically (volumes, init containers, resources, probes, node placement).
- **Validate**: block the pod if a required field is missing (e.g. `command` is empty).

---

## Policies in this chart

### 1. Secret Injection: [`clusterpolicy-secrets-manager.yaml`](templates/clusterpolicy-secrets-manager.yaml)

**Trigger annotation:** `smaitic.com/inject-secrets-poc: "<any-value>"`

What it does automatically to any annotated pod:
- Adds shared volumes (`shared-memory`, `init-scripts`, `secrets-store-inline`).
- Injects a `secrets-parser` init container at position 0 (handles both pods with and without existing init containers).
- Optionally prepends a launcher script to the main container command (`app-launcher` or `init-parser`) based on a second annotation.
- **Blocks** the pod if any container has no `command` defined. Kubernetes `command` replaces the Docker ENTRYPOINT, so omitting it breaks the injection wrapper.

---

### 2. Node Placement: [`clusterpolicy-node-placement.yaml`](templates/clusterpolicy-node-placement.yaml)

**Trigger annotation:** `smaitic.com/node-placement: "<value>"` (e.g. `staging`, `production`)

Patches `nodeSelector` and `tolerations` using whatever value is in the annotation.
No hardcoded values. The annotation drives everything.

---

### 3. Resource Requests & Limits: [`clusterpolicy-resources.yaml`](templates/clusterpolicy-resources.yaml)

**Trigger annotations:**
```
smaitic.com/resource-requests: "cpu=250m,memory=256Mi"
smaitic.com/resource-limits:   "cpu=500m,memory=512Mi"
```

Reads CPU and memory directly from these annotations and patches them into every container.
Works whether or not the pod already has a `resources` block.

---

### 4. Health Probes: [`clusterpolicy-health-probes.yaml`](templates/clusterpolicy-health-probes.yaml)

**Trigger annotation:** `smaitic.com/health-config: "<type>"`

| Type | Annotation value | Port and Path |
|---|---|---|
| Bridge app | `bridge` | Fixed. Port `9988`, paths `/actuator/health/liveness` and `/actuator/health/readiness` |
| Nginx static | `nginx-static` | Fixed. Port `8080`, path `/` |
| Custom (SSR, Next.js, etc.) | `custom` | From annotations: `smaitic.com/health-port` and `smaitic.com/health-path` |

---

### 5. Image Change Job Trigger: [`policy-image-change-job.yaml`](templates/policy-image-change-job.yaml)

> **DEFERRED. Entire file is commented out.**
> This policy is pending engineering team review and is not active.
>
> When re-enabling: also uncomment [`rbac-job-creator.yaml`](templates/rbac-job-creator.yaml). The policy needs that RBAC to create Jobs.

---

## Key config

| File | Purpose |
|---|---|
| [`values.yaml`](values.yaml) | Default annotation key and target namespace |
| [`Chart.yaml`](Chart.yaml) | Chart name and version |

The main annotation key (`smaitic.com/inject-secrets-poc`) is set in `values.yaml` under `annotationKey`.
All policy templates reference `{{ .Values.annotationKey }}` so you only change it in one place.
