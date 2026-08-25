# Prepare the macOS base image

When enrolling a new Apple Mac for use with actuated, you must first prepare a
base image from Apple's IPSW restore image. The restore image will be booted,
and walk through the setup wizard, and install actuated's guest agent. From
there, two additional stages set up the Command Line Tools CLT (such as git,
Swift, etc), then you can follow further steps to install Xcode on top to build
native applications for targets such as iOS.

Actuated does not distribute macOS due to restrictions in the EULA limiting
redistribution.

Complete the host requirements and
[install the agent binary](install-macos-agent.md#2-install-the-agent) first,
then run all commands as the regular service user, not root.

## Choose an image preparation method

The automatic method is recommended. It completes the macOS Setup Assistant
without user interaction. The manual method opens the VM console and leaves
the macOS Setup Assistant to the operator.

Both methods create the same base bundle at `~/.actuated/base.bundle`.

=== "Automatic"

    Create the VM from Apple's latest restore image supported by the agent:

    ```bash
    ~/.actuated/bin/agent base create \
      --ipsw latest \
      --bundle ~/.actuated/base.bundle
    ```

    The agent downloads the IPSW from Apple, which can take a few minutes.

    Start the VM and let the agent complete the macOS Setup Assistant. The
    default guest account is `runner` with password `runner`.

    ```bash
    ~/.actuated/bin/agent base oobe \
      --bundle ~/.actuated/base.bundle
    ```

    When setup finishes, leave the `oobe` command running. Open a second
    Terminal, install Apple's Command Line Tools and the GitHub Actions runner
    into the base image, then shut down the VM:

    ```bash
    ~/.actuated/bin/agent base provision \
      --bundle ~/.actuated/base.bundle

    ~/.actuated/bin/agent vm agent-shutdown \
      --bundle ~/.actuated/base.bundle
    ```

=== "Manual"

    Create the VM and open its console:

    ```bash
    ~/.actuated/bin/agent base create \
      --ipsw latest \
      --bundle ~/.actuated/base.bundle
    ```

    The agent downloads the IPSW from Apple, which can take a few minutes.

    Complete the macOS Setup Assistant as follows:

    1. Select the language and region.
    2. Set up the Mac as new and skip accessibility setup.
    3. Create the local account using `runner` for the full name, account name,
       and password.
    4. Skip Apple Account sign-in. The guest must not use an Apple Account.
    5. Complete the privacy and analytics screens and wait for the desktop.

    Open Terminal inside the VM and run:

    ```bash
    sudo "/Volumes/My Shared Files/guest-agent" init
    ```

    Enter the `runner` password when prompted. The VM shuts down and the
    `base create` command exits after the guest agent is installed.

    Start the VM again, install Apple's Command Line Tools and the GitHub
    Actions runner into the base image, then shut down the VM:

    ```bash
    ~/.actuated/bin/agent vm run \
      --bundle ~/.actuated/base.bundle &

    ~/.actuated/bin/agent base provision \
      --bundle ~/.actuated/base.bundle

    ~/.actuated/bin/agent vm agent-shutdown \
      --bundle ~/.actuated/base.bundle
    ```

If `latest` cannot be installed on the host, download a compatible IPSW from
Apple and pass its local path to `--ipsw` instead.

The prepared base image now contains Apple's Command Line Tools and the GitHub
Actions runner. Per-job credentials are supplied to disposable clones and are
never stored in the base.

## Add Xcode into the base image

The default base image contains Apple's Command Line Tools, but not the full
Xcode application or additional Apple platform support. If the runner needs
Xcode, create a separate Xcode base image after completing the provisioning
steps above.

Make sure [Xcode](https://developer.apple.com/xcode/resources/) is installed on
the host, then build the Xcode image from the provisioned base. By default, the
agent uses the newest installed version:

```bash
~/.actuated/bin/agent base xcode \
  --from ~/.actuated/base.bundle \
  --to ~/.actuated/base-xcode.bundle
```

The command clones the provisioned base, adds Xcode and the iOS SDK, then seals
`base-xcode.bundle` without modifying `base.bundle`. Creating the Xcode image
takes approximately 6-10 minutes.

The `--platform` flag can be used to select which additional Apple platform
support Xcode downloads into the base image. It accepts `iOS`, `watchOS`,
`tvOS`, or `visionOS`, and defaults to `iOS`. Use `--platform ''` to install
Xcode without downloading an additional platform.

If multiple Xcode versions are installed, select one by its version or provide
its path explicitly:

```bash
~/.actuated/bin/agent base xcode \
  --from ~/.actuated/base.bundle \
  --to ~/.actuated/base-xcode.bundle \
  --xcode-version 26.6

~/.actuated/bin/agent base xcode \
  --from ~/.actuated/base.bundle \
  --to ~/.actuated/base-xcode.bundle \
  --xcode-app /Applications/Xcode-26.6.app
```

Each macOS agent uses one base image for all of its job VMs. Individual jobs
cannot select a different image. To configure the agent to use the Xcode image,
specify it when
[installing the macOS agent service](install-macos-agent.md#5-install-the-service):

```bash
~/.actuated/bin/agent install \
  --base ~/.actuated/base-xcode.bundle
```

## Verify the base image

Verify the base image:

```bash
~/.actuated/bin/agent base verify \
  --bundle ~/.actuated/base.bundle
```

The command boots a disposable clone and checks the Actuated agent, Command Line
Tools, and Actions runner. When an Xcode bundle is selected, it also checks the
Xcode version and configured platform SDK. It then shuts down and removes the
clone.

Example output:

```text
$ ~/.actuated/bin/agent base verify \
  --bundle ~/.actuated/base.bundle

   Verifying base.bundle (base)
 ✓ Cloning the base image base.bundle.verify
 ✓ Booting the clone (8.3s)
 ✓ Checking the guest agent f90c2b35c2e79ed9efc03124b6b42b0ed7e017cc, macOS 26.2
 ✓ Checking the Command Line Tools git version 2.50.1 (Apple Git-155), Apple clang version 21.0.0 (20.1s)
 ✓ Checking the Actions runner 2.336.0
 ✓ Shutting down and removing the clone (6.9s)

✓ base.bundle is ready to serve jobs
```

Next, return to [Install and join the macOS agent](install-macos-agent.md#4-enroll-the-host).
