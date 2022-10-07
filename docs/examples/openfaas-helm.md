# Example: Publish an OpenFaaS function

This example will create a Kubernetes cluster using KinD, deploy OpenFaaS using Helm, deploy a function, then invoke the function. There are some additional checks for readiness for Kubernetes and the OpenFaaS gateway.

You can adapt this example for any other Helm charts you may have for E2E testing.

We also recommend considering [arkade](https://arkade.dev) for installing CLIs and common Helm charts for testing.

[Docker CE](https://docker.io) is preinstalled in the actuated VM image, and will start upon boot-up.

Certified for:

- [x] `x86_64`
- [x] `arm64`

!!! warning "Don't use a public repository"
    Due to limitations in the design of GitHub's runner, we recommend using a private repository. Learn more in the [FAQ](/faq.md).

## Try out the action on your agent

Create a new GitHub repository in your organisation.

Add: `.github/workflows/e2e.yaml`

```yaml
name: e2e

on:
  push:
    branches:
      - '*'
  pull_request:
    branches:
      - '*'

permissions:
  actions: read
  contents: read

jobs:
  e2e:
    runs-on: actuated
    steps:
      - uses: actions/checkout@master
        with:
          fetch-depth: 1
      - name: Get latest developer tools
        run: |
            curl -sLSf https://get.arkade.dev | sudo sh
            arkade get \
                faas-cli \
                kubectl \
                kind
            chmod +x $HOME/.arkade/bin/*
            sudo -E mv $HOME/.arkade/bin/* /usr/local/bin/
      - name: Install Kubernetes KinD
        run: |
          mkdir -p $HOME/.kube/
          kind create cluster --wait 300s
      - name: Add Helm chart, update repos and apply namespaces
        run: |
            kubectl apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml
            helm repo add openfaas https://openfaas.github.io/faas-netes/
            helm repo update
      - name: Install the Community Edition (CE)
        run: |
            helm repo update \
            && helm upgrade openfaas --install openfaas/openfaas \
                --namespace openfaas  \
                --set functionNamespace=openfaas-fn \
                --set generateBasicAuth=true
      - name: Wait until OpenFaaS is ready
        run: |
            kubectl rollout status -n openfaas deploy/prometheus --timeout 5m
            kubectl rollout status -n openfaas deploy/gateway --timeout 5m
      - name: Port forward the gateway
        run: |
            kubectl port-forward -n openfaas svc/gateway 8080:8080 &

            attempts=0
            max=10

            until $(curl --output /dev/null --silent --fail http://127.0.0.1:8080/healthz ); do
                if [ ${attempts} -eq ${max} ]; then
                echo "Max attempts reached $max waiting for gateway's health endpoint"
                exit 1
                fi

                printf '.'
                attempts=$(($attempts+1))
                sleep 1
            done
      - name: Login to OpenFaaS gateway and deploy a function
        run: |
           PASSWORD=$(kubectl get secret -n openfaas basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode; echo)
           echo -n $PASSWORD | faas-cli login --username admin --password-stdin 

           faas-cli store deploy env

           faas-cli invoke env <<< ""

           curl -s -f -i http://127.0.0.1:8080/function/env

           faas-cli invoke --async env <<< ""

           kubectl logs -n openfaas deploy/queue-worker

           faas-cli describe env
```

If you'd like to deploy the function, check out a more comprehensive example of how to log in and deploy in [Serverless For Everyone Else](https://store.openfaas.com/l/serverless-for-everyone-else)
