# cache & cache-upload

This task allows caching dependencies and build outputs using S3 to improve workflow execution time like [github cache action](https://github.com/actions/cache).

## Install the Task

```
kubectl apply -f https://api.hub.tekton.dev/v1/resource/tekton/task/cache/0.1/raw
```

## Parameters

- **key**: An explicit key for a cache entry
- **path**: A list of files, directories, and wildcard patterns to cache and restore from source workspace
- **bucket**: S3 bucket to save cache files
- **verbose**: Log the commands that are executed during `cache`'s operation.


## Workspaces

- **source**: To upload and download cached files and directories from and to.
- **secrets**: A workspace that consists of credentials required by `aws` which will be mounted to /root/.aws. This workspace can be omitted when using instance-profile credentials.


## Secret

AWS `credentials` and `config` both should be provided in the form of `secret`.

[This](../aws-cli/0.2/samples/secret.yaml) example can be referred to create `aws-credentials`.

Refer [this](https://docs.aws.amazon.com/sdk-for-java/v1/developer-guide/setup-credentials.html) guide for setting up AWS Credentials and Region.

## Platforms

The Task can be run on `linux/amd64` platform.

## Usage

```yaml
kind: Pipeline
apiVersion: tekton.dev/v1beta1
metadata:
  name: build

  workspaces:
  - name: source
    description: Workspace where steps runs
  - name: aws-credential
    description: AWS credential
spec:
  params:
  # Driver build
  - name: driver-revision
    description: Git revision of driver repository
    type: string
  - name: driver-repository
    description: Git repository of driver repository
    type: string

  # Firwmare build
  - name: firmware-revision
    description: Git revision of firmware repository
    type: string
  - name: firmware-repository
    description: Git repository of firmware repository
    type: string

  - name: cache
    taskRef:
      name: cache
    params:
    # Use unique cache key
    - name: key
      value: $(params.driver-revision)-$(params.firmware-revision)
    # What to be cached relative from $(workspaces.source.path)
    - name: path
      value: |
        driver.deb
        firmware.img
    - name: bucket
      value: "s3://cache-bucket"
    workspaces:
    - name: source
      workspace: source
    - name: secrets
      workspace: aws-credential

  # Build driver to $(workspaces.source.path)/driver.deb
  - name: build-driver
    taskRef:

    workspaces:
    - name: source
      workspace: source
    when:
      # Only when cache miss
      - input: $(tasks.cache.results.cache-hit)
        operator: in
        values: ["false"]

  # Build firmware to $(workspaces.source.path)/firmware.img
  - name: build-firmware
    taskRef:
        ...
    workspaces:
    - name: source
      workspace: source
    when:
      # Only when cache miss
      - input: $(tasks.cache.results.cache-hit)
        operator: in
        values: ["false"]

  finally:
  - name: cache-upload
    # Upload cache when hit failed
    when:
      - input: $(tasks.cache.results.cache-hit)
        operator: in
        values: ["false"]
    taskRef:
      name: cache-upload
    params:
    - name: key
      value: $(params.driver-revision)-$(params.firmware-revision)
    - name: path
      value: |
        driver.deb
        firmware.img
    - name: bucket
      value: "s3://cache-bucket"
    workspaces:
    - name: source
      workspace: source
    - name: secrets
      workspace: aws-credential
```
