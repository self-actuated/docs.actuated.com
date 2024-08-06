# Example: GitHub Actions cache

Jobs on Actuated runners start in a clean VM each time. This means dependencies need to be downloaded and build artifacts or caches rebuilt each time. Caching these files in the actions cache can improve workflow execution time.

A lot of the setup actions for package managers have support for caching built-in. See: [setup-node](https://github.com/actions/setup-node#caching-global-packages-data), [setup-python](https://github.com/actions/setup-python#caching-packages-dependencies), etc. They require minimal configuration and will create and restore dependency caches for you.

If you have custom workflows that could benefit from caching the cache can be configured manually using the [actions/cache](https://github.com/actions/cache).

Using the actions cache is not limited to GitHub hosted runners but can be used with self-hosted runners. Workflows using the cache action can be converted to run on Actuated runners. You only need to change `runs-on: ubuntu-latest` to `runs-on: actuated`.

## Use the GitHub Actions cache

In this short example we will build [alexellis/registry-creds](https://github.com/alexellis/registry-creds). This is a Kubernetes operator that can be used to replicate Kubernetes ImagePullSecrets to all namespaces.

### Enable caching on a supported action

Create a new file at: `.github/workflows/build.yaml` and commit it to the repository.

```yaml
name: build

on: push

jobs:
  build:
    runs-on: actuated-4cpu-12gb
    steps:
      - uses: actions/checkout@v3
        with:
          repository: "alexellis/registry-creds"
      - name: Setup Golang
        uses: actions/setup-go@v3
        with:
          go-version: ~1.19
          cache: true
      - name: Build
        run: |
          CGO_ENABLED=0 GO111MODULE=on \
          go build -ldflags "-s -w -X main.Release=dev -X main.SHA=dev" -o controller
```

To configure caching with the [setup-go action](https://github.com/actions/setup-go) you only need to set the `cache` input parameter to true.

The cache is populated the first time this workflow runs. Running the workflow after this should be significantly faster because dependency files and build outputs are restored from the cache.

### Manually configure caching

If there is no setup action for your language that supports caching it can be configured manually.

Create a new file at: `.github/workflows/build.yaml` and commit it to the repository.

```yaml

name: build

on: push

jobs:
  build:
    runs-on: actuated-4cpu-12gb
    steps:
      - uses: actions/checkout@v3
        with:
          repository: "alexellis/registry-creds"
      - name: Setup Golang
        uses: actions/setup-go@v3
        with:
          go-version: ~1.19
          cache: true
      - name: Setup Golang caches
        uses: actions/cache@v3
        with:
        path: |
            ~/.cache/go-build
            ~/go/pkg/mod
        key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
        restore-keys: |
            ${{ runner.os }}-go-
      - name: Build
        run: |
          CGO_ENABLED=0 GO111MODULE=on \
          go build -ldflags "-s -w -X main.Release=dev -X main.SHA=dev" -o controller
```

The setup `Setup Golang caches` uses the [cache action](https://github.com/actions/cache) to configure caching.

The `path` parameter is used to set the paths on the runner to cache or restore. The `key` parameter sets the key used when saving the cache. A hash of the go.sum file is used as part of the cache key.

## Further reading

* Checkout the list of `actions/cache` [examples](https://github.com/actions/cache/blob/main/examples.md) to configure caching for different languages and frameworks.
* See our blog: [Make your builds run faster with Caching for GitHub Actions](https://actuated.com/blog/caching-in-github-actions)