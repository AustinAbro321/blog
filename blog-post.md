## What is OCI and ORAS

The Open Container Initiative (OCI) is an open specification for creating industry standards around container formats and runtimes. OCI was originally invented to decouple container images from Docker. Over time OCI became generic enough to be used for any type of content as long as it follows the [OCI specification](https://github.com/opencontainers/image-spec). Similar to how Kubernetes is no longer thought of as just a container image manager, and is now a generic resource manager, OCI artifacts are no longer limited to being container images, and can now be any digital file following the OCI specification.

In order to take advantage of the benefits of this specification, a community project called [ORAS](https://oras.land/) (OCI Registry as Storage) was created. ORAS simplifies the process of storing OCI artifacts in online registries such as [GHCR](https://github.com/features/packages) or [ECR](https://aws.amazon.com/ecr/) by providing a CLI and library for pushing artifacts to registries. ORAS is generally not intended to be used by kubernetes operators, it's intent is to make it easy for products like Zarf and Helm to have built in features for interacting with OCI registries.

## Why OCI

At first glance, it may seem sufficient to just cURL up and down Zarf packages as they are always tarballs. Let's look at the benefits of using OCI:

*   Reuse of existing registries. 
    * Tools in the Kubernetes ecosystem like Zarf and Helm likely have users with an OCI registry already setup.
    * Users can use their existing authentication for their registry.
*   Layers! When a Zarf package is downloaded if the image already exists in the cache downloading is skipped.
    *   Theoretically, we could do the same for components, but we want to avoid bloating users' systems with huge files. However, registries get to take advantage of components as OCI layers which saves storage space for any components that are unchanged between architectures or versions.
*   Better user experience.
    *   Users don't need to specify which architecture they are working in, clients for OCI registries generally will detect the architecture on users' computer and automatically download the correct package.
    *   Registries group together packages by name making it easy to scroll through version and architectures
    *   ORAS runs in go, so generally tools implementing OCI work cross platform. Users no longer need to maintain a set of cURL scripts that don't work on Windows, they can simply run `zarf package pull` or `zarf package publish`.
*   Built-in integrity checks. 
    *   ORAS ensures that if any files are altered or corrupted during the download process, whether through a man-in-the-middle attack or an error, the discrepancy will be detected and the download will be cancelled.

## What is a Zarf OCI package

Here is a diagram of what an OCI package in Zarf looks like under the hood.

End users will never have to interact with the indexes, manifests, or manifest configs. Zarf, ORAS, and the registry handle the standard OCI files.

If we pull this package from an arm64 computer Zarf will check the index for a manifest with the platform arm64. If an arm64 manifest exists it will pull down that manifest. The manifest contains a list of files that Zarf uses to create the package.

One might notice in the above diagram that we have image indexes and image manifests as layers. This means that within our OCI packages we can have other OCI artifacts, usually in zarfs' case it's a container image. ORAS detects this OCIception and resolves everything properly.

Zarf has a unique type of OCI artifact called skeletons. A skeleton package is a bare-bones Zarf package definition alongside its associated local files. These packages are intended for use with component composability to provide versioned imports for components that you wish to mix and match or modify with merge-overrides across packages. Here's an example of a package that imports a skeleton https://github.com/defenseunicorns/zarf/tree/main/examples/composable-packages


## Why OCI is becoming a library
We are currently refactoring Zarf and will soon release an OCI library to be used across Defense Unicorns. It may seem strange that we are maintaining an OCI library since ORAS already exists. In short, the ORAS library is complicated to use and unopinionated. Our library aims to make it easier to interact with OCI primitives such as manifests, indexes, and layers. This simplifies common operations like copying, pulling, and pushing. By introducing this wrapper we hope to avoid duplication across Defense Unicorn projects and simplify the adoption of OCI packaging across the company. 

Another reason for separating the code out is to give the code it's own release cycle. Zarf only adheres to semver for the Zarf CLI functionality, so it can be difficult to track breaking changes in the Zarf library code. Currently, UDS CLI is the only user having to deal with these updates, however other projects in the company may create OCI packages in the future.

We plan to create other libraries out of Zarf in the near future for similar reasons. 