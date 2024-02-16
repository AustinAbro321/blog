## What is OCI and ORAS

OCI or the "Open Container Initiative" decouples containers from Docker. Similar to how Kubernetes is no longer just about managing container images, but is now also a generic resource manager, OCI artifacts are no longer limited to being container images, but can now hold any set of files so long as they're made to follow the [OCI specification](https://github.com/opencontainers/image-spec).

In order to take advantage of the benefits of this specification, a community project called [ORAS](https://oras.land/) (OCI Registry as Storage) was created. ORAS makes it simpler to store OCI artifacts in online registries such as GHCR or ECR by providing a CLI and library for pushing artifacts to registries.

## Why OCI

You might be thinking, why even use OCI for Zarf? You can just cURL up the tarballs and cURL them back down. Let's look at the benefits:

*   Reuse of existing registries. Tools in the Kubernetes ecosystem like Zarf and Helm know users are likely to have a registry so the reuse of this registry is convenient.
    *   This means we can use the same authentication system as well.
*   Layers! When you download a Zarf package if you already have the image in your Zarf cache we can skip downloading it.
    *   Theoretically, we could do the same for components, but we want to avoid bloating users' systems with huge blob files. However, registries get to take advantage of components as OCI layers which saves storage space for any components that are unchanged between architectures or versions.
*   Better user experience.
    *   Users don't need to specify which architecture they are working in, Clients for OCI registries generally will detect the architecture on your computer and automatically download it for you. Regestries group packages with different architectures and version together
    *   OCI tools have built in cross platform features for registry operations. Users no longer need to maintain a set of cURL scripts, they can simply run `zarf package pull`, or `zarf package publish`.
*   Built-in signing. This ensures that if any files are altered or corrupted during the download process, whether through a man-in-the-middle attack or an error, ORAS will automatically detect the discrepancy and abort the download

## What is a Zarf OCI package

Here is a diagram of what an OCI package in Zarf looks like under the hood.

End users will never have to interact with the indexes, manifests, or manifest configs. Zarf, ORAS, and the registry handle the standard OCI files.

Let's run through an example of what these files do. If we pull this package from an arm64 computer Zarf will check the index for a manifest with the platform arm64. If an arm64 manifest exists it will pull down that manifest. The manifest contains a list of pointers to the blob files that Zarf puts together to create a package

You might notice in the above diagram that we have image indexes, manifests, and layers. Another neat aspect of OCI is that we can store OCI artifacts within other OCI artifacts. ORAS will detect this happening and resolve everything properly during pull and push operations. This is especially important to us as we have UDS bundles that make up Zarf packages. This gives us the storage benefits of layer  ?! <- Need to make sure UDS-CLI treats Zarf in this same way before I say this. 

## Why OCI is becoming a library
We are currently refactoring Zarf and will soon release an OCI library to be used across defense unicorns. You may wonder why we need this library since ORAS already exists. In short, the ORAS library is complicated to use and unopiniated. Our library aims to make it easier to interact with OCI primitives such as manifests, indexes, and layers. This simplifies common operations like copying, pulling, and pushing. By introducing this wrapper we hope to avoid duplication across defense unicorn repositories and simplify the adoption of OCI packaging across the company. 

Another reason for separating the code out is to give the code it's own release cycle. Zarf only adheres to semvar for the Zarf CLI functionality, so it can be difficult to track breaking changes in library code. Currently, UDS CLI is the only other user having to deal with these updates, however other projects in the company may create OCI packages in the future.

We also plan to break for other libraries to spin out of Zarf in the near future for similar reasons. 