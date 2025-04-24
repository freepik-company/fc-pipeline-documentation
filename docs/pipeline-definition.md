# Pipeline Definition Schema

This document provides the complete schema for defining pipelines with the pipeline-generator.

## Full Schema

Below is the full schema for defining pipelines:

```yaml
pipelines:
  # Pipeline Types
  branches:                           # Type of pipeline: branches, tags, pull_requests, or custom
  - name: string                      # Name of the pipeline
    regex: string                     # Regular expression to match branch or tag name
    serviceAccount: string            # K8s service account with permissions to launch 
                                      # new pipelines and deploy manifests
    nextPipeline:                     # Configuration for subsequent pipelines (default: {})
      success: string                 # Custom pipeline to launch when this succeeds
                                      # (must match regex of target pipeline)
      failed: string                  # Custom pipeline to launch when this fails
      customParamsExtra: string       # Additional parameters to pass to next pipeline
                                      # Can use results from previous steps
      burnupEnabled: boolean          # Enable/disable burnup in the step that
                                      # launches next pipelines (default: true)
      burndownEnabled: boolean        # Enable/disable burndown in the step that
                                      # launches next pipelines (default: true)

    steps:                            # List of steps to run in the pipeline
    - name: string                    # Name of the step
      timeout: string                 # Timeout for the step (e.g., "1h30m0s")
      description: string             # Description of the step
      runAfter:                       # List of steps that must complete before this one
      - step-name
      burnupEnabled: string           # "true" or "false" - controls artifacts/cache download
      burndownEnabled: string         # "true" or "false" - controls artifacts/cache upload
      image: string                   # Container image for this step
      imagePullPolicy: string         # Image download policy (since 1.1.0)
      resources:                      # Container resource requirements (since 1.2.0)
        limits:                       # Maximum resource allocation
          cpu: string/number
          memory: string
        requests:                     # Requested resource allocation
          cpu: string/number
          memory: string
      env:                            # Environment variables (Kubernetes format)
      - name: string
        value: string
        # Or valueFrom with secretKeyRef, configMapKeyRef, etc.
      params:                         # Custom parameters for scripts/args
      - name: string
        value: string
      volumes:                        # Custom volumes (mounted at /mnt/[volumeName])
      - name: string
        # Volume source configuration
      command:                        # Command to run (list of strings)
      - string
      args:                           # Arguments for command (list of strings)
      - string
      script: string                  # Script to run (alternative to command/args)
      artifacts:                      # Paths to artifacts to save from this step
      - path/to/artifact
      cache:                          # Paths to cache for future pipeline runs
      - path/to/cache
      sidecars:                       # Containers running alongside the step
      - name: string
        image: string
        env:
        - name: string
          value: string
        # Other sidecar configurations

    finishSteps:                      # Steps to run at the end of the pipeline
    - name: string                    # Name of the finish step
      condition: string               # When to run: "success", "failed", or "always"
      # Other step configurations as above

  # Other pipeline types with same schema
  tags: []                            # Pipelines triggered by tags
  pull_requests: []                   # Pipelines triggered by pull requests
  custom: []                          # Custom pipelines (only launched by other pipelines)
```

## Pipeline Types

Pipeline-generator supports the following pipeline types:

### Branches Pipeline

Triggered when code is pushed to a branch that matches the specified regex pattern.

### Tags Pipeline

Triggered when a tag is created that matches the specified regex pattern. Commonly used for release processes.

### Pull Requests Pipeline

Triggered when a pull request is created or updated, and the source branch matches the specified regex pattern.

### Custom Pipeline

Not triggered directly by Git events. These pipelines are intended to be triggered by other pipelines using the `nextPipeline` configuration.

## Common Components

### Steps

Steps define the actual work to be performed in the pipeline. Each step runs in its own container.

### FinishSteps

Steps that execute at the end of the pipeline, based on the pipeline result. They can be configured to run on success, failure, or always run regardless of the pipeline result.

### Service Account

The Kubernetes service account used to run the pipeline. This account needs the appropriate permissions for the tasks performed in the pipeline.

### NextPipeline

Configuration for subsequent pipelines to launch after the current one completes. This allows for pipeline chaining and more complex workflows. 