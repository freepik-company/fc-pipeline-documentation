# pipeline-generator

Create Tekton pipelines from a simple YAML file.

The pipeline-generator allows you to define complex CI/CD pipelines using simple YAML configurations. It supports multiple pipeline types, customizable steps, and flexible deployment options.

## Documentation

This project's documentation has been split into several files for better organization:

- [Pipeline Examples](docs/examples.md) - Various examples showing how to configure pipelines for different use cases
- [Configuration Values](docs/values.md) - Complete reference of all available configuration options
- [Pipeline Definition Schema](docs/pipeline-definition.md) - The full schema for defining pipelines

## Quick Start

### 1. Create Your Pipeline Definition

Create a YAML file that defines your pipeline (`fc-pipelines.yaml`). See the [examples](docs/examples.md) for reference.

### 2. Validate the Pipeline

Before applying your pipeline, validate it using the fc-pipeline-renderer:

```bash
# Validate using the remote tekton server
$ docker run -it \
    -v $(HOME)/.config/gcloud:/gcp-credentials \
    -v $(PWD):/data \
    europe-west1-docker.pkg.dev/fc-shared/common/fc-pipeline-renderer:latest \
    --type branches \
    --pipeline main \
    --repository [MY_REPO] \
    --ref [MY_BRANCH_OR_TAG] \
    --render | kubectl --context [MY_TEKTON_CONTEXT] apply -f - --dry-run=server
```

This command:
- Mounts your Google Cloud credentials and current directory
- Renders the pipeline configuration
- Performs a dry-run validation against your Kubernetes cluster

### 3. Debug the Pipeline (Optional)

To inspect the generated Tekton PipelineRun without applying it:

```bash
# Show the tekton pipelinerun
$ docker run -it \
    -v $(HOME)/.config/gcloud:/gcp-credentials \
    -v $(PWD):/data \
    europe-west1-docker.pkg.dev/fc-shared/common/fc-pipeline-renderer:latest \
    --type branches \
    --pipeline main \
    --repository [MY_REPO] \
    --ref [MY_BRANCH_OR_TAG] \
    --render
```

### 4. Apply the Pipeline

Once validated, apply the pipeline to your Kubernetes cluster:

```bash
# Launch the pipeline
$ docker run -it \
    -v $(HOME)/.config/gcloud:/gcp-credentials \
    -v $(PWD):/data \
    europe-west1-docker.pkg.dev/fc-shared/common/fc-pipeline-renderer:latest \
    --type branches \
    --pipeline main \
    --repository [MY_REPO] \
    --ref [MY_BRANCH_OR_TAG] \
    --render | kubectl --context [MY_TEKTON_CONTEXT] apply -f -
```

## Key Features

- Support for multiple pipeline types: branches, tags, pull requests, and custom
- Automated integration with Git repositories
- Artifact and cache management with GCS bucket integration
- Customizable steps with support for sidecars, volumes, and environment variables
- Email notifications for pipeline status
- Ability to chain pipelines together with the `nextPipeline` feature
- Support for multi-file configurations to keep pipeline definitions organized
