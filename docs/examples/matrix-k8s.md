# Example: Regression test against various Kubernetes versions

This example launches multiple Kubernetes clusters in parallel for regression and end to end testing.

In the example, We're testing the CRD for the [inlets-operator](https://github.com/inlets/inlets-operator) on versions v1.16 through to v1.25. You could also switch out k3s for KinD, if you prefer.

See also: [Actuated with KinD](/examples/kind)

![Launching 10 Kubernetes clusters in parallel](/images/k3s-matrix.png)
> Launching 10 Kubernetes clusters in parallel across your fleet of Actuated Servers.

Certified for:

- [x] `x86_64`
- [x] `arm64` including Raspberry Pi 4

!!! info "Use a private repository if you're not using actuated yet"
    GitHub recommends using a private repository with self-hosted runners because changes can be left over from a previous run, even when using Actions Runtime Controller. Actuated uses an ephemeral VM with an immutable image, so can be used on both public and private repos. Learn why in the [FAQ](/faq).

## Try out the action on your agent

Create a new file at: `.github/workflows/build.yml` and commit it to the repository.

Customise both the array "k3s" with the versions you need to test and replace the step "Test crds" with whatever you need to install such as helm charts.

```yaml
name: k3s-test-matrix

on:
  pull_request:
    branches:
      - '*'
  push:
    branches:
      - master
      - main

jobs:
  kubernetes:
    name: k3s-test-${{ matrix.k3s }}
    runs-on: actuated-4cpu-12gb
    strategy:
      matrix:
        k3s: [v1.16, v1.17, v1.18, v1.19, v1.20, v1.21, v1.22, v1.23, v1.24, v1.25]

    steps:
      - uses: actions/checkout@v4
      - uses: alexellis/setup-arkade@v2
      - uses: alexellis/arkade-get@master
        with:
          kubectl: latest
          k3sup: latest

      - name: Create Kubernetes ${{ matrix.k3s }} cluster
        run: |
          mkdir -p $HOME/.kube/
          k3sup install \
            --local \
            --k3s-channel ${{ matrix.k3s }} \
            --local-path $HOME/.kube/config \
            --merge \
            --context default
          cat $HOME/.kube/config
          
          k3sup ready --context default
          kubectl config use-context default
          
          # Just an extra test on top.
          echo "Waiting for nodes to be ready ..."
          kubectl wait --for=condition=Ready nodes --all --timeout=5m
          kubectl get nodes -o wide

      - name: Test crds
        run: |
          echo "Applying CRD"
          kubectl apply -f https://raw.githubusercontent.com/inlets/inlets-operator/master/artifacts/crds/inlets.inlets.dev_tunnels.yaml
```

The matrix will cause a new VM to be launched for each item in the "k3s" array.

