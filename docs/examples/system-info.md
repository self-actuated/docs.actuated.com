# Example: Get system information about your microVM

This sample reveals system information about your runner.

Certified for:

- [x] `x86_64`
- [x] `arm64`

!!! info "Use a private repository if you're not using actuated yet"
    GitHub recommends using a private repository with self-hosted runners because changes can be left over from a previous run, even when using Actions Runtime Controller. Actuated uses an ephemeral VM with an immutable image, so can be used on both public and private repos. Learn why in the [FAQ](/faq).

## Try out the action on your agent

Create a specs.sh file:

```bash
#!/bin/bash

echo Hostname: $(hostname)

echo whoami: $(whoami)

echo Information on main disk
df -h /

echo Memory info
free -h

echo Total CPUs:
echo CPUs: $(nproc)

echo CPU Model
cat /proc/cpuinfo |grep "model name"

echo Kernel and OS info
uname -a

if ! [ -e /dev/kvm ]; then
  echo "/dev/kvm does not exist"
else
  echo "/dev/kvm exists"
fi

echo OS info: $(cat /etc/os-release)

echo PATH: ${PATH}

echo Egress IP:
curl -s -L -S https://checkip.amazonaws.com
```

Create a new file at: `.github/workflows/build.yml` and commit it to the repository.

```yaml
name: CI

on:
  pull_request:
    branches:
      - '*'
  push:
    branches:
      - master

jobs:
  specs:
    name: specs
    runs-on: actuated-4cpu-12gb
    steps:
      - uses: actions/checkout@v4
      - name: Check specs
        run: |
          ./specs.sh
```

Note how the hostname changes every time the job is run.

## Perform a basic benchmark

Update the specs.sh file to include benchmarks for disk and network connection:

```bash
echo Installing hdparm

sudo apt update -qqqqy && sudo apt install -qqqqy hdparm

echo Read speed

sudo hdparm -t $(mount |grep "/ "|cut -d " " -f1)

echo Write speed

sync; dd if=/dev/zero of=./tempfile bs=1M count=1024; sync

echo Where is this runner?

curl -s http://ip-api.com/json|jq

echo Information on main disk

df -h /

echo Public IP:

curl -s -L -S https://checkip.amazonaws.com

echo Checking speed
sudo pip install speedtest-cli
speedtest-cli
```

For the fastest servers backed by NVMes, with VMs running on a dedicated drive, we tend to see:

* Read speeds of 1000+ MB/s.
* Write speeds of 1000+ MB/s.

The Internet speed test will give you a good idea of how quickly large artifacts can be uploaded or downloading during jobs.

The instructions for a [Docker registry cache on the server](/tasks/registry-mirror) can make using container images from public registries much quicker.

