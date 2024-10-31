# Example: Debug a job with SSH

If your it's included within your actuated plan, then you can get a shell into any self-hosted runner - including GitHub's own hosted runners.

Certified for:

- [x] `x86_64`
- [x] `arm64`

!!! info "Use a private repository if you're not using actuated yet"
    GitHub recommends using a private repository with self-hosted runners because changes can be left over from a previous run, even when using Actions Runtime Controller. Actuated uses an ephemeral VM with an immutable image, so can be used on both public and private repos. Learn why in the [FAQ](/faq).

## Connect to your job via SSH

You'll need to add the `id_token: write` permission to your workflow to use this action. It allows the action to authenticate with the SSH gateway using an [GitHub Actions OIDC token](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect).

Create a `.github/workflows/workflow.yaml` file

```yaml
name: connect

on:
  push:
    branches:
      - master
      - main
  workflow_dispatch:

permissions:
  id-token: write
  contents: read
  actions: read

jobs:
  connect:
    name: connect
    runs-on: actuated-4cpu-12gb
    steps:
    - uses: self-actuated/connect-ssh@master
```

Next, trigger a build via the workflow_dispatch event or a git push to the master branch.

Open `https://$SSH_GATEWAY/list` in your browser and look for your session, you can log in using the SSH command outputted for you.

Alternatively, you can view your own SSH sessions from the [actuated dashboard](https://dashboard.actuated.com).

Whenever you have a build that you just can't figure out - or if you want to explore the runner and tune it up to your needs, then you can simply add `- uses: self-actuated/connect-ssh@master` where you want to pause the build.

To release the session run `unblock` or `sudo reboot` from the SSH session.

Watch a demo:

<iframe width="560" height="315" src="https://www.youtube.com/embed/l9VuQZ4a5pc" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## Advanced: debug without blocking the build step

If you don't want to block at this build step, but want to connect mid-way through a build, you can do with `block: false` option.

```yaml
    steps:
      - uses: self-actuated/connect-ssh@master
        with:
          block: false
```

If you are unable to connect with SSH in time, you can add a short sleep at the end of your build:

```yaml
jobs:
  connect:
    name: connect
    runs-on: [actuated-8cpu-8gb]
    steps:
      - uses: self-actuated/connect-ssh@master
        with:
          block: false
      - run: |
          echo "Sleep for 5 minutes, since connect-ssh isn't going to block"
          sleep 500
```

