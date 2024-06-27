# Fun with OCI

During my last few months on the Zarf team I have been increasingly focused on OCI within Zarf. Checkout my (last post)[] for an explanation of what OCI is and how we use OCI images within Zarf. This post will assume some knowledge of OCI. I thought it would be fun (tm) to write out some of recent OCI related issues within Zarf and how we solved them. 

## Image index sha's cannot be pulled by Zarf
Zarf preforms lots of magic image manipulation, usually this works great, however OCI images tagged with a sha pointing to an (image index)[https://github.com/opencontainers/image-spec/blob/main/image-index.md] is a notable exception that cannot be pulled by Zarf. An Image index points to specific (image manifests)[https://github.com/opencontainers/image-spec/blob/main/image-index.md] which then points to layers which make up the images. You can think of the image manifests as the objects representing the actual image for a specific platform (os & architecture), while image indexes represent the image on any available platform. Let's look at an example
```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.index.v1+json",
  "manifests": [
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:b3fabdc7d4ecd0f396016ef78da19002c39e3ace352ea0ae4baa2ce9d5958376",
      "size": 673,
      "platform": {
        "architecture": "arm64",
        "os": "linux"
      }
    },
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:454bf871b5d826b6a31ab14c983583ae9d9e30c2036606b500368c5b552d8fdf",
      "size": 673,
      "platform": {
        "architecture": "amd64",
        "os": "linux"
      }
    }
  ]
}
```
The above index.json references two manifests. They are the amd64 and arm64 images respectively. While not yet common, tagging an image using an index sha is, in my opinion, best practice, so long as you don't care about the airgap. One major advantage is that images can be given immutable tags while not being architecture or OS specific. A public helm chart can tag it's image with an immutable sha, while not limiting it's helmchart to a certain platform or architecture. Tools in the ecosystem like docker, and crane will traverse the index.json to find the desired image for the users platform. Indexes are immutable to changes because indexes they contain digests of manifests which contain digests of container filesystems. If any link in the chain changes the digest of all the parents will change. Structures like this are known as (merkle trees)[https://en.wikipedia.org/wiki/Merkle_tree]. 

This is problematic in the air-gap as an image index requires that all manifests be present in the registry. If we wanted to bring an index.json into the air-gap we would have to bring every single manifest referenced. Many popular images, such as nginx, often publish indexes with around fifteen different platforms. Bringing in fifteen different images when you expected only one could easily bloat a package and frustrate users. Instead we give users an error message and the choice to grab an index
![image](image.png) 

One day in the future we may want to support pulling indexes into Zarf, they would provide an easy way to create multi architecture packages. However, the demand for multi architecture packages is not great enough yet, to justify adding this and the potential for misuse led us to deciding to not support this feature entirely and exit with a nice error message.

# Zarf local docker fallback breaking on with Docker in containerd Mode

Zarf has the ability to fallback and grab images stored in the docker daemon. You can actually force this behavior with a clever hack created by @RothAndrew, to build your package without internet
```bash
docker network rm no-internet-net || true
docker network create --internal no-internet-net
docker run --platform linux/amd64 --rm -v $(pwd):/work -v /var/run/docker.sock:/var/run/docker.sock -w /work/zarf --network no-internet-net ghcr.io/defenseunicorns/build-harness/build-harness:2.0.24 uds zarf package create --architecture $(scripts/get_arch.sh) --confirm --skip-sbom --no-progress
docker network rm no-internet-net
```
Zarf grabs the metadata for images using Crane. Crane implements a way to grab docker images but assumes images pulled from docker client will  configName

# cosign images

# helm charts

