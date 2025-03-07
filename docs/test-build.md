# Run a test build

Now that you've [registered your GitHub organisation](/register), [created a server](/provision-server), and [configured the agent](/install-agent), you're ready for a test build.

We recommend you run the following build without changes to confirm that everything is working as expected. After that, you can modify an existing build and start using actuated for your team.

The below steps should take less than 10 minutes.

## Create a repository and workflow

This build will show you the specs, OS and Kernel name reported by the MicroVM.

Note that if you're running on an Arm server, You'll adapt the prefix of `runs-on: actuated-`, to `runs-on: actuated-arm64-` instead.

1. Create a test repository and a GitHub Action

    Create `./.github/workflows/ci.yaml`:

    ```yaml
    name: CI

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
        specs:
            name: specs
            # runs-on: actuated-arm64-2cpu-8gb
            runs-on: actuated-arm64-2cpu-8gb
            steps:
            - uses: actions/checkout@v4
            - name: Check specs
              run: |
                ./specs.sh
    ```

    Note that the `runs-on:` field says `actuated-` and not `ubuntu-latest`. This is how the actuated control plane knows to send this job to your agent. There are no fixed sets of vCPU and RAM, you can make up your own combinations.

    Then add `specs.sh` to the root of the repository, and remember, that you must run `chmod +x specs.sh` afterwards to make it executable.

    ```bash
    #!/bin/bash

    echo Information on main disk
    df -h /

    echo Memory info
    free -h

    echo Total CPUs:
    echo CPUs: $(nproc)

    echo CPU Model
    cat /proc/cpuinfo |grep -i "Model"|head -n 2

    echo Kernel and OS info
    uname -a

    echo Generally, KVM should not be available unless specifically enabled
    if ! [ -e /dev/kvm ]; then
        echo "/dev/kvm does not exist"
    else
        echo "/dev/kvm exists"
    fi

    echo OS
    cat /etc/os-release

    echo Egress IP:
    curl -s -L -S https://checkip.amazonaws.com

    echo Speed test of Internet
    sudo pip install speedtest-cli
    speedtest-cli

    echo Checking Docker
    docker run alpine:latest cat /etc/os-release
    ```

    Don't leave out this step!

    ```bash
    chmod +x ./specs.sh
    ```

2. Hit commit, and watch the VM boot up.

    You'll be able to see the runners registered for your organisation on the [Actuated Dashboard](https://dashboard.actuated.com) along with the build queue and stats for the current day's builds.

3. If you're curious

    You can view the logs of the agent by logging into one of the Actuated Servers with SSH and running the following commands:

    ```bash
    sudo journalctl -u actuated -f -o cat

    # Just today's logs:
    sudo journalctl -u actuated --since today -o cat
    ```

    And each VM writes the logs from its console and the GitHub Actions Runner to `/var/log/actuated/`.

    ```bash
    sudo cat /var/log/actuated/*
    ```

Do you have any questions or comments? Feel free to reach out to us over Slack in the `#onboarding` channel.

## Enable actuated for an existing repository

To add actuated to an existing repository, simply edit the workflow YAML file and change `runs-on:` to `runs-on: actuated-4cpu-16gb` and for Arm builds, change it to: `runs-on: actuated-arm64-4cpu-16gb`. The values for CPU and RAM can be customised to your needs, and there are no hard-coded, or fixed combinations, to allow for flexibility.

Learn more in [Custom VM Sizes](/examples/custom-vm-size/)

## Recommended: Enable a Docker Hub mirror

Do you use the Docker Hub in your builds? Any Dockerfile with a `FROM` that doesn't include a server name will be pulled from `docker.io`, and there are strict rate-limits for unauthenticated users.

1. Option 1 - authenticate

    Run `docker login` or use the [Docker Login Action](https://github.com/docker/login-action) just before you run Docker build or pull down any images with tooling like KinD

2. Option 2 - use a cache/mirror

    Use our guide to [Set up a registry cache and mirror](/tasks/registry-mirror) - this uses less bandwidth and increases the speed of builds where images are already present in the cache.
