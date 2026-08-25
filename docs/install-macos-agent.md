# Install and join the macOS agent

This guide installs the native agent on an Apple Silicon Mac, prepares its
base image, connects the host to Actuated, and runs a test job.

## Installation

### 1. Prepare the host

You need:

* An Apple Silicon Mac with at least 16 GB RAM.
* Enough free disk space for macOS, the base image, and job clones.
* A macOS user to run the agent. Do not run the agent as root.
* Apple's Command Line Tools and [arkade](https://github.com/alexellis/arkade).

Install the Command Line Tools and prevent the host from sleeping:

```bash
xcode-select --install
sudo pmset -a sleep 0 displaysleep 0 disksleep 0
```

Use shorter DHCP leases for the ephemeral VMs:

```bash
sudo defaults write \
  /Library/Preferences/SystemConfiguration/com.apple.InternetSharing.default.plist \
  bootpd -dict DHCPLeaseTimeSecs -int 600
```

The guest agent releases its DHCP lease before shutdown, but Apple's `bootpd`
may retain the lease record and continue allocating addresses sequentially.
Keep the shorter lease time so abandoned addresses return to the pool quickly.

Review the [Actuated EULA](https://github.com/self-actuated/actuated/blob/master/EULA.md)
and the macOS software licence shown when the base image is installed.

### 2. Install the agent

Download and unpack the latest macOS agent release:

```bash
arkade oci install \
  ghcr.io/openfaasltd/actuated-agent-macos \
  ~/.actuated/bin
```

Confirm that the installed binary reports its version and Git commit:

```bash
~/.actuated/bin/agent version
```

The license is available in the email you received when you purchased your
subscription.

Run the following, then paste in your license, hit Enter once, then Control+D
to save the file:

```bash
cat > ~/.actuated/LICENSE
```

### 3. Prepare the base image

Follow [Prepare the macOS base image](prepare-macos-base.md). It documents both
the recommended automatic setup and the manual macOS Setup Assistant workflow.

Return here after `~/.actuated/base.bundle` has been provisioned.

### 4. Enroll the host

There are two options for enrolling the host:

=== "Automatic enrollment"

    Use the controller details supplied during onboarding:

    ```bash
    export ACTUATED_CONTROLLER="https://controller.example.com"
    export ACTUATED_ENROLL_TOKEN="..."
    export ACTUATED_AGENT_URL="https://mac-agent.example.com"

    ~/.actuated/bin/agent autoenroll
    ```

    `ACTUATED_AGENT_URL` is the public HTTPS endpoint where the Actuated
    control plane will reach this agent. Use the externally reachable URL
    configured for this host. The endpoint may become reachable only after the
    service is installed in the next step.

    The command creates `~/.actuated/TOKEN`, `key_rsa`, and `key_rsa.pub` on
    first use. Re-running it keeps those credentials and updates the controller
    record.

=== "Manual enrollment"

    Generate the local credentials and a controller configuration entry:

    ```bash
    ~/.actuated/bin/agent keygen \
      --customer CUSTOMER \
      --orgs GITHUB_ORG \
      --endpoint https://mac-agent.example.com
    ```

    `--endpoint` is the public HTTPS endpoint where the Actuated control plane
    will reach this agent. Use the externally reachable URL configured for
    this host.

    Share the generated `hosts.yaml` configuration with the Actuated team. We
    will confirm when the host has been added to the control plane and is ready
    to accept jobs.

### 5. Install the service

Starting a macOS VM requires the agent user's login keychain to be unlocked.
To recover without intervention after a reboot, the host therefore uses
automatic login.

FileVault prevents automatic login. For unattended operation,
[turn off FileVault](https://support.apple.com/guide/mac-help/turn-off-filevault-on-mac-mchlp2560/mac),
confirm that it is off, then enable automatic login:

```bash
sudo fdesetup status  # Expect: FileVault is Off.
sudo ~/.actuated/bin/agent autologin --user "$(id -un)"
```

If FileVault remains enabled, an operator must unlock the Mac and log in as the
agent user after every reboot before jobs can start.

Install the service as the agent user, not with `sudo`.

=== "Direct connection"

    ```bash
    ~/.actuated/bin/agent install
    ```

=== "Inlets tunnel"

    An [inlets tunnel](https://docs.inlets.dev/cloud/) can provide a public
    endpoint for an agent on a private network. When using one, run the tunnel
    client and bind the agent to loopback:

    ```bash
    ~/.actuated/bin/agent install \
      --addr "127.0.0.1:8080"
    ```

The command uses the current agent binary, `~/.actuated` as its work directory,
and `~/.actuated/base.bundle` as the default base image. Use `--work-dir` when
the default work directory needs to change.

The `--base` flag selects the image used for every job VM and cannot be
overridden per job. If your jobs need Xcode, first
[prepare an image with the required version](prepare-macos-base.md#add-xcode-into-the-base-image),
then pass its path to `--base` when installing the service.

We recommend enabling automatic runner updates. Append
`--schedule-runner-updates` to the `install` command above to install a
second LaunchAgent that updates the GitHub Actions runner in the base image
every day at 04:00. This keeps the base image on a supported GitHub runner
version.

### 6. Verify the installation

Check the local service and view its logs through the agent CLI:

```bash
~/.actuated/bin/agent service status
~/.actuated/bin/agent service logs --lines 100

curl -s \
  -H "Authorization: Bearer $(cat ~/.actuated/TOKEN)" \
  http://127.0.0.1:8080/api/capacity
```

Confirm that the host appears as online in the
[Actuated Dashboard](https://dashboard.actuated.com).

Expected results:

* `service status` reports a running `com.openfaas.actuated-agent` service.
* Capacity returns JSON containing `"status":"running"`.
* The [Actuated Dashboard](https://dashboard.actuated.com) reports the host as
  online.

### 7. Run a test job

Create a GitHub Actions workflow:

```yaml
name: macOS agent test

on: workflow_dispatch

jobs:
  build:
    runs-on: actuated-macos-arm64-2cpu-8gb
    steps:
      - uses: actions/checkout@v7
      - run: sw_vers
      - run: sudo -n true
```

Trigger the workflow. The job should complete, and the single-use VM should be
removed afterward.

## Keep the agent and base image current

Upgrade the installed host agent from its published OCI image:

```bash
~/.actuated/bin/agent upgrade
~/.actuated/bin/agent service restart
~/.actuated/bin/agent version
```

The `upgrade` command replaces the installed agent when a newer version is
available.

The host binary embeds the guest agent, but upgrading the host does not modify
an existing base image. After upgrading, inject the embedded guest agent and
the latest GitHub Actions runner into the base with:

```bash
~/.actuated/bin/agent base update-runner
```

The command waits for running jobs to finish, then safely updates the base
image.

The command blocks until the update finishes. Follow its progress in another
Terminal with:

```bash
~/.actuated/bin/agent service logs --follow
```
