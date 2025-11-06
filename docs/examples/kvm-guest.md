# Example: Run a KVM guest

It is possible to launch a Virtual Machine (VM) within a GitHub Action using nested virtualisation.

With the following caveats:

- Nested Virtualisation is generally slower than bare-metal.
- Arm is not supported by the Linux Kernel or the current generation of Arm SoCs.

The `kvm` and `kvm_intel` / `kvm_amd` modules are shipped with the actuated Kernel, so you can load them in your workflow via `sudo modprobe`.

## Example: Run a Firecracker microVM

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