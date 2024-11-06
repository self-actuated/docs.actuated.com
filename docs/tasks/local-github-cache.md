# Task: Run a local GitHub Cache

The cache for GitHub Actions can speed up CI/CD pipelines. Hosted runners are placed close to the cache which means the latency is very low. Self-hosted runners can also make good use of this cache. Just like caching container images on the host in [a registry mirror](/tasks/registry-mirror/), you can also get a speed boost over the hosted cache by running your own cache directly on the host.

To improve cache speeds with Actuated runners you can run a self-hosted S3 server and switch out the official [actions/cache@v3](https://github.com/actions/cache) with [tespkg/actions-cache@v1](https://github.com/tespkg/actions-cache). The tespkg/actions-cache@v1 can target S3 instead of the proprietary GitHub cache.

You can run the cache on every actuated server for the speed of communicating over a loopback network, or you can run it on a single dedicated server that's placed in the same region as the actuated servers, which will still be very quick.

> Note that if you have multiple actuated hosts consider running a single dedicated server for the cache. Subsequent jobs can be scheduled to different hosts so there is no guarantee the cache is populated when running a cache on every actuated server.

## Set up an S3 cache

There are a couple of options to run a self-hosted S3 server, most notably [Seaweedfs](https://github.com/seaweedfs/seaweedfs) and [Minio](https://min.io/).

This guide will cover the setup of Seaweedfs but any S3 compatible service will work in a very similar way.

### Install Seaweedfs

Seaweedfs is distributed as a static Go binary, so it can be installed with [arkade](https://github.com/alexellis/arkade), or from the [GitHub releases page](https://github.com/seaweedfs/seaweedfs/releases).

```sh
arkade get seaweedfs
sudo mv ~/.arkade/bin/seaweedfs /usr/local/bin
```

Define a secret key and access key to be used from the CI jobs in the `/etc/seaweedfs/s3.conf` file.

Generate a secret key: `openssl rand -hex 16 > secret_key`

```bash
export ACCESS_KEY="" # Replace with your access key
export SECRET_KEY="$(cat ~/secret_key)"

cat >> /tmp/s3.conf <<EOF
{
  "identities": [
    {
      "name": "actuated",
      "credentials": [
        {
          "accessKey": "$ACCESS_KEY",
          "secretKey": "$SECRET_KEY"
        }
      ],
      "actions": [
        "Admin",
        "Read",
        "List",
        "Tagging",
        "Write"
      ]
    }
  ]
}
EOF

mkdir -p /etc/seaweedfs
sudo mv /tmp/s3.conf /etc/seaweedfs/s3.conf
```

Install and start Seaweedfs with a systemd unit file:

```bash
(
cat >> /tmp/seaweedfs.service <<EOF
[Unit]
Description=SeaweedFS
After=network.target

[Service]
User=root
ExecStart=/usr/local/bin/seaweedfs server -ip=192.168.128.1 -volume.max=0 -volume.fileSizeLimitMB=2048 -dir=/home/runner-cache -s3 -s3.config=/etc/seaweedfs/s3.conf
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

mkdir -p /home/runner-cache
sudo mv /tmp/seaweedfs.service /etc/systemd/system/seaweedfs.service
sudo systemctl daemon-reload
sudo systemctl enable seaweedfs --now
)
```

We have set `-volume.max=0 -volume.fileSizeLimitMB=2048` to minimize the amount of space used and to allow large zip files of up to 2GB, but you can change this to suit your needs.  See `seaweedfs server --help` for more options.

The `ip` only needs to be set to `192.168.128.1` if you are running the cache directly on the agent host. If you set up the cache to be accessible by multiple Actuated runner hosts use the appropriate interface IP address.

Check the status with:

```bash
sudo journalctl -u seaweedfs -f
```

## Use the self-hosted cache

To start using the local cache you will need to replace `actions/cache@v3` with `tespkg/actions-cache@v1` and add `tespkg/actions-cache` specific properties in addition to the `actions/cache` properties in your cache steps.

Some actions like [setup-node](https://github.com/actions/setup-node#caching-global-packages-data), [setup-python](https://github.com/actions/setup-python#caching-packages-dependencies), etc come with build-in support for the GitHub actions cache. They are not directly compatible with the self-hosted S3 cache and you will need to configure caching manually.

This is an example to manually configure caching for go:

```yaml
name: build

on: push

jobs:
  build:
    runs-on: actuated-4cpu-8gb
    steps:
    - name: Setup Golang
      uses: actions/setup-go@v3
      with:
        go-version: ~1.21
        cache: false
    - name: Setup Golang caches
      uses: tespkg/actions-cache@v1
      with:
        endpoint: "192.168.128.1"
        port: 8333
        insecure: true
        accessKey: ${{ secrets.ACTIONS_CACHE_ACCESS_KEY }}
        secretKey: ${{ secrets.ACTIONS_CACHE_SECRET_KEY }}
        bucket: actuated-runners
        region: local
        use-fallback: true

        # actions/cache compatible properties: https://github.com/actions/cache
        path: |
            ~/.cache/go-build
            ~/go/pkg/mod
        key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
        restore-keys: |
            ${{ runner.os }}-go-
```

`tespkg/actions-cache` specific properties:

* `use-fallback` - option means that if Seaweedfs is not installed on the host, or is inaccessible, the action will fall back to using the GitHub cache.
* `bucket` - the name of the bucket to use in Seaweedfs
* `region` - the bucket region - use `local` when running your own S3 cache locally. 
* `accessKey` and `secretKey` -  the credentials to use to access the bucket - we'd recommend using an organisation-level secret for this.
* `insecure` - use http instead of https. You may want to create a self-signed certificate for the S3 service and set `insecure: false` to ensure that the connection is encrypted. If you're running builds within private repositories, tampering is unlikely.

Checkout the list of `actions/cache` [examples](https://github.com/actions/cache/blob/main/examples.md) to configure caching for different languages and frameworks. Remember to replace `actions/cache@v3` with `tespkg/actions-cache@v1` and add the additional properties mentioned above.

### Caching the git checkout

Caching the git checkout can save a lot of time especially for large repos.

```yaml
jobs:
  build:
    runs-on: actuated-4cpu-8gb
    steps:
    - name: "Set current date as env variable"
      shell: bash
      run: |
        echo "CHECKOUT_DATE=$(date +'%V-%Y')" >> $GITHUB_ENV
      id: date
    - uses: tespkg/actions-cache@v1
      with:
        endpoint: "192.168.128.1"
        port: 8333
        insecure: true
        accessKey: ${{ secrets.ACTIONS_CACHE_ACCESS_KEY }}
        secretKey: ${{ secrets.ACTIONS_CACHE_SECRET_KEY }}
        bucket: actuated-runners
        region: local
        use-fallback: true
        path: ./.git
        key: ${{ runner.os }}-checkout-${{ env.CHECKOUT_DATE }}
        restore-keys: |
          ${{ runner.os }}-checkout-
```

The cache key uses a week-year format, rather than a SHA. Why? Because a SHA would change on every build, meaning that a save and load would be performed on every build, using up more space and slowing things down. In this example, there's only 52 cache entries per year.

### Caching node_modules with pnpm

For Node.js projects, the node_modules folder and yarn cache can become huge and take a long time to download. Switching to a local S3 cache can help bring that time down.

This example uses [pnpm](https://pnpm.io/), a fast, disk space efficient replacement for npm and yarn.

```yaml
jobs:
  build:
    runs-on: actuated-4cpu-8gb
    steps:
    - name: Install PNPM
      uses: pnpm/action-setup@v2
      with:
        run_install: |
          - args: [--global, node-gyp]

    - name: Get pnpm store directory
      id: pnpm-cache
      shell: bash
      run: |
        echo "STORE_PATH=$(pnpm store path)" >> $GITHUB_OUTPUT

    - uses: tespkg/actions-cache@v1
      with:
        endpoint: "192.168.128.1"
        port: 8333
        insecure: true
        accessKey: ${{ secrets.ACTIONS_CACHE_ACCESS_KEY }}
        secretKey: ${{ secrets.ACTIONS_CACHE_SECRET_KEY }}
        bucket: actuated-runners
        region: local
        use-fallback: true
        path:
          ${{ steps.pnpm-cache.outputs.STORE_PATH }}
          ~/.cache
          .cache
        key: ${{ runner.os }}-pnpm-store-${{ hashFiles('**/pnpm-lock.yaml') }}
        restore-keys: |
          ${{ runner.os }}-pnpm-store-

    - name: Install dependencies
      shell: bash
      run: |
        pnpm install --frozen-lockfile --prefer-offline
```

## Further reading

* From our blog: [Fixing the cache latency for self-hosted GitHub Actions](https://actuated.com/blog/faster-self-hosted-cache)
* A primer on using the GitHub Actions cache: [Using caching in builds](/examples/github-actions-cache/)