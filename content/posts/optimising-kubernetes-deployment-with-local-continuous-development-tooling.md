---
title: "Optimising Kubernetes deployment with local continuous development tooling"
author: "Steve Moss"
date: 2025-06-22T15:33:27.537Z
lastmod: 2025-11-14T14:28:50Z

description: "A blog post about using Skaffold for local Kubernetes development"

subtitle: "Using Skaffold to make local Kubernetes development simpler and more scalable"

image: "/skaffold-terminal.png"

images:
  - "/skaffold-terminal.png"
  - "/skaffold-logo.png"

aliases:
  - "/optimising-kubernetes-deployment-with-local-continuous-development-tooling-15b1fbf7a722"
---

![Skaffold Terminal](/skaffold-terminal.png "Skaffold running in a control loop watching for changes")

Something I’ve noticed through my work with Kubernetes is that it can be challenging to test things properly locally before deploying.

There are plenty of tools for testing things in continuous integration pipelines and as part of continuous deployment, but there aren’t many tools for increasing your confidence that your Kubernetes-based application (code and associated Helm charts, for example), will deploy properly without issues the first time.

I’ve seen people deploying to a dev cluster to test changes without first doing any checks locally. It’s a bit like Russian roulette, and the feedback loop is slow, especially if you need to first push a change to a repository and wait for CI workflows to complete, before you change is eventually deployed.

The local development workflow is set up well for running local unit tests, for example, and maybe running integration tests in a Docker Compose setup. There are also tools such as [Kubeconform](https://github.com/yannh/kubeconform) for linting and validating your manifests. But, when it comes to making sure your application will deploy onto Kubernetes without incident, things are less well defined.

This is where tools such as [Skaffold](https://skaffold.dev/), [Tilt](https://tilt.dev/), [Draft](https://github.com/Azure/draft/) or [Garden](https://garden.io/) come in.

I won’t go into the pros and cons of each tool here. There are other blog posts for that, but I will discuss why I decided on Skaffold and go into the workflow and setup for Skaffold locally.

![img](/skaffold-logo.png "Skaffold logo")

Skaffold is a tool from [Google](https://about.google/) designed to enable “Easy and Repeatable Kubernetes Development”. It’s available open-source [on GitHub](https://github.com/GoogleContainerTools/skaffold) and has extensive [documentation](https://skaffold.dev/docs/) available on its main website.

I decided to use Skaffold for my local Kubernetes development because it’s open-source, it’s lightweight, it’s easy to learn, it has the backing of Google who are essentially the creators of Kubernetes, it allows for running different types of tests against containers, and it enables continuous development by watching changes to your source-code in real-time and automatically building and deploying to your cluster.

You can install it using one of its [IDE plugins](https://skaffold.dev/docs/install/#managed-ide) or as a [standalone binary](https://skaffold.dev/docs/install/#standalone-binary). I decided to install using the [Homebrew package](https://skaffold.dev/docs/install/#homebrew) as I am running [macOS](https://www.apple.com/uk/macos/macos-sequoia/):

```bash
brew install skaffold
```

Once installed, you can browse to your application repository (such as this [Kubernetes Example Application](https://github.com/gawbul/kubernetes-example-application)) and initialise the Skaffold config using a simple:

```bash
skaffold init
```

As long as you have the basics set up, such as your source code, Helm charts, and Dockerfile, it should create a simple `skaffold.yaml` configuration file, much like the following (for builder I selected `Docker` and for buildpacks I selected `go.mod`):

```yaml
apiVersion: skaffold/v4beta13
kind: Config
metadata:
  name: kubernetes-example-application
build:
  artifacts:
    - image: ghcr.io/gawbul
      docker:
        dockerfile: Dockerfile
deploy:
  helm:
    releases:
      - name: kubernetes-example-application
        chartPath: helm-chart
        valuesFiles:
          - helm-chart/values.yaml
        version: 0.0.1
```

This would allow you to build an image for the relevant application based on the Dockerfile and deploy it using Helm to your Kubernetes cluster.

You could very simply run the following to build the image and deploy to your cluster:

```bash
skaffold run
```

However, there are a few things that aren’t considered here.

1. The image references GitHub Container Registry. We need to push the image to the registry and also set up `imagePullSecrets` to ensure the cluster can pull the image down (otherwise we’ll end up with `ImagePullBackOff`).
2. The image value isn’t set properly (it only includes the registry and the owner, and not the actual image name).
3. The values file is pointing at the Helm chart defaults, which may not include all the values we need.

This post won’t go into setting up your cluster to access GitHub Container Registry, although see the README [here](https://github.com/gawbul/kubernetes-example-application/blob/main/README.md) for a guide on how to set up [KinD](https://kind.sigs.k8s.io/) locally with all the relevant configuration.

A more appropriate `skaffold.yaml` to use once you have everything set up would be the following:

```yaml
apiVersion: skaffold/v4beta13
kind: Config
metadata:
  name: kubernetes-example-application
build:
  platforms: ["linux/arm64"]
  artifacts:
    - image: ghcr.io/gawbul/kubernetes-example-application
      docker:
        dockerfile: Dockerfile
        buildArgs:
          TARGETOS: linux
  local:
    push: true
deploy:
  helm:
    releases:
      - name: kubernetes-example-application
        chartPath: helm-chart/
        valuesFiles:
          - environments/common.values.yaml
          - environments/dev.values.yaml
        setValueTemplates:
          global.image.registry: "{{.IMAGE_REPO_ghcr_io_gawbul_kubernetes_example_application}}"
          global.image.tag: "{{.IMAGE_TAG_ghcr_io_gawbul_kubernetes_example_application}}@{{.IMAGE_DIGEST_ghcr_io_gawbul_kubernetes_example_application}}"
test:
  - image: ghcr.io/gawbul/kubernetes-example-application
    structureTests:
      - './structure-tests/*'
verify:
  - name: kubernetes-example-application-health-check
    container:
      name: kubernetes-example-application-health-check
      image: alpine/curl:8.14.1
      command: ['curl']
      args: ['-s', 'http://kubernetes-example-application.default.svc.cluster.local:8080']
    executionMode:
      kubernetesCluster: {}
```

This is based on the structure of the Kubernetes Example Application repository linked above.

It does a few things differently:

1. It sets the target platform as `linux/arm64` as I’m running on a [MacBook Pro](https://www.apple.com/uk/macbook-pro/) with [Apple silicon](https://en.wikipedia.org/wiki/Apple_silicon).
2. It adds a `TARGETOS` build-arg based on the setup in the Dockerfile for multi-platform builds.
3. It sets the local build argument `push` to `true` so that we build and push the image to the registry, rather than just storing it in the local build cache.
4. It adds separate values files, including a common values file with values relevant to all environments, and an environment-specific `dev` values file with values specific to the `dev` environment.
5. It uses `setValueTemplates` to override and inject the registry and tag into the values files, so that we can build dynamic image tags and have Skaffold be able to automatically reference them.
6. It adds a `test` section with a reference to [Container Structure Tests](https://github.com/GoogleContainerTools/container-structure-test) used for validating the built image.
7. It adds a `verify` section, which creates an ad-hoc container used to verify that the application has been deployed correctly.

Now, when we do a `skaffold run` it builds the image locally (if it doesn’t already exist) and pushes the image to GitHub Container Registry, it then runs the container structure tests to ensure the image was built correctly, before finally deploying the application to the Kubernetes cluster using Helm.

If you followed the guide above using a clone of the repository, you should then be able to do the following:

```bash
curl -k https://kubernetes.localhost:8443/kubernetes-example-application
```

Which returns the following:

```text
Hello from Kubernetes!%
```

You can then verify the deployment using:

```bash
skaffold verify
```

Which should return the following:

```text
Loading images into kind cluster nodes...  
Images loaded in 41ns  
[kubernetes-example-application-health-check] Hello from Kubernetes!
```

The steps above have allowed you to confirm that your image has been built correctly and is structured in the manner you expect, as well as verifying that it has been deployed and is running correctly in your Kubernetes cluster.

The test step can test various components of your image (see the [README] (https://github.com/GoogleContainerTools/container-structure-test/blob/main/README.md)for more details of the different tests), but in this case is just using base file existence and metadata tests:

```yaml
schemaVersion: 2.0.0

fileExistenceTests:
  - name: 'app directory should not exist'
    path: '/app'
    shouldExist: false
  - name: 'main binary should exist'
    path: '/main'
    shouldExist: true

metadataTest:
  cmd: ['/main']
  entrypoint: []
  labels:
    - key: 'org.opencontainers.image.source'
      value: 'https://github.com/gawbul/kubernetes-example-application'
    - key: 'org.opencontainers.image.description'
      value: 'Kubernetes Example Application'
    - key: 'org.opencontainers.image.title'
      value: 'Kubernetes Example Application'
  exposedPorts: ['8080']
  user: ''
  volumes: []
  workdir: '/'
```

This checks that an `/app` directory doesn’t exist (this should only exist in the build step), checks that a `/main` binary exists, and checks various elements of the image metadata, including the labels and exposed ports. All very useful checks for ensuring your image is structured as it should be.

To remove the resources from your local cluster once you’re happy, you just need to do the following:

```bash
skaffold delete
```

To build and test a standalone image, you can do the following:

```bash
skaffold build --filename=skaffold.yaml --file-output=build.json  
skaffold test --build-artifacts=build.json
```

The `build.json` is structured as follows:

```json
{
  "builds": [
    {
      "imageName": "ghcr.io/gawbul/kubernetes-example-application",
      "tag": "ghcr.io/gawbul/kubernetes-example-application:0a91c77@sha256:5df4b0c92b7435cfcd0d9be9d9ed5a095284882c07345fd895884c9fb61e2cdf"
    }
  ]
}
```

One of the most powerful pieces of functionality of Skaffold, however, is the ability to test changes to your code and Helm charts in real-time.

You can run the following command:

```bash
skaffold dev
```

This does the same as `skaffold run` but adds a control loop at the end, which watches your application for changes. You can then update the source code for your application or the associated Helm charts, and it automatically rebuilds the image, and/or redeploys the Helm chart as appropriate. Once you’re finished, you just hit Ctrl+C and it automatically cleans up for you.

This is valuable when you're trying to iterate on components of your application to debug issues with your code or deployment, for example.

[![Demonstrating Scaffold Dev](https://cdn.loom.com/sessions/thumbnails/5547f88ac64c4340a5a4ef75eb2ccbf1-26b880030fd1be3e-full-play.gif#t=0.1 "Demonstrating Skaffold Dev")](https://www.loom.com/share/5547f88ac64c4340a5a4ef75eb2ccbf1)

Skaffold isn’t just limited to single repository deployments either. If you have a selection of microservices that integrate together, you can use Skaffold to test your changes in one application alongside the others.

For example, you might have the following `skaffold-all.yaml` file:

```yaml
apiVersion: skaffold/v4beta13
kind: Config
metadata:
  name: kubernetes-micro-services
requires:
  - configs: ["kubernetes-micro-service-one"]
    git:
      repo: git@github.com:gawbul/kubernetes-micro-service-one.git
      path: skaffold.yaml
      ref: main
  - configs: ["kubernetes-micro-service-two"]
    git:
      repo: git@github.com:gawbul/kubernetes-micro-service-two.git
      path: skaffold.yaml
      ref: main
  - configs: ["kubernetes-micro-service-three"]
    git:
      repo: git@github.com:gawbul/kubernetes-micro-service-three.git
      path: skaffold.yaml
      ref: main
build:
  local:
    push: true
  artifacts:
    - image: ghcr.io/gawbul/kubernetes-micro-service-four
      docker:
        dockerfile: Dockerfile
deploy:
  helm:
    releases:
      - name: kubernetes-micro-service-four
        chartPath: helm-chart/
        valuesFiles:
          - environments/common.values.yaml
          - environments/dev.values.yaml
        setValueTemplates:
          global.image.registry: "{{.IMAGE_REPO_ghcr_io_gawbul_kubernetes_micro_service_four}}"
          global.image.tag: "{{.IMAGE_TAG_ghcr_io_gawbul_kubernetes_micro_service_four}}@{{.IMAGE_DIGEST_ghcr_io_gawbul_kubernetes_micro_service_four}}"
test:
  - image: ghcr.io/gawbul/kubernetes-micro-service-four
    structureTests:
      - './structure-tests/*'
verify:
  - name: kubernetes-micro-service-one-health-check
    container:
      name: kubernetes-micro-service-one-health-check
      image: alpine/curl:8.14.1
      command: ['curl']
      args: ['-s', 'http://kubernetes-micro-service-one.default.svc.cluster.local:8000/health']
    executionMode:
      kubernetesCluster: {}
  - name: kubernetes-micro-service-two-health-check
    container:
      name: kubernetes-micro-service-two-health-check
      image: alpine/curl:8.14.1
      command: ['curl']
      args: ['-s', 'http://kubernetes-micro-service-two.default.svc.cluster.local:8001/health']
    executionMode:
      kubernetesCluster: {}
  - name: kubernetes-micro-service-three-health-check
    container:
      name: kubernetes-micro-service-three-health-check
      image: alpine/curl:8.14.1
      command: ['curl']
      args: ['-s', 'http://kubernetes-micro-service-three.default.svc.cluster.local:8002/health']
    executionMode:
      kubernetesCluster: {}
  - name: kubernetes-micro-service-four-health-check
    container:
      name: kubernetes-micro-service-four-health-check
      image: alpine/curl:8.14.1
      command: ['curl']
      args: ['-s', 'http://kubernetes-micro-service-four.default.svc.cluster.local:8003/health']
    executionMode:
      kubernetesCluster: {}
```

This pulls in the Skaffold config from three separate micro-service repositories and allows you to build your current application alongside it to run integration and end-to-end tests if you wish.

Additionally, Skaffold can use any local kubeconfig to access clusters that may not be local. So, if you have access to a remote [GKE](https://cloud.google.com/kubernetes-engine) or [EKS](https://aws.amazon.com/eks/) cluster for example, then you can use it to deploy there too. This is especially useful if your remote cluster is configured in a way that might be more difficult to replicate locally, such as if there are specific [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) or [Kyverno](https://kyverno.io/) policies in place.

In order to visually validate that your application(s) have been deployed you can use [kubectl](https://kubernetes.io/docs/reference/kubectl/) to inspect the pods for example. However, one of the things the Skaffold is lacking that tools such as Tilt and Garden have, is a UI. This wasn’t something that bothered me, and the licensing implications meant that Skaffold was even more attractive.

If you want a UI to visualise your resources then there are plenty available. Two that I rate highly are [k9s](https://k9scli.io/) and [Headlamp](https://headlamp.dev/).

So, if you want to improve the speed of development when it comes to Kubernetes, I thoroughly recommend you check out Skaffold and spend some time learning about it’s capabilities. I’ve only touched the surface here. Check out the [website](https://skaffold.dev/) for much more detail on what it can do!
