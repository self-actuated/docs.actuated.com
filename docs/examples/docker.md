# Example: Kubernetes with KinD

[Docker CE](https://docker.io) is preinstalled in the actuated VM image, and will start upon boot-up.

Certified for:

- [x] `x86_64`
- [x] `arm64` including Raspberry Pi 4

!!! info "Use a private repository if you're not using actuated yet"
    GitHub recommends using a private repository with self-hosted runners because changes can be left over from a previous run, even when using Actions Runtime Controller. Actuated uses an ephemeral VM with an immutable image, so can be used on both public and private repos. Learn why in the [FAQ](/faq).

## Try out the action on your agent

Create a new file at: `.github/workflows/build.yml` and commit it to the repository.

Try running a container to ping Google for 3 times:

```yaml
name: build

on: push
jobs:
  ping-google:
    runs-on: actuated-4cpu-16gb
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 1
      - name: Run a ping to Google with Docker
        run: |
          docker run --rm -i alpine:latest ping -c 3 google.com
```

Build a container with Docker:

```yaml
name: build

on: push
jobs:
  build-in-docker:
    runs-on: actuated-4cpu-16gb
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 1
      - name: Build inlets-connect using Docker
        run: |
          git clone --depth=1 https://github.com/alexellis/inlets-connect
          cd inlets-connect
          docker build -t inlets-connect .
          docker images
```

To run this on ARM64, just change the actuated prefix from `actuated-` to `actuated-arm64-`.