# pipeline-generator

Create Tekton pipelines from a simple yalm file

![Version: 1.15.0](https://img.shields.io/badge/Version-1.15.0-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square)

## Examples

Next an example of pipeline:

A simple pipeline configuration with a selection of nodes of kubernetes using `nodeSelector` and `tolerations`.
Also create automatically the namespace and service account where to run the steps of this pipeline if it doesn't exist.
Set custom labels in the namespace resource to allow tools like `kyverno` deploying custom resources for our new namespace:
```yaml
apiVersion: "1.1.0"
namespace: my-namespace
createNamespace: true # 'false' by default
customLabels: "custom.label=true another.customLabel='customValue'"

configuration:
  gitCredentialsSecret: bitbucket-credentials # The secret name with the credentials to allow
                                              # access to private repositories. Call it github-credentials,
                                              # for example, to authenticate with github repositories.
  tektonDashboardURL: https://tekton.cdnpk.net
  sendmailSender: jarvis@freepik.com
  defaultImagePullPolicy: IfNotPresent # Since 1.1.0 you can choose a policy to download images
  cloneDepth: 20 # since 1.0.4 yu can  choose a depth to clone repository
  # Example configuration to set the nodepool where to run the pipelines.
  # This is optional.
  nodeSelector:
    type: pipelines
  tolerations:
  - key: "type"
    operator: "Equal"
    value: "pipelines"
    effect: "NoSchedule"

pipelines:
  ...
```

A simple pipeline configuration with a shared data workspace:
```yaml
apiVersion: "1.1.0"
namespace: my-namespace

configuration:
  gitCredentialsSecret: bitbucket-credentials
  tektonDashboardURL: https://tekton.cdnpk.net
  sendmailSender: jarvis@freepik.com
  # This is an example of a workspace configuration where to save the code,
  # dependencies and builds between steps. This is optional and you need to use it
  # carefully due to use this configuration does not allow you to run steps in
  # more than one kubernetes node because a Workspace is a Volume Disk linked an
  # only one machine. It's recommended to use artifacts and cache instead.
  sharedDataWorkspace:
    volumeClaimTemplate:
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi

pipelines:
  ...
```

A configuration of pipeline for a `master` branch. It's composed of two steps:
```yaml
...

pipelines:
  # A pipeline of type 'branch'. It will run with a push to the branch master.
  branches:
  - name: example-pipeline
    regex: ^master$
    # Service account with permission to launch new pipelines.
    serviceAccount: my-sa
    # This pipeline won't launch a new pipeline
    nextPipeline: {}

    steps:
    # Sample step
    - name: sample
      description: Just a sample step
      image: alpine:latest
      script: |
        #!/usr/bin/env sh
        set -ex
        echo "This is a sample step"

    steps:
    # Sample step with burns disables
    - name: sample
      burnupEnabled: "false"
      burndownEnabled: "false"
      description: Just a sample step
      image: alpine:latest
      script: |
        #!/usr/bin/env sh
        set -ex
        echo "This is a sample step"
```

Another sample step that shows us a 'tag' type of pipeline running a new pipeline when the first one finishes successfully:

```yaml
...

pipelines:
  tags:
  - name: integration
    # A clumsy regex to allow pushed tags in a simple SemVer way
    regex: ^([0-9]+)\.([0-9]+)\.([0-9]+)$
    serviceAccount: my-sa
    nextPipeline:
      # If the integration pipeline finishes successfully, it will run a custom pipeline whose regular expression match 'deploy'.
      success: deploy

    steps:
    # Download dependencies
    - name: deps
      description: download dependencies
      image: composer:latest
      # Example of usage of custom environment variables
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
      # Example of artifacts and cache usage
      artifacts:
      - ./code/vendor
      cache:
      - ./code/vendor

    # tests
    - name: tests
      image: docker pull phpunit/phpunit:7.4.0
      # Run after example usage
      runAfter:
      - deps
      script: |
        #!/usr/bin/env sh
        set -ex
        echo "This step runs the tests
      # Usage example of sidecars needed by the tests
      sidecars:
      - name: redis
        image: 'bitnami/redis:latest'
        env:
        - name: REDIS_PASSWORD
          value: password
      - name: memcached
        image: 'bitnami/memcached:latest'

    # build image
    - name: build
      description: build and push docker image
      image: gcr.io/kaniko-project/executor:latest
      runAfter:
      - test
      # In this step we show how to use args to the command instead using a script
      args: ["-f", "Dockerfile", "--target", "app", "--skip-unused-stages",  "-d", "europe-west1-docker.pkg.dev/fc-shared/mentor/test-pipeline:$(params.shortCommit)", "--context", "."]

  # Custom pipelines for deploy with burns disabled on nextPipeline
  custom:
  - name: deployment
    regex: ^deploy$
    serviceAccount: my-sa
    nextPipeline:
      success: deploy
      burnupEnabled: "false"
      burndownEnabled: "false"
      customParamsExtra: "value1,value2"

    steps:
    - name: deploy
      description: deploy application
      image: alpine:latest
      burnupEnabled: "false"
      burndownEnabled: "false"
      script: |
        #!/usr/bin/env sh
        echo "Deployment step"
```

Next we show how to run a pull request pipeline:

```yaml
...

pipelines:

  # Pull requests
  pull_requests:
  - name: pr-example
    # Run with whatever the name of the pull request branch
    regex: ".+"
    serviceAccount: my-sa
    nextPipeline: {}

    steps:
    - name: tests
      description: test in a pull request
      image: alpine:latest
      script: |
        #!/usr/bin/env sh
        echo "This runs the test in a pull request pipeline"
```

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| apiVersion | string | `"1.15.0"` | API Version to use to build the pipeline |
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
| configuration.gcsCacheConfiguration | object | `{"parallelCompositeUpload":{"componenSize":32,"threshold":"100M"},"slidedObjectDownload":{"componentSize":100,"threshold":"100M"}}` | Configuration options for faster downloads/uploads caches/artifacts with gcs available.  (since 1.3.0) |
| configuration.gcsCacheConfiguration.parallelCompositeUpload | object | `{"componenSize":32,"threshold":"100M"}` | Configuration options for uploads  (since 1.3.0) |
| configuration.gcsCacheConfiguration.parallelCompositeUpload.componenSize | int | `32` | Chunks parallelize uploads  (since 1.3.0) |
| configuration.gcsCacheConfiguration.parallelCompositeUpload.threshold | string | `"100M"` | Min size for parallelize upload  (since 1.3.0) |
| configuration.gcsCacheConfiguration.slidedObjectDownload | object | `{"componentSize":100,"threshold":"100M"}` | Configuration options for downloads  (since 1.3.0) |
| configuration.gcsCacheConfiguration.slidedObjectDownload.componentSize | int | `100` | Chunks parallelize download  (since 1.3.0) |
| configuration.gcsCacheConfiguration.slidedObjectDownload.threshold | string | `"100M"` | Min size for parallelize download  (since 1.3.0) |
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
| configuration.tektonDashboardURL | string | `"http://tekton.example.com"` | Tekton Dashboard URL used to create a link in the email sent when a failure ocurr. For debugging process. |
| configuration.timeout | string | `"1h0m0s"` | Step max time execution. [timeout](https://tekton.dev/docs/pipelines/pipelines/#specifying-timeout)  (since 1.7.0) |
| configuration.tolerations | list | `[]` | Pod tolerations to run the pipelines launcher instances [tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) |
| configuration.type | string | `"custom"` | Type of pipeline to build. Currently the differents types supported are: branches, tags, pull_requests and custom. Custom is used only to be launched from a pipeline, not from a remote event (webhook) |
| createNamespace | bool | `false` | If this variable is set to 'true' automatically the launcher process create the namespace and service account needed to run the pipeline. The service account name will have the same name than the namespace. |
| customLabels | string | `""` | If `createNamespace` is set to 'true' it creates the namespace adding the labels here specified to it. The value of this variables is a string of `keys=values` where `keys` is the name of the label and `values` its value. Every `key=value` must be separated by a blank space. Example: `customLabels: "custom.label2=value1 custom.label2=value2" |
| image | string | `"europe-west1-docker.pkg.dev/fc-tekton/containers/gcloud-kubectl-helm-tkn:0.4.1"` | Image with helm (gcloud, kubectl and tkn also) to parse the custom pipeline and install in the Tekton cluster like a PipelineRun CRD |
| namespace | string | `"default"` | Namespace where the pipeline will be installed |
| pipelines | list | `[]` | Pipeline configuration. More information below. |
| resources | object | `{}` | Resources configuration in the run step |

----------------------------------------------
Autogenerated from chart metadata using [helm-docs v1.13.1](https://github.com/norwoodj/helm-docs/releases/v1.13.1)

## Definition of pipeline

```yaml
pipelines:
  branches:                           # type of pipeline: branches, tags, pull_requests
                                      # and custom
  - name:                             # name of the pipeline
    regex:                            # regular expression to match
                                      # the branch or tag
    serviceAccount:                   # k8s service account with permission
                                      # to launch new pipelines and deploy
                                      # new manifests
    nextPipeline:                     # default value '{}'
      success:                        # custom pipeline to launch whose regex field
                                      # match with the value of 'success'
      failed:                         # custom pipeline to launch whose regex field
                                      # match with the value of 'failed'
      customParamsExtra:              # add the value of this variable to customParams if exists and send it to next pipelines. It can use the results of previous steps.
      burnupEnabled:                  # enable (true and default) or disable (false) the `burnup` in the step that launch next pipelines
      burndownEnabled:                # enable (true and default) or disable (false) the `burndown` in the step that launch next pipelines

    steps:                            # list of steps to run in the pipeline
    - name:                           # name of the step
      timeout:                        # timeout for the step. Use tekton syntax. Example: 1h30m0s
      description:                    # a description for the step
      runAfter: []                    # list of steps in this pipeline to run
                                      # after them
      burnupEnabled:                  # values are "true" or "false", for download repository, caches or artifacts, since 0.3.2 fc-pipeline-generator version, default value is true
      burndownEnabled:                # values are "true" or "false", for download repository, caches or artifacts, since 0.3.2 fc-pipeline-generator version, default value is true
      image:                          # container imagen to run in this step
      imagePullPolicy:                # since 1.1.0 specific policy for download image for step
      resources: {}                   # since 1.2.0 this option is treatment as resources of pods
                                      #  resources:
                                      #    limits:
                                      #      cpu: 4
                                      #     requests:
                                      #      cpu: 3
                                      # https://kubernetes.io/docs/concepts/configuration/manage-resources-containers
      env: []                         # list of environment variables in kubernetes
                                      # format.
                                      # https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/
      params:                         # list of custom parameter to use later in the
                                      # script or arguments
      - name:                         # name of the custom parameter
        value:                        # value of the custom paramater
      volumes: []                     # custom volumes. It mount a volume in /mnt/[volumeName] in the step.
                                      # If a configmap or secret is used, it will be mounted in /mnt/[volumeName]/[keyName].
                                      # More information in https://tekton.dev/vault/pipelines-v0.16.3/tasks/#specifying-volumes
      command: []                     # list of string to define the command of
                                      # the step
      args: []                        # list of string to define the arguments
                                      # of the command
      script:                         # script to run instead a command/args
      artifacts: []                   # list of string to define the generated artifacts
                                      # we want to save in this step.
      cache: []                       # list of string to defint wich files will be
                                      # chached to be retrieve in a future pipeline runs
      sidecars:                       # list of containers which run at the same
                                      # time that this step, for example to run
                                      # depedencies for testing.
      - name:                         # name of the container
        image:                        # name of the image
        env: []                       # list of environment variables in kubernetes
                                      # format.
                                      # https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/
        ...                           # More information in https://tekton.dev/vault/pipelines-v0.16.3/tasks/#specifying-sidecars
    finishSteps:                      # steps to run at the end of the pipeline
      - name:                         # name of the finish step
        condition:                    # condition when this step must be run.
                                      # possible values are: success, failed and
        ...                           # always. Read about steps above

  tags: {}                            # pipelines of type tag.
                                      # Read branches above
  pull_requests: {}                   # pipelines of type pull_request.
                                      # Read branches above
  common: {}                          # pipelines of type common.
                                      # Read branches above
```
