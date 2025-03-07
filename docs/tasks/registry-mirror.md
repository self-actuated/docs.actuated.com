# Task: Set up a registry mirror

Use-cases:

* Increase speed of pulls and builds by caching images on Actuated Servers
* Reduce failed builds due to rate-limiting

If you use Docker in your self-hosted builds, there is a chance that you'll run into the rather conservative rate-limits.

See also: [Docker.com: Download rate limit](https://docs.docker.com/docker-hub/download-rate-limit/)

The Docker Hub allows for 100 image pulls within a 6 hour period, but this can be extended to 200 by logging in, or to 5000 by paying for a Pro license.

A registry mirror / pull-through cache running on an actuated agent is significantly faster than pulling from a remote server.

We will create a mirror that:

* Has no authentication, to keep the changes to your build to a minimum
* Is read-only - for pulling images only
* Only has access to pull images from the Docker Hub
* Is not exposed to the Internet, but only to Actuated VMs
* When unavailable for any reason, the build continues without error
* Works on both Intel/AMD and ARM64 hosts

This tutorial shows you how to set up what was previously known as "Docker's Open Source Registry" and is now a CNCF project called [distribution](https://github.com/distribution/distribution).

If you'd like to mirror another registry like gcr.io, ecr.io, quay.io, or your own registry, then you can use the same approach, but run each registry on a different port. The configuration may need to be set up manually, since the current action we have built is only designed for one mirror.

Certified for:

- [x] `x86_64`
- [x] `arm64`

## Create a Docker Hub Access token

Create a Docker Hub Access token with "Public repos only" scope, and save it as `~/hub.txt` on the Actuated Server.

![Settings for a public token](/images/read-only-public-token.png)

> Settings for an authorization token, with read-only permissions to public repositories

## Set up the registry on an actuated agent

```bash
(
  curl -sLS https://get.arkade.dev | sudo sh

  sudo arkade system install registry

  sudo mkdir -p /etc/registry
  sudo mkdir -p /var/lib/registry
)
```

Create a config file to make the registry only available on the Linux bridge for Actuated VMs.

Before doing so, you'll need to:

1. Create a file named `hub.txt` in your home directory.
2. Set the `USERNAME` variable to your Docker Hub username.
3. If you're using cloud-hypervisor, set the `BRIDGE` variable to `192.168.129.1`

```bash
export USERNAME=""
export TOKEN=$(cat ~/hub.txt)
export BRIDGE="192.168.128.1"

cat > /tmp/registry.yml <<EOF
version: 0.1
log:
  accesslog:
    disabled: true
  level: warn
  formatter: text

storage:
  filesystem:
    rootdirectory: /var/lib/registry

proxy:
  remoteurl: https://registry-1.docker.io
  username: $USERNAME

  # A Docker Hub Personal Access token created with "Public repos only" scope
  password: $TOKEN

http:
  addr: $BRIDGE:5000
  relativeurls: false
  draintimeout: 60s

  # Enable self-signed TLS from the TLS certificate and key
  # managed by actuated for server <> microVM communication
  tls:
    certificate: /var/lib/actuated/certs/server.crt
    key: /var/lib/actuated/certs/server.key
EOF
```

As the certificate is expired, actuated will automatically restart the `registry` service to use the new certificate.

Install and start the registry with a systemd unit file:

```bash
(
cat >> /tmp/registry.service <<EOF
[Unit]
Description=Registry
After=network.target actuated.service

[Service]
Type=simple
Restart=always
RestartSec=5s
ExecStart=/usr/local/bin/registry serve /etc/registry/config.yml

[Install]
WantedBy=multi-user.target
EOF

sudo mv /tmp/registry.service /etc/systemd/system/registry.service
sudo systemctl daemon-reload
sudo systemctl enable registry --now
)
```

Check the status with:

```bash
sudo journalctl -u registry -f
```

## Use the registry within a workflow

Create a new registry in your organisation, along with a: `.github/workflows/build.yml` file and commit it to the repository.

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

jobs:
    build:
        runs-on: [actuated-4cpu-8gb]
        steps:

        - name: Setup mirror
          uses: self-actuated/hub-mirror@master

        - name: Checkout
            uses: actions/checkout@v4
    
        - name: Pull image using cache
            run: |
            docker pull alpine:latest
```

!!! note
    The `self-actuated/hub-mirror` action already runs the `docker/setup-buildx` action, so if you have that in your builds already, you can remove it, otherwise it will overwrite the settings for the mirror. Alternatively, move the `self-actuated/hub-mirror` action to after the `docker/setup-buildx` action.

## Checking if it worked

You'll see the build run, and cached artifacts appearing in: `/var/lib/registry/`.

```bash
find /var/lib/registry/ -name "alpine"

/var/lib/registry/docker/registry/v2/repositories/library/alpine
```

Add actuated's bridge <> VM CA bundle to the trust store on the server, to test the registry via curl:

```bash
sudo cp /var/lib/actuated/certs/ca.crt /usr/local/share/ca-certificates/actuated-ca.crt
sudo update-ca-certificates
```

You can also use the registry's API to query which images are available:

```bash
curl -i https://192.168.128.1:5000/v2/_catalog

HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Docker-Distribution-Api-Version: registry/2.0
Date: Wed, 16 Nov 2022 09:41:18 GMT
Content-Length: 52

{"repositories":["library/alpine","moby/buildkit"]}
```

You can check the status of the mirror at any time with:

```bash
sudo journalctl -u registry --since today
```

If you're not sure if the registry is working, or want to troubleshoot it, you can enable verbose logging, by editing the `log` section of the service file.

```
log:
  accesslog:
    disabled: false
  level: debug
  formatter: text
```

Then restart the service, and check the logs again. We do not recommend keeping this setting live as it will fill up the logs and disk quickly.

## A note on KinD

The self-actuated/hub-mirror action will configure both the Docker Daemon, and buildkit, however KinD uses its own instance of containerd and so must be configured separately.

See notes on [KinD with actuated](/examples/kind) for more information.

## Further reading

* [Docker: Configuration for the registry](https://docs.docker.com/registry/configuration/)
* [GitHub: View the project on GitHub](https://github.com/distribution/distribution)
