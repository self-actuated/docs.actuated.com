## Provision a Server for actuated

You'll need to provision a Server which is capable of virtualisation with Linux KVM. Each of your builds will run in an isolated microVM, with its own networking, Kernel and immutable filesystem.

We have done extensive research and testing both independently and with our customers. The recommendations on this page are updated regularly. We recommend bare-metal for the best performance, but cloud VMs which support nested virtualisation are also an option.

Did you know? Bare-metal servers from European providers are available from 50-150 EUR / mo. Using your own hardware can also be really cost effective.

So what makes one server quicker than another?

* CPU Clock speed - the base and turbo speeds affect how some builds perform like Go and Rust
* Core core - The amount of vCPU allocated to a build affects multi-processing
* RAM and disk space - tune these to your needs to prevent builds slowing down
* Generation of hardware - hosted runners may use obsolete hardware, you can use the latest generation 
* Network bandwidth - how quickly images, artifacts and caches will be transferred
* Storage - NVMe is the only viable option for high performance builds
* Multi-tenancy - are other customers contenting for the same resources, or is the server dedicated to your team?

!!! info "What Operating System (OS) should I use?"
    The certified Operating System for an Actuated server is: Ubuntu Server 22.04.

## How many VMs or jobs can a server run?

Depending on the level of concurrency in your plan, each server will be able to run a set number of jobs. So we suggest dividing the RAM and CPU threads between them. For instance, if your server has 32 threads and 128GB of RAM, you could allocate 6 vCPU and 25 GB of RAM to each job for 5x jobs in parallel, or 4x vCPU and 12GB of RAM for 10x jobs in parallel.

In addition, you can also specify vCPU and RAM requirements on a per-job basis by changing the `runs-on: actuated` label to: `runs-on: actuated-2cpu-8gb` and so forth. This is useful for when you have a particular jobs which needs a lot of resources like building Kernels, Kubernetes/Docker E2E tests and browser testing.

### Just tell me what I need

For the absolute best value in terms of performance and cost, we recommend the following options from [Hetzner's](https://www.hetzner.com) Dedicated range:

* *x86_64* - [Hetzner's A102](https://www.hetzner.com/dedicated-rootserver/ax102)
* *Arm64* - [Hetzner's RX170](https://www.hetzner.com/dedicated-rootserver/matrix-rx)

Hetzner machines come with a "rescue" system, use it to install Ubuntu 22.04, and make sure you disable software RAID, so that the two NVMe drives are presented as separate devices. One will run the system, the other will be used for filesystems for all the VMs.

## Our research on servers for actuated

!!! tip "Want us to recommend a server?"
    There's a lot of options when it comes to picking a server. On the onboarding call, we can help you find a match for your performance requirements, budget, and vendor preferences.

### Intel/AMD

Intel and AMD CPUs can be used interchangeable and are known as `amd64` or `x86_64`.

1. Bare-metal on the cloud (higher cost, convenient, high performance)

    Bare-metal doesn't have to mean managing hardware in your own physical datacenter. You can deploy machines by API, pay-as-you-go and get the highest performance available.
    
    Bear in mind that whilst the cost of bare-metal is higher than VMs, you will be able to pack more builds into them and get better throughput since actuated can schedule builds much more efficiently than GitHub's self-hosted runner.

    We have seen the best performance from hosts with high clock speeds like the recent generation of AMD processors, combined with local NVMe storage. Rotational drives and SATA SSDs are significantly slower. At the lower end of bare-metal providers, you'll pay 40-50 EUR / mo per host, moving up to 80-150 EUR / mo for NVMe and AMD processors, when you go up to enterprise-grade bare-metal with 10Gbit uplinks, you'll be more in the range of 500-1500 USD / mo.

    Some providers have a setup fee, a one-month commitment, or they don't have an API/automated way to order machines. This coupled with the low costs and capacity of bare-metal means autoscaling servers is usually unnecessary.

    There are at least a dozen options for hosted bare-metal servers:
    
    - [Alibaba Cloud](https://eu.alibabacloud.com/en)
    - [AWS](https://aws.amazon.com/) - untenable pricing for bare-metal servers
    - [Berry Byte](https://berrybyte.net/dedicated/) - US region available
    - [Cherry Servers](https://www.cherryservers.com/)
    - [fasthosts](https://www.fasthosts.co.uk/)
    - [Glesys](https://glesys.com/dedicated) 
    - [Hetzner](https://hetzner.com) - Region: Germany or Finland
    - [Ionos](https://ionos.co.uk) - UK based
    - [latitude.sh](https://www.latitude.sh/) - EU and US region available 
    - [OVHcloud](https://www.ovhcloud.com/en-gb/bare-metal/rise/) - EU and US regions available
    - [PhoenixNAP](https://phoenixnap.com) - US and EU regions available 
    - [Scaleway](https://scaleway.com) - France region
    - [Vultr](https://www.vultr.com/)
    
    You can see a separate [list here](https://github.com/alexellis/awesome-baremetal#bare-metal-cloud).
    
    > A note on Scaleway: [Having tested several of Scaleway bare-metal offerings](https://twitter.com/alexellisuk/status/1605866713815437312?s=20&t=JGh5fGZJWklLTCTVkTVElg), we do not recommend their current generation of bare-metal due to slow I/O and CPU speeds.

    Are you on a budget or looking to cut costs? Both [Ionos](https://ionos.co.uk) (UK) and [Hetzner](https://hetzner.com) (Germany) have excellent value, with NVMe storage very fast AMD CPUs available.
    
    Hetzner have a minimum commitment of one month, and most of the time will also charge a one-time setup fee. We recommend their AX-Line with NVMe and ECC RAM - for instance the AX41-NVME, AX52, or AX102. The best machine on offer is the AX161 which also has a fast delivery time.

2. Cloud Virtual Machines (VMs) with nested virtualization (lowest cost, convenient, mid-level performance)

    This option may not have the raw speed and throughput of a dedicated, bare-metal host, but keeps costs low and is convenient for getting started.

    We know of at least three providers which have options for nested virtualisation: [DigitalOcean](https://m.do.co/c/8d4e75e9886f), [Google Compute Platform (GCP)](https://cloud.google.com/compute) (new customers get 300 USD free credits from GCP) support nested virtualisation on their Virtual Machines (VMs), and [Azure](https://azure.com/).'

    Nested Virtualisation is enabled by default on most clouds, however on GCP you will need to enable it via the `--enable-nested-virtualization` flag when creating a VM via `gcloud` CLI or API. This option is not listed in the GCP's web console.

    If your only option is to use AWS EC2 VMs, then we can help you get up and running using Firecracker with KVM backed by Pagetable Isolation, [learn more here](https://blog.alexellis.io/how-to-run-firecracker-without-kvm-on-regular-cloud-vms/). 

3. Bare-metal on-premises (cheap, convenient, high performance)

    Running bare-metal on-premises is a cost-effective and convenient way to re-use existing hardware investment.
    
    The machine could be racked in your server room, under your desk, or in a co-location datacenter.

    You can use [inlets to expose your agent to actuated](/expose-agent).

    Make sure you segment or isolate the agent into its own subnet, VLAN, DMZ, or VPC so that it cannot access the rest of your network. If you are thinking of running an actuated runner at home, [we have suggested iptables rules that worked well for our own testing](/expose-agent#preventing-the-runner-from-accessing-your-local-network).

### Arm

64-bit Arm is also known as both `aarch64` and `arm64`.

Arm CPUs are highly efficient when it comes to power consumption and pack in many more cores than the typical x86_64 CPU. This makes them ideal for running many builds in parallel. In typical testing, we've seen Arm builds running under emulation taking 35-45 minutes being reduced to 1-3 minutes total.

For [Fluent Bit](https://twitter.com/alexellisuk/status/1671455406097326080?s=20), a build that was failing after 6 hours using QEMU completed in just 4 minutes using actuated and an Ampere Altra server.

1. Arm on-demand, in the cloud

    For ARM64, [Hetzner](https://hetzner.com) provides outstanding value in their [RX-Line](https://www.hetzner.com/dedicated-rootserver/matrix-rx) with 128GB / 256GB RAM coupled with NVMe and 80 cores for around 200 EUR / mo. These are [Ampere Altra Servers](https://amperecomputing.com/processors/ampere-altra/). There is a minimum commitment of one month, and an initial setup cost per server.

    We have several customers running Intel/AMD and Arm builds on Hetzner who have been very happy. Stock can take anywhere between hours, days or weeks to be delivered, and could run out, so check their status page before ordering. 

    [Glesys](https://glesys.com/dedicated) have the Ampere Altra Q80-26	available for roughly €239 / mo. They are a very similar price to Hetzner and are based in Sweden.

    [PhoenixNAP](https://twitter.com/williamlbell/status/1671465178691674112?s=20) just started to stock the Ampere Altra Q80-30 as of June 2023. These can be bought on a commitment of hourly, monthly or annually with a varying discount. The range was between 600-700 USD / mo.
    
    Following on from that, you have the [a1.metal](https://aws.amazon.com/ec2/instance-types/a1/) instance on AWS with 16 cores and 30GB / RAM [for roughly 0.4 USD / hour](https://instances.vantage.sh/aws/ec2/a1.metal?region=us-east-1&os=linux&cost_duration=hourly&reserved_term=Standard.noUpfront), and roughly half that cost with a 1x year reservation. The a1.metal is the first generation of Graviton and in our testing with customers came up quite a bit slower than Ampere or Graviton 3. On the plus side, these machines are cheap and if you're already on AWS, it may be easier to start with. GP3 volumes or provisioned concurrency may increase performance over the default of GP2 volumes. Reach out to us for more information.
 
2. Arm for on-premises

    For on-premises ARM64 builds, we recommend the Mac Mini M1 or M2 with Asahi Linux with 16GB of RAM or more, and 512GB of disk or more.
    
    Ampere and their partners also offer 1U and 2U servers, along with and [desktop-form workstations](https://www.ipi.wiki/products/ampere-altra-developer-platform) which can be racked or installed in your office.

    The Raspberry Pi 4 and 5 also work well when used with an external NVMe, and in one instance [was much faster than using emulation with a Hosted GitHub Runner](https://twitter.com/alexellisuk/status/1583092051398524928?s=20&t=2SelTpdc5idJLmayIu3Djw).

3. Arm VMs with nested virtualisation

    The current generations of Arm CPUs available from cloud providers do not support KVM, or nested virtualisation, which means you need to pick from the previous two options.

    There are Arm VMs available on Azure, GCP, and Oracle OCI. We have tested each and since they are based on the same generation of Ampere Altra hardware, we can confirm that they do not have KVM available and will not work for running actuated builds.

### Want to talk to us?

Still not sure which option is right for your team? Get in touch with us on the [Actuated Slack](https://self-actuated.slack.com) and we'll help you decide.

## Next steps

Now that you've created a Server or VM with the recommended Operating System, you'll need to install the actuated agent and get in touch with us, to register it.

* [Install the Actuated Agent](/install-agent)
