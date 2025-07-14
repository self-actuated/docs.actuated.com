# Example: Use a GPU in your workflow

If you have a server with one or more GPUs, you can mount them into your workflows for AI/ML workloads, or for other tasks that require a GPU.

In order to mount a GPU, you'll need to switch the server from Firecracker to Cloud Hypervisor. You can reach out to us to configure this for you, or to provide instructions on how to do it yourself.

Once configured, the agent will detect all available GPUs on start-up and put them into a GPU pool ready to be mounted via VFIO into jobs that request them.

GPUs are allocated wholely and exclusively for the duration of a job before being returned to the pool.

When using consumer-grade hardware, you may need to enable IOMMU in your BIOS settings to make sure the GPU isn't used by the host system, we can help and advise you with this if needed. Most server-grade hardware will have this enabled by default. Make sure the NVIDIA drivers are not installed on the host system.

### Example workflow

To schedule the job to a host with a GPU, edit the `runs-on` label such as `gpu` i.e. `runs-on: [actuated-16cpu-32gb, gpu]`.

Our own [self-actuated/nvidia-run@master](https://github.com/self-actuated/nvidia-run) action can be used to install the Nvidia drivers to the microVM, which can also be cached for much quicker job times.

Example of how to install the Nvidia drivers and run `nvidia-smi` in a job:

```yaml
name: gpu-job

on:
    pull_request:
        branches:
        - '*'
    push:
        branches:
        - master
        - main
    workflow_dispatch:

jobs:
    gpu-job:
        name: gpu-job
        runs-on: [actuated-8cpu-16gb, gpu]
        steps:
        - uses: actions/checkout@v1
        - run: |
            lspci -kk -nn
        - uses: self-actuated/connect-ssh@master

        - uses: self-actuated/nvidia-run@master
        - name: Run nvidia-smi
          run: |
            nvidia-smi -L
            nvidia-smi
```

## Blog posts

* [Accelerate GitHub Actions with dedicated GPUs](https://actuated.com/blog/gpus-for-github-actions)
* [Run AI models with ollama in CI with GitHub Actions](https://actuated.com/blog/ollama-in-github-actions)

