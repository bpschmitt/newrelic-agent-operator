<a href="https://opensource.newrelic.com/oss-category/#new-relic-experimental"><picture><source media="(prefers-color-scheme: dark)" srcset="https://github.com/newrelic/opensource-website/raw/main/src/images/categories/dark/Experimental.png"><source media="(prefers-color-scheme: light)" srcset="https://github.com/newrelic/opensource-website/raw/main/src/images/categories/Experimental.png"><img alt="New Relic Open Source experimental project banner." src="https://github.com/newrelic/opensource-website/raw/main/src/images/categories/Experimental.png"></picture></a>

# newrelic-agent-operator
<strong>newrelic-agent-operator</strong> is an implementation of the [opentelemetry-operator](https://github.com/open-telemetry/opentelemetry-operator), that manages auto-instrumentation of the workloads using New Relic APM agents.

## Description
If you are looking for an automated way to enable New Relic APM agents for applications running in your Kubernetes cluster, then you are in the right place. This will guide you in configuring APM agents in seamless manner. The idea behind the <strong>newrelic-agent-operator</strong> is to provide simple and efficient installation to the users. It uses a Kubernetes operator to create and configure a custom resource called <strong>Instrumentation</strong>. This custom resource provides higher-level abstraction for language specific agent configuration(s) in the Kubernetes cluster. Also, the underlying <strong>newrelic-agent-operator</strong> leverages Init containers to execute the required one-time agent setup, enabling seamless auto-instrumentation functionality. 

## Pre-requisites

- [Cert Manager](https://cert-manager.io/docs/installation/) will need to be installed in your cluster

## Installation
You’ll need a Kubernetes cluster to run against. You can use [KIND](https://sigs.k8s.io/kind) or [Minikube](https://minikube.sigs.k8s.io/docs/start/) to get a local cluster for testing, or run against a remote cluster.
**Note:** Your controller will automatically use the current context in your kubeconfig file (i.e. whatever cluster `kubectl cluster-info` shows).

1. Install newrelic-agent-operator using the helm chart
```sh
helm upgrade --install newrelic-agent-operator chart/ --set licenseKey='ABCD1234XYZ' -n newrelic --create-namespace
```

2. Create a secret containing your New Relic license key within your namespace.

```
kubectl create secret generic newrelic-key-secret -n my-namespace --from-literal=new_relic_license_key=ABCD1234XYZ
```

3. Configure and Install the Instrumentation custom resource.  This is a namespaced resource so each namespace will need an `Instrumentation` resource defined within it.

```sh
kubectl apply -f config/samples/instrumentation_v1alpha1_instrumentation.yaml -n my-namespace
```

Example `Instrumentation` custom resource: 

```
---
apiVersion: newrelic.com/v1alpha1
kind: Instrumentation
metadata:
  labels:
    app.kubernetes.io/name: instrumentation
    app.kubernetes.io/created-by: newrelic-agent-operator
  name: newrelic-instrumentation
spec:
# A required but separate opentelemetry collector is required for go autoinstrumentation set to 4318.
# exporter:
#  endpoint: http://opentelemetry-collector.newrelic:4318
# propagators:
#   - tracecontext
# sampler:
#   type: always_on
# resource:
#   resourceAttributes:
#     cluster.name: "newrelic-operator-demo"
  java:
    image: ghcr.io/newrelic-experimental/newrelic-agent-operator/instrumentation-java:latest
    env:
    # Example New Relic agent supported environment variables
      - name: NEW_RELIC_LABELS
        value: "environment:auto-injection"
    # Example overriding the appName configuration
    # - name: NEW_RELIC_POD_NAME
    #   valueFrom:
    #     fieldRef:
    #       fieldPath: metadata.name
    # - name: NEW_RELIC_APP_NAME
    #   value: "$(NEW_RELIC_LABELS)-$(NEW_RELIC_POD_NAME)"
  nodejs:
    image: ghcr.io/newrelic-experimental/newrelic-agent-operator/instrumentation-nodejs:latest
  python:
    image: ghcr.io/newrelic-experimental/newrelic-agent-operator/instrumentation-python:latest
  dotnet:
    image: ghcr.io/newrelic-experimental/newrelic-agent-operator/instrumentation-dotnet:latest
  php:
    image: ghcr.io/newrelic-experimental/newrelic-agent-operator/instrumentation-php:latest
  go:
    image: ghcr.io/open-telemetry/opentelemetry-go-instrumentation/autoinstrumentation-go:latest
```

## Getting Started

In order to enable auto-instrumentation for your application, you'll need to add the appropriate annotation to your deployment manifest.

```
instrumentation.newrelic.com/inject-java: "true"
instrumentation.newrelic.com/inject-nodejs: "true"
instrumentation.newrelic.com/inject-python: "true"
instrumentation.newrelic.com/inject-dotnet: "true"
instrumentation.newrelic.com/inject-php: "true"
instrumentation.newrelic.com/inject-go: "true"
```

Below is an example Java deployment:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-petclinic
spec:
  selector:
    matchLabels:
      app: spring-petclinic
  replicas: 1
  template:
    metadata:
      labels:
        app: spring-petclinic
      annotations:
        instrumentation.newrelic.com/inject-java: "true"
    spec:
      containers:
        - name: spring-petclinic
          image: ghcr.io/pavolloffay/spring-petclinic:latest
          ports:
            - containerPort: 8080
          env:
          - name: NEW_RELIC_APP_NAME
            value: spring-petclinic-demo
```

After deploying your application, ensure the operator successfully injected the language-specific instrumentation initContainer.  

```
$ kubectl get pod spring-petclinic-7fcccbcbd6-qm8kr -n my-namespace -o jsonpath='{.spec.initContainers[*].name}'
newrelic-instrumentation
```

Finally, login to your [New Relic](https://one.newrelic.com/) account and locate your APM application name under `Services - APM`.


## Building

1. Build and push your image to the location specified by `IMG`:

```sh
make container container-push  IMG=<some-registry>/newrelic-agent-operator:tag
```

2. Deploy the controller to the cluster with the image specified by `IMG`:

```sh
make deploy IMG=<some-registry>/newrelic-agent-operator:tag
```

### Uninstall CRDs
To delete the CRDs from the cluster:

```sh
make uninstall
```

### Undeploy controller
Undeploy the controller from the cluster:

```sh
make undeploy
```

### Test It Out
1. Install the CRDs into the cluster:

```sh
make install
```

2. Run your controller (this will run in the foreground, so switch to a new terminal if you want to leave it running):

```sh
make run
```

**NOTE:** You can also run this in one step by running: `make install run`

### Modifying the API definitions
If you are editing the API definitions, generate the manifests such as CRs or CRDs using:

```sh
make manifests
```

**NOTE:** Run `make --help` for more information on all potential `make` targets

More information can be found via the [Kubebuilder Documentation](https://book.kubebuilder.io/introduction.html)

## Things to consider
* Currently, <strong>newrelic-agent-operator</strong> supports instrumentation for [Python](https://docs.newrelic.com/docs/apm/agents/python-agent/configuration/python-agent-configuration/), [Java](https://docs.newrelic.com/docs/apm/agents/java-agent/getting-started/introduction-new-relic-java/), [Node.js](https://docs.newrelic.com/docs/apm/agents/nodejs-agent/getting-started/introduction-new-relic-nodejs/) and [.NET](https://docs.newrelic.com/docs/apm/agents/net-agent/getting-started/introduction-new-relic-net/).
* [cert-manager](https://cert-manager.io/docs/installation/) is required to be presented on cluster before starting newrelic-agent-operator


## Support

New Relic hosts and moderates an online forum where customers can interact with New Relic employees as well as other customers to get help and share best practices. If you're running into a problem, please raise an issue on this repository and we will try to help you ASAP. Please bear in mind this is an open source project and hence it isn't directly supported by New Relic.

## Contributing
We encourage your contributions to improve <strong>newrelic-agent-operator</strong> . Keep in mind when you submit your pull request, you'll need to sign the CLA via the click-through using CLA-Assistant. You only have to sign the CLA one time per project.
If you have any questions, or to execute our corporate CLA, required if your contribution is on behalf of a company,  please drop us an email at opensource@newrelic.com.

**A note about vulnerabilities**

As noted in our [security policy](../../security/policy), New Relic is committed to the privacy and security of our customers and their data. We believe that providing coordinated disclosure by security researchers and engaging with the security community are important means to achieve our security goals.

If you believe you have found a security vulnerability in this project or any of New Relic's products or websites, we welcome and greatly appreciate you reporting it to New Relic through [HackerOne](https://hackerone.com/newrelic).

## License
Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
