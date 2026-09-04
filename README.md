# kyverno-poc

This repo contains Kyverno admission policies and test workloads for the Smaitic platform.
Kyverno is a Kubernetes policy engine. It automatically patches and validates pods at admission time based on annotations you put on your workloads.

---

## What is in this repo

| Path | What it is |
|---|---|
| [`helm-charts/kyverno-policies/`](helm-charts/kyverno-policies/) | Helm chart with all admission policies. Deploy this first. |
| [`helm-charts/workloads/`](helm-charts/workloads/) | Helm chart with test workloads (Deployment, StatefulSet, CronJob). |
| [`helmfile.yaml`](helmfile.yaml) | Deploys both charts in the correct order. |
| [`override.yaml`](override.yaml) | Override values for the workloads chart. |
| [`deployments/`](deployments/) | Standalone workload manifests for manual testing. |
| [`demo-invalid-pod.yaml`](demo-invalid-pod.yaml) | A pod that is missing a required field. Apply this to verify the validation policy blocks it. |

## Policy details

See [`helm-charts/kyverno-policies/README.md`](helm-charts/kyverno-policies/README.md) for a full explanation of every policy and the annotations that trigger them.
