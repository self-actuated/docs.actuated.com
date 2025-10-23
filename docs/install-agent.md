# Add your first agent to actuated

actuated is split into three parts:

1. An Actuated Agent (agent) that you run on your own machines or VMs (server), which can launch a VM with a single-use GitHub Actions runner.
2. A VM image launched by the agent, with all the preinstalled software found on a hosted GitHub Actions runner.
3. Our own control plane that talks to GitHub on your behalf, and schedules builds across your fleet of agents.

All we need you to do is to install our agent on one or more servers, then we take care of the rest. We'll even be able to tell you if your server goes offline for any reason.

!!! info "Have you registered your organisation yet?"
    Before you can add an agent, you or your GitHub organisation admin will need to install the: [Actuated GitHub App](https://docs.actuated.com/register/#install-the-github-app).

### Pick your Actuated Servers

Pick your Actuated Servers carefully using our guide: [Pick a host for actuated](/provision-server)

## Review the End User License Agreement (EULA)

Make sure you've read the [Actuated EULA](https://github.com/self-actuated/actuated/blob/master/EULA.md) before registering your organisation with the actuated GitHub App, or starting the agent binary on one of your hosts.

If you missed it in the "Provision a Server" page, we recommend you use Ubuntu 22.04 as the host operating system on your Server.

## Install the Actuated Agent

!!! info "Do you want a free, expert installation?"

    Our team can install the agent and configure the server for you. Just request our public SSH key, and add it to `.ssh/authorized_keys` and create a DNS A or CNAME record for your server, and send all the details over to us on the Actuated Slack.

    Alternatively, you can run through the semi-automatic installation with the details below.


1. Install your license for actuated

    The license is available in the email you received when you purchased your subscription. If someone else bought the subscription, they should be able to forward it to you.

    Run the following, then paste in your license, hit enter once, then Control + D to save the file.

    ```bash
    mkdir -p ~/.actuated
    cat > ~/.actuated/LICENSE
    ```

2. Download the Actuated Agent and installation script to the server

    > Setting up an ARM64 agent? Wherever you see `agent` in a command, change it to: `agent-arm64`. So instead of `agent keygen` you'd run `agent-arm64 keygen`.

    Install [arkade](https://github.com/alexellis/arkade) using the command below, or download it from the [releases page](https://github.com/alexellis/arkade/releases).

    Download the latest agent and install the binary to `/usr/local/bin/`:

    ```bash
    (
    # Install arkade
    curl -sLS https://get.arkade.dev | sudo sh

    # Use arkade to extract the agent from its OCI container image
    arkade oci install ghcr.io/openfaasltd/actuated-agent:latest --path ./agent
    chmod +x ./agent/agent*
    sudo mv ./agent/agent* /usr/local/bin/
    )
    ```

    The install.sh script installs required dependencies such as: containerd, CNI plugins, and the Firecracker binary.

    **For best performance**, a dedicated drive, volume or partition is required to store the filesystems for running VMs. If you do not have a volume or extra drive attached, then you can shrink the root partition, and use the resulting free space.

    Two storage backends are supported. By default the agent uses [Device mapper](https://en.wikipedia.org/wiki/Device_mapper). Alternatively the agent can use ZFS volumes within a ZFS pool.

    The disk space allocated to job runners can be configured by setting the `BASE_SIZE` environment variable. The default is `30GB` which is generally sufficient for most teams.

    === "Devmapper (recommended)"

        The `VM_DEV` variable should be set with a disk or partition that can be formatted for VM storage.

        ```bash
        (
        cd agent
        VM_DEV=/dev/nvme0n2 sudo -E ./install.sh
        )
        ```

    === "Devmapper loopback"

        If you do not have additional storage available at this time you can omit the `VM_DEV` variable and the installer will generate a loopback filesystem for you.

        ```bash
        (
        cd agent
        sudo -E ./install.sh
        )
        ```

        The default size of the loopback file is `200GB` which is stored at `/var/lib/containerd/devmapper`.

    === "ZFS"

        The script will create a ZFS pool and dataset for you. If you want to manually setup ZFS or use an existing ZFS pool or dataset see [ZFS Configuration](#installation-options).

        The `VM_DEV` variable should be set with a disk or partition that can be formatted for ZFS storage.

        ```bash
        (
        cd agent
        VM_DEV=/dev/nvme0n2 STORAGE=zfs sudo -E ./install.sh
        )
        ```

    === "ZFS loopback"

        If you do not have additional storage available at this time you can omit the `VM_DEV` variable and the installer will generate a loopback filesystem for you.

        ```bash
        (
        cd agent
        STORAGE=zfs sudo -E ./install.sh
        )
        ```

3. Generate your enrollment file

    You'll need to create a DNS A or CNAME record for each server you add to actuated, this could be something like `server1.example.com` for instance.

    Run the following to create an enrollment file at `$HOME/.actuated/agent.yaml`:

    > For an Arm server run `agent-arm64` instead of `agent`

    ```bash
    agent enroll --url https://server1.example.com
    ```

    The enrollment file contains:

    * The hostname of the server
    * The public key of the agent which we use to encrypt tokens sent to the agent to bootstrap runners to GitHub Actions
    * A unique API token encrypted with our public key, which is used by the control plane to authenticate each message sent to the agent

4. Configure and start the agent

    Use the `install-service` command to configure and install a systemd service to start the actuated agent.

    The actuated control plane will only communicate with agents exposed over HTTPS to ensure proper encryption is in place. An API token is used in addition with the TLS connection for all requests.

    Any bootstrap tokens sent to the agent are further encrypted with the agent's public key.

    For hosts with public IPs, you will need to use the built-in TLS provisioning with Let's Encrypt. For hosts behind a firewall, NAT or in a private datacenter, you can [use inlets](/expose-agent/) to create a secure tunnel to the agent.

    We're considering other models for after the pilot, for instance GitHub's own API has the runner make an outbound connection and uses long-polling.

    === "Expose the agent on the Internet with HTTPS"

        The easiest way to configure everything is to run as root. The --user flag can be used to run under a custom user account, however sudo access is still required for actuated.

        For an *x86_64* server, run:

        ```bash
        DOMAIN=agent1.example.com
        # Replace with "zfs" if you opted for ZFS storage in the installation script.
        STORAGE=devmapper

        sudo -E agent install-service \
          --letsencrypt-domain $DOMAIN \
          --letsencrypt-email webmaster@$DOMAIN \
          --storage $STORAGE
        ```

        For an *Arm* server, run:

        ```bash
        DOMAIN=agent1.example.com
        # Replace with "zfs" if you opted for ZFS storage in the installation script.
        STORAGE=devmapper

        sudo -E agent-arm64 install-service \
          --letsencrypt-domain $DOMAIN \
          --letsencrypt-email webmaster@$DOMAIN \
          --storage $STORAGE
        ```

        > Note the different binary name: `agent-arm64`

    === "Run the agent from a private network"

        For an Actuated Agent behind an [inlets tunnel](https://inlets.dev), do not include the `--letsencrypt-*` flags, and instead add `--listen-addr "127.0.0.1:"`. See [expose the agent with HTTPS](/expose-agent/) for instructions on how the setup inlets.

        The easiest way to configure everything is to run as root. The --user flag can be used to run under a custom user account, however sudo access is still required for actuated

        ```bash
        # Replace with "zfs" if you opted for ZFS storage in the installation script.
        STORAGE=devmapper

        sudo -E agent install-service \
            --listen-addr "127.0.0.1:" \
            --storage $STORAGE
        ```

    If you need to make changes you can run the command again, or edit `/etc/default/actuated`.

    Check the service's status with:

    ```bash
    sudo systemctl status actuated
    sudo journalctl -u actuated --since today -f
    ```

5. Check that the control-plane is accessible

    ```bash
    curl -i https://server1.example.com
    ```

    A correct response is a *403*.

6. Send us your agent's connection info

    Share the `$HOME/.actuated/agent.yaml` file with us so we can add your agent to the actuated control plane.

    We'll let you know once we've added your agent to actuated and then it's over to you to start running your builds.

    Once you've run our test build, you need to run the steps for systemd mentioned above.

## Next steps

You can now start your first build and see it run on your actuated agent.

[Start a build on your agent](/test-build)

```diff
name: ci

on: push

jobs:
  build-golang:
-    runs-on: ubuntu-latest
+    runs-on: actuated-4cpu-16gb
```

The amount of RAM and CPU can be picked independently.

For Arm servers change the prefix from `actuated-` to `actuated-arm64`:

```diff
name: ci

on: push

jobs:
  build-golang:
-    runs-on: ubuntu-latest
+    runs-on: actuated-arm64-8cpu-32gb
```

> You can also specify `actuated-any-4cpu-8gb` if you don't mind whether the job runs on one of your amd64 or arm64 servers.

### Other considerations

If you'd like to install a firewall, [ufw](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-20-04) should be relatively quick to configure.

You will need the following ports open:

* `443` - the encrypted control plane for actuated
* `80` - used with Let's Encrypt to obtain a TLS certificate during the HTTP01 challenge
* `22` - we recommend leaving port 22 open so that you can log into the machine with SSH, if needed. You could also change this to a high or random port

We do not recommend restricting outgoing traffic on the server as this will probably cause you issues with your builds.

See also: [Troubleshooting your agent](/troubleshooting)

### Installation options for the `install.sh` script

When no options are given, all the defaults are assumed, which uses devmapper with a loopback file for VM storage, which is suitable for basic exploration or light use.

Dedicated disks or partitions with devmapper or ZFS is recommended for production use.

General options:

| ENV | Description | Default |
| --- | ----------- | ------- |
| `VM_DEV` | Disk or partition to use for VM storage. If omitted a loopback file will be created. | `""` |
| `STORAGE` | Storage backend to use for VM storage. Options are `devmapper` or `zfs`. | `devmapper` |
| `BASE_SIZE` | Base size of each VM filesystem. | `30GB` |

ZFS-specific options:

| ENV | Description | Default |
| --- | ----------- | ------- |
| `ZPOOL` | Name of the ZFS pool to use or create for ZFS storage. | `actuated_zpool` |
| `ZFS_DATASET` | Name of the ZFS dataset to use or create for ZFS . | `${ZPOOL}/snapshots}` |

The install script will automatically try to create a ZFS pool and dataset if you have opted for ZFS storage in the installation script and they do not already exist.

If you have an existing pool or dataset that you want to use or if you want to create them manually you can configure the script to use them

```sh
STORAGE=zfs ZPOOL=mypool ZFS_DATASET=mypool/actuated sudo ./install.sh
```