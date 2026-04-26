`---
name: Mirror containers locally
---

## Set-up

* Download `skopeo` - a command line utility that performs various operations
  on container images and image repositories.
  * Windows: 
    ```
    curl.exe -LO https://github.com/passcod/winskopeo/releases/latest/download/skopeo.exe
    ```
* 

### Registry

Build or download registry.exe`

* `go.exe install -v github.com/distribution/distribution/cmd/registry@3`

I don't think the above command works, I suspect I ended up cloning the
repository and building it.


Create `registry.config.yml` with:
```yaml
# OTEL_TRACES_EXPORTER=none
version: 0.1
log:
  level: info
  fields:
    service: registry
    environment: development
storage:
    delete:
      enabled: true
    cache:
        blobdescriptor: inmemory
    filesystem:
        rootdirectory: K:/Downloads/ContainerRegistry
    tag:
      concurrencylimit: 5
http:
    addr: :5000
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
```

Run it.
```powershell
$env:OTEL_TRACES_EXPORTER="none"
c:\data\programs\registry.exe serve c:\data\programs\registry.config.yml
```

## Copy to local registry
```
skopeo.exe copy --dest-tls-verify=false docker://public.ecr.aws/aws-cli/aws-cli:latest docker://localhost:5000/aws-cli/aws-cli:latest
skopeo.exe copy --dest-tls-verify=false docker://public.ecr.aws/amazonlinux/amazonlinux:2023 docker://localhost:5000/amazonlinux/amazonlinux:2023
skopeo.exe copy --dest-tls-verify=false docker://public.ecr.aws/ubuntu/ubuntu:26.04 docker://localhost:5000/ubuntu/ubuntu:26.04
skopeo.exe copy --dest-tls-verify=false docker://p public.ecr.aws/aws-dynamodb-local/aws-dynamodb-local:3.3.0 docker://localhost:5000/aws-dynamodb-local/aws-dynamodb-local:3.3.0
skopeo.exe copy --dest-tls-verify=false docker://quay.io/libpod/busybox:latest docker://localhost:5000/libpod/busybox:latest
skopeo.exe copy --dest-tls-verify=false docker://quay.io/libpod/hello:latest docker://localhost:5000/libpod/hello:latest
skopeo.exe copy --dest-tls-verify=false docker://quay.io/podman/banner:latest docker://localhost:5000/podman/banner:latest
```

### Require Auth
For example to download from private registries which require auth such as
Amazon ECR registries.

1. Determine token
    ```
    aws ecr get-login-password --profile example
    ```
2. Define environment variable
    ```powershell
    $env:CREDS="AWS:<token here>"
    ```
3. Use `--src-creds $env:CREDS` with `skopeo copy`.

## Download image

* Choosing `dir:` means it will be downloaded to directory with the manifest
and blobs (tarballs) are separate files.
* Choosing `oci:` means a directory that matches the OCI specification so there
  are sub-directories. You can then use `tar` on this to create an OCI archive
  that can be imported.

On Windows this will try downloading for OS being windows. The `--all` command
can be provided to have all the images download, which often means both 
AMD64 and ARM64, which unless you wanting to mirror things well is overkill..
This needs `--override-os linux`.

### OCI
```
skopeo.exe copy --override-os linux --dest-tls-verify=false docker://public.ecr.aws/ubuntu/ubuntu:26.04 oci:ubuntu_26.04
skopeo.exe copy --override-os linux --dest-tls-verify=false docker://quay.io/podman/banner:latest oci:podman_banner
```

### Directory
```
skopeo.exe copy --override-os linux --dest-tls-verify=false docker://public.ecr.aws/aws-cli/aws-cli:latest dir:aws-cli
```

## Containers Pull
The following are containers that I typically pull after resetting the Podman
machine on windows:

```
podman pull public.ecr.aws/aws-cli/aws-cli:latest
podman pull public.ecr.aws/amazonlinux/amazonlinux:2023
podman pull public.ecr.aws/ubuntu/ubuntu:26.04
podman pull quay.io/libpod/busybox
podman pull quay.io/libpod/hello
podman pull quay.io/podman/banner
```

These are the AWS CLI, Amazon Linux 2023, Ubuntu 26.04 then Podman tutorial
images.


## History

### April 2026

* docker.io/library/rust:1.94-alpine3.23
* docker.io/library/rust:1.94
* ghcr.io/u-root/cpu:main
* docker.io/openapitools/openapi-generator-cli:latest
* public.ecr.aws/docker/library/rockylinux:9 
* public.ecr.aws/aws-dynamodb-local/aws-dynamodb-local:3.3.0

### March 2026
Images that I had were:
* osrm-backend from ghcr
* Valkey
* postres
* pgadmin4
* minio - This has since been abandoned by the company behind it.
* Issue trackers - to evaluate replacement for self-hosted Jira
** Makeplane
** Taigaio
* registry.access.redhat.com/ubi10/ubi 
* quay.io/curl/curl  
* quay.io/lib/mongo-express 
* rabbitmq
