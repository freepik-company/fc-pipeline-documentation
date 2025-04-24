# Configuration Values

This document describes all configuration values available for pipeline-generator.

## Values Table

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
| customLabels | list | `[]` | If `createNamespace` is set to 'true' it creates the namespace adding the labels specified to it. The value should be a list of strings in the format `key=value`. Example: `customLabels: ["custom.label=value1", "custom.label2=value2"]` or using YAML list syntax: `customLabels: [- custom.label=value1, - custom.label2=value2]` |
| image | string | `"europe-west1-docker.pkg.dev/fc-tekton/containers/gcloud-kubectl-helm-tkn:0.4.1"` | Image with helm (gcloud, kubectl and tkn also) to parse the custom pipeline and install in the Tekton cluster like a PipelineRun CRD |
| namespace | string | `"default"` | Namespace where the pipeline will be installed |
| pipelines | list | `[]` | Pipeline configuration. More information below. |
| resources | object | `{}` | Resources configuration in the run step | 