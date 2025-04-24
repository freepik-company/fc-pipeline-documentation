# pipeline-generator

Create Tekton pipelines from a simple YAML file

## Examples

Below are examples of pipeline configurations:

### Single File Pipeline

A simple pipeline configuration in a single file:

```yaml
apiVersion: "1.21.0"
namespace: my-namespace
createNamespace: true # 'false' by default
customLabels:
  - custom.label=true
  - another.customLabel=customValue

configuration:
  gitCredentialsSecret: bitbucket-credentials 
  tektonDashboardURL: https://tekton-dashboard.example.org/
  sendmailSender: jarvis@freepik.com
  defaultImagePullPolicy: IfNotPresent
  cloneDepth: 20
  nodeSelector:
    type: pipelines
  tolerations:
  - key: "type"
    operator: "Equal"
    value: "pipelines"
    effect: "NoSchedule"

pipelines:
  branches:
  - name: example-pipeline
    regex: ^main$
    serviceAccount: my-namespace
    nextPipeline: {}

    steps:
    - name: sample
      description: Just a sample step
      image: alpine:latest
      script: |
        #!/usr/bin/env sh
        set -ex
        echo "This is a sample step"

    - name: sample-no-burn
      burnupEnabled: "false"
      burndownEnabled: "false"
      description: Sample step without burnup/burndown
      image: alpine:latest
      script: |
        #!/usr/bin/env sh
        set -ex
        echo "This is a sample step with burns disabled"
```

### Multi-file Pipeline

You can split your pipeline configuration into multiple files for better organization:

**Main configuration file (fc-pipelines.yaml):**
```yaml
apiVersion: "1.21.0"
createNamespace: true
namespace: &namespace my-service
customLabels:
  - smc.slack-tektonbot-token=true
  - smc.jarvis-github-token=true
  - role.deploy-k8s/my-project=true

!include .pipelines/configuration.yaml

!include .pipelines/aliases.yaml

pipelines:
  branches:
    !include .pipelines/push.yaml

  pull_requests:
    !include .pipelines/pr.yaml
```

**Configuration file (.pipelines/configuration.yaml):**
```yaml
configuration:
  gitCredentialsSecret: github-credentials
  tektonDashboardURL: https://tekton-dashboard.example.org/
  sendmailSender: jarvis@freepik.com
  nodeSelector:
    machine_type: n2d-standard-16
  tolerations:
    - key: "type"
      operator: "Equal"
      value: "pipelines"
      effect: "NoSchedule"
  enableAutocloneRepo: false
  burnupAndBurndownEnabled: false
  sharedDataWorkspace:
    volumeClaimTemplate:
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
```

**Branch pipeline file (.pipelines/push.yaml):**
```yaml
- name: main
  regex: "^main$"
  serviceAccount: *namespace
  nextPipeline: {}
  steps:
    - name: git-clone
      description: Clone the repository
      <<: *gitClone  # Uses an alias defined in aliases.yaml

    - name: build
      description: Build the application
      runAfter:
        - git-clone
      image: alpine:latest
      script: |
        #!/usr/bin/env sh
        set -ex
        echo "Building the application"
```

**Aliases file (.pipelines/aliases.yaml):**
```yaml
# Aliases for reusable configurations

# Repository name reference
repositoryName: &repositoryName my-organization/my-service

# Custom images
builderImage: &builderImage
  image: my-registry/common/builder:1.0.0

# Git clone alias
gitClone: &gitClone
  <<: *builderImage
  burnupEnabled: "false"
  burndownEnabled: "false"
  script: |
    #!/bin/sh
    set -e
    git clone -b $(params.branch) $(params.repository) /workspace
    git reset --hard $(params.commit)
    git checkout $(params.branch)
```

### Tag Pipeline Example

A tag-based pipeline that triggers on semantic versioning tags:

```yaml
apiVersion: "1.21.0"
namespace: my-namespace

configuration:
  gitCredentialsSecret: github-credentials
  tektonDashboardURL: https://tekton-dashboard.example.org/
  sendmailSender: jarvis@freepik.com

pipelines:
  tags:
  - name: integration
    regex: ^([0-9]+)\.([0-9]+)\.([0-9]+)$
    serviceAccount: my-namespace
    nextPipeline:
      success: deploy

    steps:
    - name: deps
      description: Download dependencies
      image: composer:latest
      env:
      - name: CUSTOM_VARIABLE_TOKEN
        valueFrom:
          secretKeyRef:
            name: composer-auth-token
            key: token
      volumes:
      - name: secret
        secret:
          secretName: my-secret     
      script: |
        #!/usr/bin/env sh
        set -ex
        echo "Run compose with custom authentication using the token. ${CUSTOM_VARIABLE_TOKEN}"
      artifacts:
      - ./code/vendor
      cache:
      - ./code/vendor

    - name: tests
      image: phpunit/phpunit:7.4.0
      runAfter:
      - deps
      script: |
        #!/usr/bin/env sh
        set -ex
        echo "This step runs the tests"
      sidecars:
      - name: redis
        image: 'bitnami/redis:latest'
        env:
        - name: REDIS_PASSWORD
          value: password
      - name: memcached
        image: 'bitnami/memcached:latest'

    - name: build
      description: Build and push docker image
      image: gcr.io/kaniko-project/executor:latest
      runAfter:
      - tests
      args: ["-f", "Dockerfile", "--target", "app", "--skip-unused-stages",  "-d", "my-registry/my-repo/my-image:$(params.shortCommit)", "--context", "."]

  custom:
  - name: deployment
    regex: ^deploy$
    serviceAccount: my-namespace
    nextPipeline:
      success: deploy
      burnupEnabled: "false"
      burndownEnabled: "false"
      customParamsExtra: "value1,value2"

    steps:
    - name: deploy
      description: Deploy application
      image: alpine:latest
      burnupEnabled: "false"
      burndownEnabled: "false"
      script: |
        #!/usr/bin/env sh
        echo "Deployment step"
```

### Pull Request Pipeline Example

```yaml
apiVersion: "1.21.0"
namespace: my-namespace

configuration:
  gitCredentialsSecret: github-credentials
  tektonDashboardURL: https://tekton-dashboard.example.org/

pipelines:
  pull_requests:
  - name: pr-example
    regex: ".+"
    serviceAccount: my-namespace
    nextPipeline: {}

    steps:
    - name: tests
      description: Test in a pull request
      image: alpine:latest
      script: |
        #!/usr/bin/env sh
        echo "This runs the test in a pull request pipeline"
        
    finishSteps:
    - name: notification
      condition: "always"
      description: Send notification about PR status
      image: curl:latest
      script: |
        #!/usr/bin/env sh
        echo "Sending notification about PR status"
```

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| apiVersion | string | `"1.21.0"` | API Version to use to build the pipeline |
| charts.OCIRegistry | string | `"europe-west1-docker.pkg.dev"` | OCI image registry where the charts will be saved |
| charts.pipelineGenerator | string | `"europe-west1-docker.pkg.dev/fc-tekton/charts/pipeline-generator"` | OCI image package of the pipeline generator chart |
| configuration.artifactsEnabled | bool | `true` | If burnup and burndown is enabled this enable the artifact creation and its download process also. Enabled by dafault. |
| configuration.author | string | `"nobody"` | User name of the email sent when a failure ocurr |
| configuration.author | string | `"nobody"` | Author name of the email sent when a failure ocurr |
| configuration.burnupAndBurndownEnabled | bool | `true` | Burnup and Burndown is enabled by dafault. These automatic steps are used to download the code from the repository, the artifacts and cache packeges from GCS bucket and create the artifacts and cache packages and upload them is needed. |
| configuration.cacheEnabled | bool | `true` | If burnup and burndown is enabled this enable the cache package creation and its download process also. Enabled by dafault. |
| configuration.cloneDepth | int | `1` | When AutocloneRepo is enabled, by default clone repository with 1 as depth |
| configuration.commit | string | `""` |  |
| configuration.customParams | string | `""` | Custom parameters passed to the pipeline. You can use it to pass any parameter  needed to the pipeline in order to customize execution.  Example: `customParams: "value1,value2,value3"`, `customParams: "value1|value2|value3"`.  You can choose the separator charater, just take into account to process it properly on custom pipeline execution. |
| configuration.defaultImage | string | `"alpine:edge"` | If an image name is not given in a step this image name will be used by default |
| configuration.defaultImagePullPolicy | string | `"IfNotPresent"` | Global configuration for download policy images used on pipelines (since 1.1.0) |
| configuration.email | string | `"nobody@example.com"` | Email account where to send the email when a failure ocurr |
| configuration.enableAutocloneRepo | bool | `true` | If burnup is enabled this enable also the autoclone process to download from the repository. This is enabled by default. |
| configuration.gcsBucket | string | `"fc-tekton-artifacts"` | Bucket name to save artifacts and cache packages. Current GCS is only supported. TODO: When this program is free this will need to be changed to an example bucket name. |
| configuration.gcsCacheConfiguration | object | `{"parallelCompositeUpload":{"componenSize":32,"threshold":"100M"},"slidedObjectDownload":{"componentSize":100,"threshold":"100M"}}` | Configuration options for faster downloads/uploads caches/artifacts with gcs available.  (since 1.3.0) |
| configuration.gcsCacheConfiguration.parallelCompositeUpload | object | `{"componenSize":32,"threshold":"100M"}` | Configuration options for uploads  (since 1.3.0) |
| configuration.gcsCacheConfiguration.parallelCompositeUpload.componenSize | int | `32` | Chunks parallelize uploads  (since 1.3.0) |
| configuration.gcsCacheConfiguration.parallelCompositeUpload.threshold | string | `"100M"` | Min size for parallelize upload  (since 1.3.0) |
| configuration.gcsCacheConfiguration.slidedObjectDownload | object | `{"componentSize":100,"threshold":"100M"}` | Configuration options for downloads  (since 1.3.0) |
| configuration.gcsCacheConfiguration.slidedObjectDownload.componentSize | int | `100` | Chunks parallelize download  (since 1.3.0) |
| configuration.gcsCacheConfiguration.slidedObjectDownload.threshold | string | `"100M"` | Min size for parallelize download  (since 1.3.0) |
| configuration.gitCredentialsSecret | string | `""` | K8s secret name to authenticate with the git reposotiry if needed |
| configuration.gitFiles | string | `""` | List of files with relative path of files added, changed or deleted from the last push (set of one or more commits) (since 1.4.0) |
| configuration.nodeSelector | object | `{}` | Node selector configuration to set the pod of the pipeline launcher instance [nodeSelector](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector) |
| configuration.securityContext | object | `{"runAsNonRoot":false,"runAsUser":0}` | Pod securityContext to run the pipelines launcher instances, [securityContext](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/). Currently the user is forced to be root and to have permissions on the `/fc/workspace-data`. (since 1.2.2) |
| configuration.sendmailBody | string | `"$(params.author), the pipeline of the application '$(params.project)'  has failed.\n\nRepository '$(params.repository)'.\nBranch is '$(params.branch)'.\nCommit '$(params.commit)'.\n\n$(params.tektonDashboardURL)/#/namespaces/$(context.pipelineRun.namespace)/pipelineruns/$(context.pipelineRun.name)\n"` | Default body of the email when a failure ocurr |
| configuration.sendmailSecret | string | `"sendmail-secret"` | K8s secret with the information about the user account and the email server. More information about the structure [here](pipeline-launcher/README.md). |
| configuration.sendmailSender | string | `"launcher@example.com"` | Email account of the sender when a failure ocurr |
| configuration.sendmailSubject | string | `"Pipeline of project '$(params.project)' failed!"` | Default subject of the email when a failure ocurr |
| configuration.sendmailTaksName | string | `"fc-launcher-sendmail"` | Name of the Task in Tekton which runs the process of sending the email. The Task is installed with the launcher and its names depends of the release name given. |
| configuration.sharedDataWorkspace | object | `{}` | Tekton workspace configuration to use in a pipeline. By default no workspace is used. More information in [Workspaces](https://tekton.dev/docs/pipelines/pipelineruns/#specifying-workspaces) |
| configuration.tektonDashboardURL | string | `"https://tekton-dashboard.example.org/"` | Tekton Dashboard URL used to create a link in the email sent when a failure ocurr. For debugging process. |
| configuration.timeout | string | `"1h0m0s"` | Step max time execution. [timeout](https://tekton.dev/docs/pipelines/pipelines/#specifying-timeout)  (since 1.7.0) |
| configuration.tolerations | list | `[]` | Pod tolerations to run the pipelines launcher instances [tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) |
| configuration.type | string | `"custom"` | Type of pipeline to build. Currently the differents types supported are: branches, tags, pull_requests and custom. Custom is used only to be launched from a pipeline, not from a remote event (webhook) |
| createNamespace | bool | `false` | If this variable is set to 'true' automatically the launcher process create the namespace and service account needed to run the pipeline. The service account name will have the same name than the namespace. |
| customLabels | list | `[]` | If `createNamespace` is set to 'true' it creates the namespace adding the labels specified to it. The value should be a list of strings in the format `key=value`. Example: `customLabels: ["custom.label=value1", "custom.label2=value2"]` or using YAML list syntax:
```yaml
customLabels:
  - custom.label=value1
  - custom.label2=value2
```
 |
| image | string | `"europe-west1-docker.pkg.dev/fc-tekton/containers/gcloud-kubectl-helm-tkn:0.4.1"` | Image with helm (gcloud, kubectl and tkn also) to parse the custom pipeline and install in the Tekton cluster like a PipelineRun CRD |
| namespace | string | `"default"` | Namespace where the pipeline will be installed |
| pipelines | list | `[]` | Pipeline configuration. More information below. |
| resources | object | `{}` | Resources configuration in the run step |

----------------------------------------------
Autogenerated from chart metadata using [helm-docs v1.13.1](https://github.com/norwoodj/helm-docs/releases/v1.13.1)

## Pipeline Definition Reference

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
