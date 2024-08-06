# Roadmap

Actuated is in a pilot phase, running builds for participating customers. The core experience is functioning and we are dogfooding it for actuated itself, OpenFaaS Pro and inlets.

Our goal with the pilot is to prove that there's market fit for a solution like this, and if so, we'll invest more time in automation, user experience, agent autoscaling, dashboards and other polish.

The technology being used is run at huge scale in production by AWS (on Lambda and Fargate) and GitHub (self-hosted runners use the same runner software and control plane).

We believe actuated solves a real problem making CI fast, efficient, and multi-arch.

If you'd like to try it out for your team, [Register interest for the pilot](https://forms.gle/8XmpTTWXbZwWkfqT6).

Shipped

* [x] Firecracker MicroVM support for runners
* [x] Secure builds for both public and private repos
* [x] *Fat VM* image to match tooling installed by GitHub Actions
* [x] KinD support for runner's Kernel
* [x] K3s support for runner's Kernel
* [x] ARM64 support, including Raspberry Pi 4B
* [x] Efficient scheduling of jobs across fleet of agents
* [x] Samples for K3s/KinD/Matrix builds and OpenFaaS functions
* [x] Subscription plans delivered by Gumroad
* [x] API for reviewing connected agents and queue depth
* [x] Job event auditing for review via API
* [x] Documentation site with detailed GitHub Actions examples
* [x] Customer dashboard UI to show connected agents and build queue
* [x] Official website [actuated.dev](https://actuated.dev)
* [x] Remote / automated update of agents via control plane
* [x] Blog feature on actuated.dev with news, tutorials and updates from our team
* [x] Performance testing for Ionos & Scaleway for cost effective AMD bare-metal
* [x] Daily build statistics on your dashboard
* [x] Docker cache directly on the Actuated Hosts (servers) for much faster builds and avoiding rate-limiting
* [x] Subscriptions: migration to LemonSqueezy for lower fees, and more payment options
* [x] Dashboard - animation on all data pages for better feedback when refreshing data
* [x] Detailed insights across your organisation on usage
* [x] Detailed insights across your repos
* [x] Detailed insights by committer
* [x] Integrated SSH debug for runners within dashboard and CLI
* [x] At a glance insights for the day's activity so far
* [x] CLI/API for remote logs of VMs and the actuated agent
* [x] CLI/API for restarting the agent and rebooting a server
* [x] Examples for using S3/Minio running on the server as an actions cache, instead of the default hosted cache within Azure
* [x] Specify a custom runner size for an individual workflow - i.e. `actuated-8cpu-12gb`
* [x] Specify `actuated-any` to run jobs on any available server whether amd64 or arm64, for architecture-agnostic workflows such as npm or for browser testing. 
* [x] GPU pass-through for ML and AI workloads - [Accelerate GitHub Actions with dedicated GPUs](https://actuated.com/blog/gpus-for-github-actions) - [Run AI models with ollama in CI with GitHub Actions](https://actuated.com/blog/ollama-in-github-actions)
* [x] Linux Kernel 6.1 for 64-bit Arm
* [x] Burst above subscription concurrency for busy periods - [Introducing burst billing and capacity for GitHub Actions](https://actuated.com/blog/burst-billing-capacity)
* [x] Acquisition of actuated.com TLD

In progress:

* [ ] Actuated for self-hosted GitLab. (see below section)

Coming next:

* [ ] Linux Kernel 6.1 for *x86_64*
* [ ] Support for private, self-hosted GitHub Enterprise Server (GHES) installations

Open for customer contributions:

* [ ] Examples for setting up an apt/yum mirror for faster builds
* [ ] Example for configuring two different Docker pull through registries instead of just one.

Under consideration:

* [ ] Custom CA for self-hosted S3, Minio, Docker Registries, apt/yum mirrors, etc.
* [ ] Summary of CPU/RAM/disk consumption of builds
* [ ] Right-sizing of build VMs based upon prior build history
* [ ] Automated agent installation and bootstrap
* [ ] Actuated for Jenkins

Items marked under consideration are awaiting customer interest. Reach out to us if you'd like to see these features sooner.

Is there something else you need? If you're already a customer, contact us via [the actuated Slack](https://self-actuated.slack.com) or [Register for interest](https://forms.gle/8XmpTTWXbZwWkfqT6).

## Actuated for GitLab

[Learn about the tech preview](https://actuated.com/blog/secure-microvm-ci-gitlab)

Ready for use by customers:

* [x] Actuated integration with self-hosted GitLab CI either on-premises or on the cloud
* [x] Ephemeral one-time runners with their own dedicated Docker Daemon
* [x] Immutable VM image for each build, built with automation
* [x] Schedule jobs across multiple bare-metal hosts or VMs with KVM available
* [x] Custom VM size scheduling
* [x] Manual enrollment of of projects as required

Coming soon:

* [ ] Automatic enrollment of of projects as required when the `actuated` tag has been added
* [ ] actuated-cli integration
* [ ] actuated dashboard - daily glance, runners and build queue

Coming later:

* [ ] actuated dashboard - SSH debug, insights / reports

[Register your interest](https://forms.gle/8XmpTTWXbZwWkfqT6) if you'd like to talk to our team.

