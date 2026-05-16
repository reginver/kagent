# Agent Reconciliation Loop

This document describes the Kubernetes controller reconciliation logic for `Agent` resources in kagent.

## Overview

The `AgentReconciler` watches `Agent` CRDs and ensures the desired state (defined in the spec) matches the actual state (running AutoGen agents, associated ConfigMaps, Services, and Deployments).

## Reconciliation Flow

```
Agent CR created/updated
        │
        ▼
  Fetch Agent object
        │
        ├─── Not found → Remove finalizer, clean up resources
        │
        ▼
  Add finalizer if missing
        │
        ▼
  Validate spec (model config, tools, memory)
        │
        ├─── Invalid → Update status condition: Ready=False, reason=InvalidSpec
        │
        ▼
  Reconcile ModelConfig reference
        │
        ├─── Missing → Requeue after 30s, status: Ready=False, reason=ModelConfigNotFound
        │
        ▼
  Reconcile Tool references
        │
        ├─── Missing tool → status: Ready=False, reason=ToolNotFound
        │
        ▼
  Reconcile ConfigMap (agent config JSON)
        │
        ▼
  Reconcile Deployment
        │
        ▼
  Reconcile Service
        │
        ▼
  Update status: Ready=True, observedGeneration
```

## Key Reconciler Functions

### `reconcileConfigMap`

Builds a JSON blob from the Agent spec (system prompt, tools list, model config ref) and stores it in a ConfigMap named `<agent-name>-config`. The agent container mounts this at `/etc/kagent/config.json`.

**Triggers re-reconcile when:**
- Agent spec changes (generation bump)
- Referenced ModelConfig changes (owner reference watch)
- Referenced Tool CRD changes

### `reconcileDeployment`

Ensures a `Deployment` exists with:
- Image pulled from `agent.spec.image` (defaults to `ghcr.io/kagent-dev/kagent-agent:latest`)
- EnvFrom referencing the ModelConfig secret
- VolumeMount for the config ConfigMap
- Resource limits from `agent.spec.resources`
- Liveness/readiness probes on `/healthz` port `8080`

**Scaling:** Defaults to 1 replica. HPA is not managed by this reconciler.

### `reconcileService`

Creates a `ClusterIP` Service exposing port `8080` (HTTP/gRPC multiplexed). The Service is labelled with `kagent.dev/agent: <name>` for discovery by the `AgentRouter`.

## Status Conditions

| Condition Type | Reason | Description |
|---|---|---|
| `Ready` | `Reconciling` | Initial state, reconcile in progress |
| `Ready` | `InvalidSpec` | Spec validation failed |
| `Ready` | `ModelConfigNotFound` | Referenced ModelConfig CR missing |
| `Ready` | `ToolNotFound` | One or more referenced Tool CRs missing |
| `Ready` | `DeploymentUnavailable` | Deployment pods not ready |
| `Ready` | `AgentReady` | All resources healthy |

## Finalizer

Finalizer: `kagent.dev/agent-finalizer`

On deletion, the reconciler:
1. Removes the Deployment (cascade deletes ReplicaSet/Pods)
2. Removes the Service
3. Removes the ConfigMap
4. Removes the finalizer to allow CR deletion

Orphan prevention: all owned resources have `ownerReference` set to the Agent CR so garbage collection handles partial failures.

## Requeue Strategy

| Situation | Requeue After |
|---|---|
| ModelConfig not found | 30s |
| Tool not found | 30s |
| Deployment not yet available | 15s |
| Transient API error | exponential backoff (controller-runtime default) |
| Successful reconcile | none (watch-driven) |

## Common Issues

### Agent stuck in `Reconciling`

```bash
kubectl describe agent <name> -n <namespace>
# Check Events and Status.Conditions

kubectl logs -n kagent-system deployment/kagent-controller-manager -c manager | grep <agent-name>
```

### ConfigMap not updating after spec change

Ensure the Agent's `metadata.generation` incremented. If the spec change only touched `metadata.annotations`, generation does not bump and reconcile won't re-run the configmap diff.

Force reconcile:
```bash
kubectl annotate agent <name> kagent.dev/force-sync=$(date +%s) --overwrite
```

### Deployment image not updating

The reconciler uses `reflect.DeepEqual` on the container spec. If the image tag is `latest` and the digest changed, the reconciler will **not** detect a diff. Pin to a digest or use a versioned tag.

## Testing

Unit tests live in `pkg/controller/agent_controller_test.go` and use `envtest` with a real API server.

Key test cases:
- `TestReconcileNewAgent` — happy path, all resources created
- `TestReconcileAgentMissingModelConfig` — status set to not ready, requeue
- `TestReconcileAgentDeletion` — finalizer removed, resources cleaned up
- `TestReconcileAgentSpecUpdate` — ConfigMap updated on spec change

Run:
```bash
go test ./pkg/controller/... -v -run TestAgent
```
