
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
* Allow you to copy from a registery to a OCI image file.

## Image Copying

`skopeo` can be useful for copying a container image in a public registry
such as AWS ECR, Docker Hub or Quay into a on-premise registry such as one
hosted by a local GitLab instance.

It can also be used to copy an image from a private AWS ECR requiring
authentication to your local registry.


[docker-hub]: https://hub.docker.com/
[gh-skopeo]: https://github.com/containers/skopeo
[aws-ecr-registry]: https://gallery.ecr.aws/
[quay]: https://quay.io/

