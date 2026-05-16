# Tool Execution in kagent

This document describes how tools are registered, dispatched, and executed within the kagent controller and agent runtime.

## Overview

Tools in kagent are wrappers around Kubernetes operations, HTTP calls, or custom logic that agents invoke during task execution. Each tool is defined as a CRD (`Tool`) and referenced by an `Agent` spec.

## Tool CRD Structure

```yaml
apiVersion: kagent.dev/v1alpha1
kind: Tool
metadata:
  name: kubectl-get
  namespace: kagent
spec:
  description: "Run kubectl get for a given resource type"
  runtime: exec
  exec:
    command: ["kubectl", "get"]
    args: ["{{ .resourceType }}", "-n", "{{ .namespace }}"]
  inputSchema:
    type: object
    properties:
      resourceType:
        type: string
      namespace:
        type: string
    required: ["resourceType"]
```

## Tool Runtimes

### `exec` Runtime

Runs a subprocess command on the controller pod. Arguments support Go template interpolation from the tool call input.

- **Security**: Commands are allowlisted; arbitrary shell is not permitted.
- **Timeout**: Defaults to 30s; configurable via `spec.exec.timeoutSeconds`.
- **Output**: stdout is returned as the tool result; non-zero exit codes surface as errors.

### `http` Runtime

Makes an HTTP request to an internal or external endpoint.

```yaml
spec:
  runtime: http
  http:
    url: "http://metrics-service.monitoring.svc/query"
    method: POST
    bodyTemplate: '{"query": "{{ .promql }}"}'
    headers:
      Content-Type: application/json
```

### `builtin` Runtime

Invokes a Go function registered in the controller binary. Builtins have direct access to the Kubernetes client and are used for operations like applying manifests or reading secrets.

Registration (in `pkg/tools/registry.go`):

```go
registry.Register("apply-manifest", tools.ApplyManifest)
```

## Execution Flow

```
Agent LLM response
  └─► ToolCall extracted (name + JSON args)
        └─► ToolDispatcher.Dispatch(ctx, toolCall)
              ├─► Lookup Tool CR by name
              ├─► Validate args against inputSchema
              ├─► Select runtime handler
              └─► Execute → ToolResult
                    └─► Appended to conversation history
```

## Error Handling

| Scenario | Behaviour |
|---|---|
| Tool CR not found | Returns error result; agent receives "tool not available" message |
| Schema validation failure | Returns structured validation error; agent may retry with corrected args |
| Exec timeout | Process killed; error surfaced to agent |
| HTTP non-2xx | Response body included in error for agent context |
| Builtin panic | Recovered; logged as `ERROR`; agent receives generic failure |

## Tool Result Format

All runtimes return a `ToolResult` struct:

```go
type ToolResult struct {
    ToolCallID string `json:"tool_call_id"`
    Content    string `json:"content"`
    IsError    bool   `json:"is_error,omitempty"`
}
```

The `Content` field is plain text or JSON string passed back into the LLM context window.

## Adding a New Tool

1. Define the `Tool` CR in `config/samples/tools/`.
2. If using `builtin` runtime, implement the handler in `pkg/tools/` and register it in `pkg/tools/registry.go`.
3. Add the tool name to the relevant `Agent` CR under `spec.tools[]`.
4. Write an e2e test in `test/e2e/` that creates an agent referencing the new tool and validates the output.

## Common Pitfalls

- **Template escaping**: JSON args containing `{{` must be escaped as `{{"{{"}}` in Go templates.
- **RBAC**: `exec` tools running `kubectl` rely on the controller's ServiceAccount; ensure the necessary ClusterRole rules are present.
- **Idempotency**: Builtin tools should be idempotent where possible since agents may retry on transient failures.
- **Large outputs**: Tool results exceeding ~8k tokens are truncated before being appended to context. Use pagination args where applicable.
