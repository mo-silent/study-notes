# Dapr Troubleshooting Guide

## Resolving Invalid memory address or Nil Pointer Panic

This FQA provides the root cause analysis and solutions for the "nil pointer dereference" panic that may occur when using Dapr for local development.

### Symptom
When running a Dapr application, the daprd process exits unexpectedly with a panic. The error log indicates a runtime error: invalid memory address or nil pointer dereference originating from an OpenTelemetry module.

The following stack trace is captured when the panic occurs:

```shell
panic: runtime error: invalid memory address or nil pointer dereference [recovered]
        panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x2 addr=0x18 pc=0x1071210a0]

goroutine 300 [running]:
go.opentelemetry.io/otel/sdk/trace.(*recordingSpan).End.deferwrap1()
        /Users/runner/go/pkg/mod/go.opentelemetry.io/otel/sdk@v1.35.0/trace/span.go:467 +0x2c
go.opentelemetry.io/otel/sdk/trace.(*recordingSpan).End(0x140014f8960, {0x0, 0x0, 0x0?})
        /Users/runner/go/pkg/mod/go.opentelemetry.io/otel/sdk@v1.35.0/trace/span.go:506 +0xa04
panic({0x1096d7fc0?, 0x10e33ed70?})
        /Users/runner/hostedtoolcache/go/1.24.4/arm64/src/runtime/panic.go:792 +0x124
github.com/dapr/dapr/pkg/runtime/wfengine/state.LoadWorkflowState({0x10a5e01f8, 0x14001979ad0}, {0x0, 0x0}, {0x14000c43c20, 0x27}, {{0x16d4f6314, 0xc}, {0x14001532390, 0x2b}, ...})
        /Users/runner/work/dapr/dapr/pkg/runtime/wfengine/state/state.go:263 +0x100
... (remaining stack trace)
❌  The daprd process exited with error code: exit status 2
ℹ️  
terminated signal received: shutting down
❌  Error exiting Dapr: exit status 2
✅  Exited App successfully
```

### Root Cause Analysis
The stack trace shows the panic occurs within the LoadWorkflowState function, which is part of the Dapr Workflow engine. Features like Dapr Actors and Dapr Workflows are stateful and require a state store component to be configured to manage and persist their state.

The error arises because Dapr is attempting to load or manage an actor's state but cannot find a configured state store component. This failure within the state-loading operation leads to an unstable state, which subsequently causes a nil pointer panic when the associated OpenTelemetry trace span attempts to close.

This issue typically occurs if:

- The dapr init command was not run, which normally creates a default statestore.yaml file.
- The default `~/.dapr/components` directory was cleared or the statestore.yaml file was manually deleted.

### Solution

To resolve this issue, you must define a state store component for Dapr to use. Dapr building blocks that require state, such as Actors or Workflows, have a dependency on a state store component.

You can fix this by creating a component manifest file. By default, Dapr loads components from `~/.dapr/components/` for local development.

#### Step-by-Step Instructions
1. Create a component YAML file.
    Create a file named `statestore.yaml` in your components directory. For local development, the default directory is `~/.dapr/components/`.
    If you are using a custom directory for your resources, create the file there and ensure you specify the path using the --resources-path flag when running Dapr.

2. Add the component definition.
    The following example configures an in-memory state store, which is suitable for local development and testing. It is also correctly configured for use with Dapr Actors (actorStateStore: "true").
    `statestore.yaml` Example:

    ```yaml
    apiVersion: dapr.io/v1alpha1
    kind: Component
    metadata:
      name: statestore
    spec:
      type: state.in-memory
      version: v1
      metadata:
      - name: actorStateStore
        value: "true"
    ```

After creating this file, restart your Dapr application. The `daprd` sidecar will now be able to successfully load the required state store component, resolving the panic.