# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single Dockerfile that layers GNU libc (glibc) onto an otherwise musl-only
Alpine base, and sets `C.UTF-8` as the default locale. There is no app code; the
repo is the `Dockerfile` plus a GHCR publish workflow.

## Base image dependency (cross-repo)

`FROM ghcr.io/liskl/alpine-base:3.24` — this image is built on the sibling
`liskl/alpine-base` repo's published image (a `FROM scratch` Alpine minirootfs).
The `:3.24` tag pins it to the Alpine 3.24 line. If alpine-base publishes a new
Alpine version, this tag must be bumped deliberately; it does not auto-track.

## How glibc gets installed (and why it's fragile)

Alpine is musl-based; glibc is added via the third-party
[`sgerrand/alpine-pkg-glibc`](https://github.com/sgerrand/alpine-pkg-glibc)
packages, downloaded as `.apk` files from that project's GitHub releases and
installed locally. Three things here are load-bearing and have each broken before:

- **`ALPINE_GLIBC_PACKAGE_VERSION`** (currently `2.35-r1`) is the sgerrand release
  version, pinned independently of the Alpine base version. Old pins (e.g.
  `2.25-r0`) are no longer downloadable.
- **Signing key URL** is `https://alpine-pkgs.sgerrand.com/sgerrand.rsa.pub`. The
  older `raw.githubusercontent.com/.../sgerrand.rsa.pub` URLs are dead (404) —
  don't revert to them.
- **`apk add --force-overwrite`** is REQUIRED, not optional. On modern Alpine the
  glibc packages pull in `gcompat`, which also wants to own
  `/lib/ld-linux-x86-64.so.2`. Without `--force-overwrite`, the install aborts
  with a file-conflict error. Removing this flag will break the build.

The `RUN` is a single chained layer that installs build deps (`wget`,
`ca-certificates`) as a virtual package, fetches and installs glibc, generates
the `C.UTF-8` locale, then removes the key, the `.apk` files, and the build deps
to keep the layer small. Edits must preserve that cleanup tail or the image bloats.

Architecture is x86_64-specific (the `ld-linux-x86-64.so.2` conflict and the
glibc `.apk` builds are amd64).

## Commands

```bash
# Build
docker build -t alpine-glibc:test .

# Verify glibc is present and working (expect: ldd (GNU libc) 2.35)
docker run --rm alpine-glibc:test /usr/glibc-compat/bin/ldd --version

# Verify default locale (expect: C.UTF-8) and base Alpine release
docker run --rm alpine-glibc:test sh -c 'echo $LANG'
docker run --rm alpine-glibc:test cat /etc/alpine-release
```

## CI / publishing

`.github/workflows/publish.yml` builds and publishes to
`ghcr.io/liskl/alpine-glibc`. It pushes on push to `master`, on `v*` tags, and on
manual dispatch; pull requests build only (no push) as a CI check. The image is
tagged with the glibc version greped from the Dockerfile
(`ALPINE_GLIBC_PACKAGE_VERSION`), plus `latest` (default branch only), the git
tag for `v*` pushes, and a short commit SHA. Auth uses the built-in
`GITHUB_TOKEN`. Build platform is `linux/amd64` only.

## Bumping glibc

Check the latest release at
<https://github.com/sgerrand/alpine-pkg-glibc/releases>, update
`ALPINE_GLIBC_PACKAGE_VERSION`, and rebuild. The filenames
(`glibc-`, `glibc-bin-`, `glibc-i18n-` + version + `.apk`) derive from that one
variable, so it's the only value to change.
