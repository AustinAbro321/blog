## What is OCI and ORAS

OCI or open container initiative is an initiative that decoupled containers from docker. Similar to how kubernetes is no longer just about managing images, but is now also seen as a generic resource manager, OCI artifacts are no longer limited to container images, but now can hold generic OCI artifacts. An OCI artifact can be any set of files so long as they're made to follow the [ocispec](https://github.com/opencontainers/image-spec)

There are a ton of benefits that come with storing OCI images. The community wanted an easy way to implement OCI for generic artifacts. This birthed a project called [ORAS](https://oras.land/), OCI registry as storage. ORAS makes it simpler to store OCI artifacts in online registries such as GHCR, or ECR. ORAS provides a CLI and library for pushing OCI artifacts to registries.

## Why OCI
You might be thinking, we even use OCI for Zarf? You can just curl up the tarballs and curl them back down. Let's look at the benefits:
- Re-use of existing registries. Tools in the k8s ecosystem like Zarf and Helm know users are likely to have a registry so reuse of this registry is convenient.
  - This means we can use the same authentication system as well.
- Layers! When you download a Zarf package if you already have the image in your Zarf cache we can skip downloading it. 
  - Theoretically, we could do the same for components, but we don't to avoid bloating users' systems with huge blob files. However, registries get to take advantage of components as OCI layers which saves storage space for any components that are unchanged between architectures or versions.
- Better user experience. 
  - Users don't need to specify which architecture they are working in, CLIents for OCI registries generally will detect the architecture on your computer and automatically download it for you.
  - Organization. When users tag images with a version registries generally know how to order and organize artifacts so we can users can see how versions change over time.
- Built-in signing, if any of the files you mean to download somehow change during the download process, through a man in the middle attack or error, ORAS will automatically detect this and fail the download.

## What is a Zarf OCI package

Here is a diagram of what an OCI package in Zarf looks like under the hood.

End users will never have to interact with the indexes, manifests, or manifest configs. Zarf, ORAS, and the handle the standard OCI files so users can simply run a command like `Zarf package pull oci://🦄/init:v0.32.3` <- The 🦄 really works! 

Let's run through an example of what these files do. If we pull this package from an arm64 computer Zarf will check the index for a manifest with the platform arm64. If an arm64 manifest exists it will pull down that manifest. The manifest consists of a list of pointers to blob files that make up the meat of our Zarf package. Zarf then takes those files to create a regular zarf package and sends it straight to your current directory! Or another directory or registry if specified through the --output flag.

You might notice in the above diagram that we have image indexes, manifests, and layers. Another neat aspect of OCI is that we can store OCI components within other OCI components. ORAS will detect this happening and resolve everything properly into their binary. This is especially important to us as we have UDS bundles that make up Zarf packages allowing for the same behavior. ?! <- Need to make sure UDS-CLI treats Zarf in this same way before I say this. 

## Why OCI is becoming a library
We are currently refactoring Zarf and will soon release an OCI library to be used across defense unicorns. You may wonder why we need this library since ORAS already exists. In short, the ORAS library is complicated to use and unopiniated. We aim to make it easier to interact with OCI primitives such as manifests, indexes, and layers. Thereby simplifying common operations like copying, pulling, and pushing. Additionally, we have small helper functions that we don't want to duplicate across projects. By introducing this wrapper we can simplify the adoption of OCI packaging across the company. Another reason for this change is since Zarf only adheres to semvar for the Zarf CLI functionality it can be difficult to track when breaking changes occur. UDS CLI is the only user of the Zarf OCI library currently, but more projects in the company may create OCI packages in the future and we'd like to avoid churn from unexpected updates. For this same reason, you can expect future common libraries to spin out of Zarf to be used across products. 