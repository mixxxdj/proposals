# Move VCPK buildenv artifact to Github Release assets

* **Owners:**
  * `@acolombier`

* **Implementation Status:** `Not implemented`

* **Related Issues and PRs:**
  * [CI footprint reduction](https://github.com/mixxxdj/mixxx/issues/14297)

> TL;DR: Reduce the bandwidth footprint induced by our VCPKG archive distribution setup

## Why

Reduce the amount of used bandwidth to distribute our buildenv VCPKG artifacts, optimize the delivery performance and
simplify the developer experience by providing more transparency

### Pitfalls of the current solution

Currently, the VCPKG buildenv artifacts are stored on Mixxx download server. This server appears to be located in
France, and is protected by Cloudflare. Our biggest user of the VCPKG buildenv is our own CI, located in US.

This means that  our current setup induce many roundtrip over [the pond](https://en.wikipedia.org/wiki/Atlantic_Ocean):
The VCPKG CI would upload the artifact from the US, where the Github runner public infrastructure is (mostly) located,
latter other Mixxx CI would downloading it. Since our download server is served via CloudFlare, it is likely being
cached at the edges closer to the GH infrastructure but there is still gain in removing that extra hop and store
everything within GH blob storage. From a developer perspective, centralizing our artifact as part of the existing GH
CDN infrastructure would also reduce the footprint of CDNs.

Unfortunately, unlike European organizations, Github and CLoudFlare don't have requirement to communicate efficiently
and be transparent about the actual footprint of the resource consumption. THis means that while on paper, this
proposal would significantly reduce the footprint, we have no way to quantify it and ensure it's exact value.

## Goals

The goal here is the solely reduce the footprint. Any side gains (e.g visibility of the VCPK release in the UI) is
only nice to have and shouldn't be the driver for this proposal outcome.

### Audience

Any Mixxx enthusiast and contributor is welcome to share their though and feedback on this proposal

## Non-Goals

* Change how our buildenv scripts work
* Induce changes on Mixxx CI
* Move Mixxx build artifact (MSI) off the download server

## How

Here is the proposed plan to address this:

1. Create a release and store VCPKG assets as part of the main VCPKG CI
2. Update the buildenv script to fetch from GH instead by default, but still allow download from `download.mixxx.org`
