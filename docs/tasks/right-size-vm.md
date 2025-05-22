# Task: Right size a VM by profiling a job

Use-cases:

* Reduce costs by using smaller VMs that fit the workload better
* Increase density by running more jobs on the same hardware
* Improve performance bottlenecks by adding more resources when needed

In the blog post [Right sizing VMs for GitHub Actions](https://actuated.com/blog/right-sizing-vms-github-actions), we introduce the need for profiling (or metering) a running job in order to make sure the VM is the right size for the workload.

A range of metrics are collected in addition to the standard ones like CPU & RAM consumption, vmmeter also shows contention on I/O, whether a job is running out of disk apce, and how many open files are in use. Non-obvious metrics like entropy and I/O contention are also collected, which can also be linked to degraded performance.

See also: [Custom VM sizes](/examples/custom-vm-size/)

## Try it out

Add the following to the top of your workflow file:

```diff
name: ci
jobs:
  my-job-1:
    name: my-job-1
    runs-on: actuated-4cpu-12gb
    steps:
+       # vmmeter start
+       - uses: alexellis/setup-arkade@master
+       - uses: self-actuated/vmmeter-action@master
+       # vmmeter end
```

The `vmmeter-action` will run in the background and collect metrics about the job. Its *Post run action* will collect the metrics and upload them to the job's summary.

Here is an example from building a Linux Kernel:

```bash
Total RAM: 61.2GB
Total vCPU: 32
Load averages:
Max 1 min: 5.76 (18.00%)
Max 5 min: 1.34 (4.19%)
Max 15 min: 0.44 (1.38%)


RAM usage (10 samples):
Max RAM usage: 2.348GB


Max 10s avg RAM usage: 1.788GB
Max 1 min avg RAM usage: 1.233GB
Max 5 min avg RAM usage: 1.233GB


Disk read: 405.4MB
Disk write: 469.6MB
Max disk I/O inflight: 0
Free: 45.16GB	Used: 4.66GB	(Total: 52.52GB)


Egress adapter RX: 275.6MB
Egress adapter TX: 1.341MB


Entropy min: 256
Entropy max: 256


Max open connections: 23
Max open files: 1856
Processes since boot: 19819


Run time: 47s
```
