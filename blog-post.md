## What is OCI and ORAS

OCI, open container initative, is an initative that decoupled containers from docker. Similar to how kubernetes is no longer just about managing images, but is now also seen as a generic resource manager, OCI artifacts are no longer limited to container images, but now can hold generic OCI artifacts. An OCI artifact can be any set of files so long as they're made to follow the [OCIspec](https://github.com/opencontainers/image-spec)

There are a ton of benefits that come with storing OCI images. The community wanted an easy way to implement OCI for generic artifacts. This birthed a project called [ORAS](https://oras.land/), OCI registry as storage. ORAS makes it simpler to store OCI artifacts in online registries such as ghcr, ecr, or gitlab packages (well kinda gitlab packages, but more on that later). ORAS provides a cli and library for pushing OCI artifacts to registries.

## Why OCI
You might be thinking, we even use OCI for Zarf? You can just curl up the tarball and curl them back down. Let's look at the benefits:
- Re use of existing registries. Tools in the k8s ecosystem like Zarf and helm know users are likely to have a registry so reuse of this registry is convienent.
  - This means we can use the same authentication system as well.
- Layers! When you downlod a Zarf package if you already have the image in your Zarf cache we can skip downloading it. 
  - Theoretically, we could do the same for components. We don't as we want to avoid bloating users system with huge blob files. Although, registries get to take advantage of components as OCI layers which saves storage space for any components that are unchanged betweeen architectures or versions.
- Better user experience. 
  - Users don't need to specify which architecture they are working in, clients for OCI regestries generally will detect the architecture on your computer and automatically download it for you.
  - Organization. When we tag images with a version registries generally know to order and organize artifacts to so we can see how version change over time.
- Built in signing, if any of the files you mean to download somehow change during the download process, through man in the middle attack or error, ORAS will automatically detect this and fail the download.

## What is a Zarf OCI package

Here is a diagram for what an OCI package in Zarf looks like under the hood.

End users will never have to interact with the indexes, manifests, or manifest configs. Zarf, ORAS, and the handle the standard OCI files so users can simply run a command like `Zarf package pull oci://🦄/init:v0.32.3` <- The 🦄 really works! 

Let's run through an example of what these files do. If we pull this package from an arm64 computer Zarf will check the index for a manifest with the platform arm64. If an arm64 manifest exists it will pull down that manifest. The manifest consists of a list of poitners to blob files which make up the meat of our Zarf package. Zarf puts then packages those files together into a regular package and sends it straight to your current directory! Or another directory or registry if specified through the --output flag.

You might notice in the above diagram that we have image indexs, manifests, and layers. Another neat aspect about OCI is that we can store OCI components within other OCI components. ORAS will detect this happening and resolve everythign properly into their binary. This is especially important to us as we have uds bundles which make up Zarf packages allowing for the same behavior. ?! <- Need to make sure UDS-cli treats Zarf in this same way before I say this. 

## Why OCI is becoming a library
We are currently refactoring Zarf and will soon release an OCI library to be used across defense unicorns. This library is based on current Zarf code but is decoupled from Zarf. Zarf will still have it's own library, but this library is now heavily opinionated towards Zarf. 

You may wonder why we even need this library since ORAS already exists. In short, the ORAS library is complicated to use and unopiniated. We aim to make easier to interact with oci primitives such as manifests, indexes, and layers. Thereby simplifying common operations like copying, pulling and pushing. Additionally, we have small helper functions that we don't want to
duplicate across projects. By introducing this wrapper with we can simplfy adoption of OCI packaging across the company. Another reason for this change is since Zarf only adheres to semvar for the Zarf cli functionality it can be difficult to track when breaking changes occur. UDS cli is the only user of the Zarf OCI library currently, but more projects in the company may create OCI packages in the future and we'd like to avoid churn from unexpected updates. For this same reason you can expect future common libraries to spin out of zarf and be used company wide. 