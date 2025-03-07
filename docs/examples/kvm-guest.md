# Example: Run a KVM guest

It is possible to launch a Virtual Machine (VM) within a GitHub Action. Support for virtualization is not enabled by default for Actuated. The Agent has to be configured to use a custom kernel.

There are some prerequisites to enable KVM support:

- `aarch64` runners are not supported at the moment.
- A bare-metal host for the Agent is required.


!!! info "Nested virtualisation is a premium feature"
    This feature requires a plan size of 15 concurrent builds or greater, however you can get a 14-day free trial by contacting our team directly through the actuated Slack.

## Configure the Agent

1. Make sure [nested virtualization is enabled](https://ostechnix.com/how-to-enable-nested-virtualization-in-kvm-in-linux/) on the Agent host.

2. Edit `/etc/default/actuated` on the Actuated Agent and add the `kvm` suffix to the `AGENT_KERNEL_REF` variable:

    ```diff
    - AGENT_KERNEL_REF="ghcr.io/openfaasltd/actuated-kernel:x86_64-latest"
    + AGENT_KERNEL_REF="ghcr.io/openfaasltd/actuated-kernel:x86_64-kvm-latest"
    ```

3. Also add it to the `AGENT_IMAGE_REF` line:

    ```diff
    - AGENT_IMAGE_REF="ghcr.io/openfaasltd/actuated-ubuntu22.04:x86_64-latest"
    + AGENT_IMAGE_REF="ghcr.io/openfaasltd/actuated-ubuntu22.04:x86_64-kvm-latest"
    ```

3. Restart the Agent to use the new kernel.

    ```bash
    sudo systemctl daemon-reload && \
        sudo systemctl restart actuated
    ```

4. Run a [test build](/test-build/) to verify KVM support is enabled in the runner. The specs script from the test build will report whether `/dev/kvm` is available.

## Run a Firecracker microVM

This example is an adaptation of the [Firecracker quickstart guide](https://github.com/firecracker-microvm/firecracker/blob/main/docs/getting-started.md) that we run from within a GitHub Actions workflow.

The workflow instals Firecracker, configures and boots a guest VM and then waits 20 seconds before shutting down the VM and exiting the workflow.

1. Create a new repository and add a workflow file.

    The workflow file: `./.github/workflows/vm-run.yaml`:

    ```yaml
    name: vm-run

    on: push
    jobs:
    vm-run:
        runs-on: actuated-4cpu-8gb
        steps:
        - uses: actions/checkout@v4
            with:
            fetch-depth: 1
        - name: Install arkade
            uses: alexellis/setup-arkade@v2
        - name: Install firecracker
            run: sudo arkade system install firecracker
        - name: Run microVM
            run: sudo ./run-vm.sh
    ```

2. Add the `run-vm.sh` script to the root of the repository.

    Running the script will:

    - Get the kernel and rootfs for the microVM
    - Start fireckracker and configure the guest kernel and rootfs
    - Start the guest machine
    - Wait for 20 seconds and kill the firecracker process so workflow finishes.

    The `run-vm.sh` script:

    ```bash
    #!/bin/bash

    # Clone the example repo
    git clone https://github.com/skatolo/nested-firecracker.git

    # Run the VM script
    ./nested-firecracker/run-vm.sh 
    ```

4. Hit commit and check the run logs of the workflow. You should find the login prompt of the running microVM in the logs.

The full example is available on [GitHub](https://github.com/skatolo/nested-firecracker)

For more examples and use-cases see:

- [How to run a KVM guest in your GitHub Actions](https://actuated.com/blog/kvm-in-github-actions)