---
layout: post
title:  "An OCI Image from Python"
date:   2024-08-10 20:00:00 +1030
---

After looking into the recent addition of experimental support for Windows in
`BuildKit` for `Containerd`, I decided to take a deep dive into seeing if I could
create my own [OCI (Open Container Initiative)](0) image from Python.

The plan was to start with the [oci-python](1) project. I didn't get far wit
 it as it had a strange HTTP wrapper class which results in it looking
too much like the caller is making HTTP requests themselves and thus wasn't
providing the abstraction that I was looking for.

There are essentially two parts to this project, the first is interacting
with a container registry to download the base image, well specifically the
base layer and the second is to create a tarball for another layer and put that
all together into the tarball that forms the OCI image.

Starting with an overview over the components of an image by taking a look
at an existing OCI image to see what goes into it there is:
* oci_image.tar
    * oci-layout - JSON document containing `{"imageLayoutVersion":"1.0.0"}`
    * index.json - [Image Index](3) containing an annotated list of manifests.
    * blobs/ - Directory
        * sha256/ - This directory is named after the digest algorithm, the
          other supported algorithm at this time is sha512.
            * digest-1 - tarball for first layer.
            * digest-2 - tarball for second layer.
            * digest-3 - The [Image Manifest](4), which is a JSON Document
              containing information about the layers.
            * digest-4 - The [Image Configuration](5), which is contains
              information about how the image was created as well as how to
              run it like the entry point / commands.

The tarball for a layer essentially contains the contents of the file-system.
For a Windows containers, it contains two top-level directories, `Files` and
`Hive` which the first map to C:\ within the container and the second is loaded
into the Windows Registry.

The different tasks that needed to be performed to construct the OCI image are:
- Fetch the tarball for the first layer from a existing container registry.
- Create a tarball of the second layer.
- Describe and create the image manifest.
- Describe and create the image configuration.
- Describe and create the image index.
- Put it all together in a final tarball.

## Fetching first layer

As mentioned before, I started with [oci-python](1) for this part however
quickly abandoned it in favour of using [requests](6) to handle the HTTP
requests myself. I had previously looked querying Microsoft's container registry
before so I already had a starting point for the API.

If you are however wondering there is an OCI specification called the
[OCI Distribution][7] which describes how to push, pull and discover content
from a registry. The two parts I was interested in was:
- Pulling an Image Manifest - with the ideas of going from a registry, name and
  reference such as `mcr.microsoft.com`, `windows/nanoserver` and `2004-amd64`
  respectively and getting the layer.
- Pulling a Layer - Downloading the blob that makes the layer.

The former is done by performing a GET request on
`<registry>/v2/<name>/manifests/<reference>` and once you read the manifest it
tells you the blobs to pull and then you perform a GET request on
`<registry>/v2/<name>/blobs/<digest>`. From there you have the ID of the
blob that makes up the layer (or layers) and do another GET request on the
blobs resource.

## Create the second layer
The layer is simply a tarball of the files that should exist in the filesystem
when applied to the previous layer.

This is where I spent the most type trying to get things correct. The big
mistake I made was mis-interpreting "REG" to mean registry instead of
regular file.

Building a image layer for Windows container required two extra PAX headers.

* MSWINDOWS.fileattr which is "32" for a regular file, and "16" for a directory.
* MSWINDOWS.rawsd which is a special encoding of the security descriptor, which
  you can think of it as the owner, group and permissions associated with the
  file. The values I used came straight from [buildkit](8).

While it didn't seem to have any negative affect, the permissions in the tar
simply said if the member/entry was a directory or not (i.e. set no 
read/write or execute bit).

## Create the image manifest
The media type for this is application/vnd.oci.image.manifest.v1+json.

The two things this JSON document provides are the:
- List of layers (the digest, size and media type of each layer) which the
  object in the JSON is called a ["descriptor"](9).
* The digest and size of the blob that represents the image configuration,
  which hasn't been tackled yet, so that is the next part.

## Describe and create the image configuration.
The media type for this is application/vnd.oci.image.config.v1+json.

This is mostly JSON document with most the values hardcoded, with the
exception of adding the diff_ids to the rootfs object. Since I had no need to
be able to bake additional environment variables into the image that part was
hard-coded.

The tricky part here was what diff IDs meant. There are two options for the
IDs. The first is the digest of the compressed tarball and the second is the
the digest of the uncompressed tarball. In this case it needs to be the
uncompressed tarball.

In my implementation, I created the configuration based on the layers from the
manifest which tracked the ID of the uncompressed layer via that despite
the manifest itself not needing that.

## Describe and create the image index.
This is again straight forward in my case the resulting image was something
like this, where the image name and ref name was configurable.
```
{
    "schemaVersion": 2,
    "mediaType": "application/vnd.oci.image.index.v1+json",
    "manifests": [
        {
            "mediaType": "application/vnd.oci.image.manifest.v1+json",
            "digest": "sha256:92a8fd3c1fc9a5da5f725b1a13c07f2c4bf6d862dd537981e6b07a036ea58379",
            "size": 588,
            "annotations": {
                "io.containerd.image.name": "donno.id.au/cowsay:frompython",
                "org.opencontainers.image.created": "2024-08-09T00:00:00Z",
                "org.opencontainers.image.ref.name": "frompython"
            },
            "platform": {
                "architecture": "amd64",
                "os": "windows",
                "os.version": "10.0.19045"
            }
        }
    ]
}
```

The annotations are the metadata about the image and the key part of the
index is the digest of the manifest so it can then look in the blobs directory
within the tarball to find the manifest.

## Problems and Troubleshooting

Without including the directories themselves in the tarball, this resulted in
the following error:
```
unpacking donno.id.au/cowsay:frompython (sha256:34f54d28819f1ad6b0b8ad5656eca89b79301802e721079275495a257433bcbf)...
time="2024-08-10T09:51:18+09:30" level=info msg="apply failure, attempting cleanup" error="failed to extract layer sha256:e70d0d37cb36f8b3aa222fce81c1437f7802634f71988c4f94c8ec5fce540c20: mkdir \\\\?\\C:\\Users\\Donno\\AppData\\Local\\Temp\\hcs900059622\\Files\\app\\cowsay.exe: The system cannot find the path specified.: unknown" key="extract-656255200-YeoL sha256:39c976a964e30b7c67c289aae03f47847af94ee81beddd2e6cd1276c3fbda9ca"
time="2024-08-10T09:51:18+09:30" level=fatal msg="failed to extract layer sha256:e70d0d37cb36f8b3aa222fce81c1437f7802634f71988c4f94c8ec5fce540c20: mkdir \\\\?\\C:\\Users\\Donno\\AppData\\Local\\Temp\\hcs900059622\\Files\\app\\cowsay.exe: The system cannot find the path specified.: unknown"
```

While trying to get the image to run the steps performed to help trouble shoot
the problem came down to comparing the the container created from Python with
one created with BuildKit.

* The first mistake made was the wrong digest had been included in the
  configuration. This meant it used thee layer made with buildkit instead.
  The result was when the image was loaded and run it worked because despite
  teh reference layer not being in the OCI image it was already available on
  the system. If the image was loaded onto another machine then it would
  have failed as the blob would have been missing.
* Comparing the image tarball with 7-zip to see how the PAX headers wearere set
  as well as permissions.
  * For the layer image tarball, I originally had read/write and execute set
    where the other image didn't.
* Comparing the two images with [Skopeo](10) to see where the differences
  lie. For Windows there is [WinSkopeo](11) which generates builds from the
  former project specially for Windows.
    * This highlighted that the `ENV` section should have had  `PATH` set
      to C:\Windows\System32 and C:\Windows..

## Future
- Allow the script to build a Linux image by testing it and separating it out
  the Windows specific parts.
- Add option to produce the same second layer with varying base layer. For
  example, be able to produce an image with nanoserver, servercore and server
  or at least handle different versions, for example ltsc2019, 20H2, 1903,
  1909, 2004 and 2009.
- Add option to make the tarballs reproducible by having none of the dates
  based on current time. The idea is if the files added to the layer are the
  same then the resulting image for the layer should be the same.

## Python API

```python
@dataclasses.dataclass
class Layer:
    """Describes a layer for the OCI Manifest.

    This itself is a OCI Content Descriptors.
    """

    uncompressed_size: int
    """Specifies the size, in bytes, of the raw content."""

    compressed_digest: str
    """The digest of the targeted content.

    The format of the digest is <algorithm>:<hash>.
    """

    uncompressed_digest: str
    """The digest of the targeted content when it is uncompressed.

    The format of the digest is <algorithm>:<hash>.
    """

    media_type: str = "application/vnd.oci.image.layer.v1.tar+gzip"
    """The media type of the referenced

    The value must comply with RFC 6838, including the naming requirements in
    its section 4.2.
    """


class Manifest:
    """An OCI image manifest.

    The specification for this can be found at the following address:
        https://github.com/opencontainers/image-spec/blob/main/manifest.md
    """

    def set_config(self, config: str):
        """Set the config section of the image manifest as given."""
        ...

    def add_layer(self, layer: Layer):
        """Add a layer to the manifest."""
        ...
```

### Functions Declarations
```python
def base_image(name: str, config: Configuration) -> Layer:
    """Provide the base image by downloading it through a registry."""
    ...

def create_layer(
    config: Configuration,
    source_directory: pathlib.Path,
    target_directory: pathlib.PurePosixPath,
    *,
    is_windows: bool = True,
) -> Layer:
    """Create a layer."""
    ...
```

### Usage
Brining that all together to create a Windows Nanoserver image with cowsay
in it becomes:

```python
def example(config: Configuration):
    """Example of creating an image."""
    # Prepare
    url = "https://github.com/Code-Hex/Neo-cowsay/releases/download/v2.0.4/cowsay_2.0.4_Windows_x86_64.zip"
    cowsay_archive = download_file(
        url, config.work_directory, "cowsay.zip", skip_if_exists=True
    )
    extraction_directory = config.work_directory / "cowsay_extracted"
    with zipfile.ZipFile(cowsay_archive) as opened_archive:
        opened_archive.extractall(extraction_directory)

    # Create layer
    new_layer = create_layer(
        config,
        extraction_directory,
        pathlib.PurePosixPath("app"),
    )

    # Grab the information for the base layer.
    base_layer = base_image(
        "mcr.microsoft.com/windows/nanoserver:2004-amd64",
        config,
    )

    new_manifest = Manifest()
    new_manifest.add_layer(base_layer)
    new_manifest.add_layer(new_layer)

    create_oci_image(
        new_manifest,
        config.work_directory / "new-oci_image.tar",
        config,
        image_name="donno.id.au/cowsay:frompython",
        reference="frompython",
    )
```

[0]: https://github.com/opencontainers/image-spec
[1]: https://github.com/vsoch/oci-python/
[2]: https://github.com/opencontainers/image-spec/blob/main/spec.md
[3]: https://github.com/opencontainers/image-spec/blob/main/image-index.md
[4]: https://github.com/opencontainers/image-spec/blob/main/manifest.md
[5]: https://github.com/opencontainers/image-spec/blob/main/config.md
[6]: https://requests.readthedocs.io/en/latest/
[7]: https://github.com/opencontainers/distribution-spec/blob/v1.0.1/spec.md
[8]: https://github.com/moby/buildkit/blob/22156ab20bcaea1a1466d277dbf1f1386fa23bd9/util/winlayers/differ.go#L194-L204
[9]: https://github.com/opencontainers/image-spec/blob/main/descriptor.md
[10]: https://github.com/containers/skopeo
[11]: https://github.com/passcod/winskopeo
