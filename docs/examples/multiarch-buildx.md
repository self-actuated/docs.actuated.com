# Example: Multi-arch with buildx

A multi-arch or multi-platform container is effectively where you build the same container image for multiple different Operating Systems or CPU architectures, and link them together under a single name.

So you may publish an image named: `ghcr.io/inlets-operator/latest`, but when this image is fetched by a user, a manifest file is downloaded, which directs the user to the appropriate image for their architecture.

If you'd like to see what these look like, run the following with [arkade](https://arkade.dev):

```bash
arkade get crane

crane manifest ghcr.io/inlets/inlets-operator:latest
```

You'll see a manifests array, with a platform section for each image:

```json
{
  "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",
  "manifests": [
    {
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "digest": "sha256:bae8025e080d05f1db0e337daae54016ada179152e44613bf3f8c4243ad939df",
      "platform": {
        "architecture": "amd64",
        "os": "linux"
      }
    },
    {
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "digest": "sha256:3ddc045e2655f06653fc36ac88d1d85e0f077c111a3d1abf01d05e6bbc79c89f",
      "platform": {
        "architecture": "arm64",
        "os": "linux"
      }
    }
  ]
}
```

## Try an example

This example is taken from the Open Source [inlets-operator](https://github.com/inlets/inlets-operator).

It builds a container image containing a Go binary and uses a Dockerfile in the root of the repository. All of the images and corresponding manifest are published to GitHub's Container Registry (GHCR). The action itself is able to authenticate to GHCR using a built-in, short-lived token. This is dependent on the "permissions" section and "packages: write" being set.

View [publish.yaml](https://github.com/inlets/inlets-operator/blob/master/.github/workflows/publish.yaml), adapted for actuated:

```diff
name: publish

on:
  push:
    tags:
      - '*'

jobs:
  publish:
+    permissions:
+      packages: write

-   runs-on: ubuntu-latest
+   runs-on: actuated-4cpu-12gb
    steps:
      - uses: actions/checkout@master
        with:
          fetch-depth: 1

+     - name: Setup mirror
+       uses: self-actuated/hub-mirror@master
      - name: Get TAG
        id: get_tag
        run: echo TAG=${GITHUB_REF#refs/tags/} >> $GITHUB_ENV
      - name: Get Repo Owner
        id: get_repo_owner
        run: echo "REPO_OWNER=$(echo ${{ github.repository_owner }} | tr '[:upper:]' '[:lower:]')" > $GITHUB_ENV

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      - name: Login to container Registry
        uses: docker/login-action@v3
        with:
          username: ${{ github.repository_owner }}
          password: ${{ secrets.GITHUB_TOKEN }}
          registry: ghcr.io

      - name: Release build
        id: release_build
        uses: docker/build-push-action@v5
        with:
          outputs: "type=registry,push=true"
          platforms: linux/amd64,linux/arm/v6,linux/arm64
          build-args: |
            Version=${{  env.TAG }}
            GitCommit=${{ github.sha }}
          tags: |
            ghcr.io/${{ env.REPO_OWNER }}/inlets-operator:${{ github.sha }}
            ghcr.io/${{ env.REPO_OWNER }}/inlets-operator:${{ env.TAG }}
            ghcr.io/${{ env.REPO_OWNER }}/inlets-operator:latest
```

You'll see that we added a `Setup mirror` step, this explained in the [Registry Mirror example](/tasks/registry-mirror)

The `docker/setup-qemu-action@v3` step is responsible for setting up QEMU, which is used to emulate the different CPU architectures.

The `docker/build-push-action@v5` step is responsible for passing in a number of platform combinations such as: `linux/amd64` for cloud, `linux/arm64` for Arm servers and `linux/arm/v6` for Raspberry Pi.

Within [the Dockerfile](https://github.com/inlets/inlets-operator/blob/master/Dockerfile), we needed to make a couple of changes.

You can pick to run the step in either the BUILDPLATFORM or TARGETPLATFORM. The BUILDPLATFORM is the native architecture and platform of the machine performing the build, this is usually amd64. The TARGETPLATFORM is important for the final step of the build, and will be injected based upon one each of the platforms you have specified in the step.

```diff
- FROM golang:1.22 as builder
+ FROM --platform=${BUILDPLATFORM:-linux/amd64} golang:1.22 as builder
```

For Go specifically, we also updated the `go build` command to tell Go to use cross-compilation based upon the TARGETOS and TARGETARCH environment variables, which are populated by Docker.

```diff
GOOS=${TARGETOS} GOARCH=${TARGETARCH} go build -o inlets-operator
```

Learn more in the Docker Documentation: [Multi-platform images](https://docs.docker.com/build/building/multi-platform/)

## Is it slow to build for Arm?

Using QEMU can be slow at times, especially when building an image for Arm using a hosted GitHub Runner.

We found that we could increase an Open Source project's build time by 22x - from ~ 36 minutes to 1 minute 26 seconds.

See also [How to make GitHub Actions 22x faster with bare-metal Arm](https://actuated.com/blog/native-arm64-for-github-actions)

To build a separate image for Arm on an Arm runner, and one for amd64, you could use a [matrix build](/examples/matrix).

## Need a hand with GitHub Actions?

Check your plan to see if access to Slack is included, if so, you can contact us on Slack for help and guidance.
