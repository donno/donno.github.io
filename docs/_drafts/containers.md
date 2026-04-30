
Containers
==========

Place holder / reminder to perform a write-up on containers.

Topics to cover
* Container Image vs Container
* Windows vs Linux
* Job objects and Control Groups / Namespaces
* OCI Images
    * layers
    * manifest
* Container runtime interface (CRI)
* Container registeries
* OCI / container runtimes
    * containerd
    * runc
    * crun
    * gvisord
    * cri-o
    * rkt
    * kata
    * yuki
    * moby
* Management tools
    * docker
    * nerdctl
    * podman
* Snapshotters
    * Manage the snapshots of the container filesystems.
    * overflayfs - Default
    * devmapper`
    * btrfs
    * zfs
    * erofs
    * Non-core snapshotter plugins
        * Nydus
        * OverlayBD
        * Stargz
* Tools
    * buildah
    * skopeo
    * dive
* Container Centric Distros
    * Fedora CoreOS
    * Bottlerocket CoreOS
* Software as a Service offerings
    * Amazon ECS

Outside scope
* Kubernetes - haven't experienced this yet.

# Public Registries

For public registries it is important to check the terms of service for using
the registry. For example, rate limits may occur for unauthenticated users
or free users.

* [Docker Hub][docker-hub]
* [Quay.io by RedHat][quay]
* [Amazon ECR Public Registry][aws-ecr-registry]

# skopeo

* Works without interacting with a container runtime
* Allows you to copy a container image between registries.
* Allow you to copy from a registry to a OCI image file.

## Image Copying

`skopeo` can be useful for copying a container image in a public registry
such as AWS ECR, Docker Hub or Quay into a on-premise registry such as one
hosted by a local GitLab instance.

It can also be used to copy an image from a private AWS ECR requiring
authentication to your local registry.

## Snapshotter
* Snapshotters manage the snapshots of the container filesystems.
* Remote Snapshotter leverages snapshots stored in shared place.
** The Stargz Snapshotter is example from it.

### Nydus

* Nydus implements a content-addressable file system on the RAFS format.
* Aims to output perform OCI images when ti comes to cold startup time on containerd.
** For example, `tensorflow` takes 218s vs 2.2s with Nydus.
* Supports containerd, CIRO-, PodmMan and Kubernetes via CRI

### OverlayBD
* OverlayBD is a remote container image format base on block-device
* Use `nerdctl image convert` to convert from OCI image to OverlayBD image.
* `nerdctl image convert --overlaybd --oci <source_image> <target_image>`
* Paper: [DADI: Block-Level Image Service for Agile and Elastic Application Deployment](https://www.usenix.org/conference/atc20/presentation/li-huiba)

[docker-hub]: https://hub.docker.com/
[gh-skopeo]: https://github.com/containers/skopeo
[aws-ecr-registry]: https://gallery.ecr.aws/
[quay]: https://quay.io/
[erofs-containers]: https://static.sched.com/hosted_files/kccncosschn21/fd/EROFS_What_Are_We_Doing_Now_For_Containers.pdf
