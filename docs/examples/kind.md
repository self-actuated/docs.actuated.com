# Example: Kubernetes with KinD

You may need to access Kubernetes within your build. [KinD](https://kind.sigs.k8s.io) is a popular option, and easy to run in an action.

Certified for:

- [x] `x86_64`
- [x] `arm64` including Raspberry Pi 4

!!! info "Use a private repository if you're not using actuated yet"
    GitHub recommends using a private repository with self-hosted runners because changes can be left over from a previous run, even when using Actions Runtime Controller. Actuated uses an ephemeral VM with an immutable image, so can be used on both public and private repos. Learn why in the [FAQ](/faq).

## Try out the action on your agent

Create a new file at: `.github/workflows/build.yml` and commit it to the repository.

Note that it's important to make sure Kubernetes is responsive before performing any commands like running a Pod or installing a helm chart.

```yaml
name: build

on: push
jobs:
  start-kind:
    runs-on: actuated-4cpu-16gb
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 1
      - name: get arkade
        uses: alexellis/setup-arkade@v1
      - name: get kubectl and kubectl
        uses: alexellis/arkade-get@master
        with:
          kubectl: latest
          kind: latest
      - name: Create a KinD cluster
        run: |
          mkdir -p $HOME/.kube/
          kind create cluster --wait 300s
      - name: Wait until CoreDNS is ready
        run: |
          kubectl rollout status deploy/coredns -n kube-system --timeout=300s
      - name: Explore nodes
        run: kubectl get nodes -o wide
      - name: Explore pods
        run: kubectl get pod -A -o wide
      - name: Show kubelet logs
        run: docker exec kind-control-plane journalctl -u kubelet
```

To run this on ARM64, just change the actuated prefix from `actuated-` to `actuated-arm64-`.

### Using a registry mirror for KinD

Whilst the instructions for a [registry mirror](/tasks/registry-mirror) work for Docker, and for buildkit, KinD uses its own containerd configuration, so needs to be configured separately, as required.

When using KinD, if you're deploying images which are hosted on the Docker Hub, then you'll probably need to either: authenticate to the Docker Hub, or configure the registry mirror running on your server.

Here's an example of how to create a KinD cluster, using a registry mirror for the Docker Hub:

```bash
#!/bin/bash

kind create cluster --wait 300s --config /dev/stdin <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
containerdConfigPatches:
- |-
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
    endpoint = ["http://192.168.128.1:5000"]
EOF
```

With open source projects, you may need to run the build on GitHub's hosted runners some of the time, in which case, you can use a check whether the mirror is available:

```bash
curl -f --connect-timeout 0.1 -s http://192.168.128.1:5000/v2/_catalog &> /dev/null

if [ "$?" == "0" ]
then
  echo "Mirror found, configure KinD for the mirror"
else
  echo "Mirror not found, use defaults"
fi
```

To use authentication instead, create a Kubernetes secret of type `docker-registry` and then attach it to the default service account of each namespace within your cluster.

The OpenFaaS docs show how to do this for private registries, but the same applies for authenticating to the Docker Hub to raise rate-limits.

You may also like Alex's [alexellis/registry-creds](https://github.com/alexellis/registry-creds) project which will replicate your Docker Hub credentials into each namespace within a cluster, to make sure images are pulled with the correct credentials.
