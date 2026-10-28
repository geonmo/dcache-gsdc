# GSDC dCache site build

This repository is not part of dCache. It holds the site patches that GSDC applies
on top of official dCache releases, and the workflow that builds them:

| Path | Purpose |
|---|---|
| `patches/*.patch` | Site patches, applied with `git am` on top of the release tag |
| `container/overlay.txt` | Files of the patched build that replace those in the official image |
| `container/Dockerfile` | Official `dcache/dcache:<version>` image plus the files above |
| `container/prepare-overlay.sh` | Prepares the container build context from the patched tarball |
| `.github/workflows/gsdc-build.yml` | Builds RPM and image for new releases of the tracked series |

## What gets built

When a new `11.2.x` image appears on Docker Hub (checked daily, or started by hand
from the *Actions* tab), the workflow builds:

- the RPM `dcache-<version>-<N>_GSDC.noarch.rpm`, attached to the GitHub release
  `<version>-<N>_GSDC`
- the image `ghcr.io/geonmo/dcache-gsdc:<version>-<N>_GSDC`, and the moving tag
  `ghcr.io/geonmo/dcache-gsdc:11.2-gsdc` for the newest release

`<N>` starts at 2, as official RPMs are always `-1`. Running the workflow by
hand with *force* builds the next number (e.g. `-3_GSDC` after `-2_GSDC`), so
earlier site builds stay available.

An official release with a higher version (e.g. `11.2.11-1`) replaces the site
build when installed. If the patches no longer apply to a new release, the
workflow opens an issue instead of building.

## Updating the patches

```console
git -C dcache checkout -b site/<version> <version>
git -C dcache am /path/to/this/branch/patches/*.patch   # fix conflicts, if any
git -C dcache format-patch -o /path/to/this/branch/patches <version>..HEAD
```

If a patch touches another module or defaults file, add it to
`container/overlay.txt`.

## Running with podman quadlet

```ini
# /etc/containers/systemd/dcache.container
[Container]
Image=ghcr.io/geonmo/dcache-gsdc:11.2.10-2_GSDC
# Alternatively, follow the newest site build with `podman auto-update`:
# Image=ghcr.io/geonmo/dcache-gsdc:11.2-gsdc
# AutoUpdate=registry
```

The official image is based on AlmaLinux 10, which needs an x86-64-v3 CPU
(AVX2). On older CPUs the container fails with
`Fatal glibc error: CPU does not support x86-64-v3`.
