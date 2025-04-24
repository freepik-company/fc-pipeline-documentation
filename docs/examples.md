# Pipeline Examples

This document contains examples of different pipeline configurations using the pipeline-generator.

## Single File Pipeline

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

## Multi-file Pipeline

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

## Tag Pipeline Example

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

## Pull Request Pipeline Example

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