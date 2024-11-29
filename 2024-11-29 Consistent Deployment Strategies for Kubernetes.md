---
layout: post
title: Consistent Deployment Strategies for Kubernetes
date: 2024-11-29 00:00:00 +0000
categories: SRE, DevOps, Kubernetes, CloudNative
---

There are a few different ways to deploy manifests to Kubernetes. Picking one and sticking with it can be difficult due to lack of support. Some resources support Helm charts, some prefer jsonnet, and then some only support Kustomize. 

Every manifest that is deployed to Kubernetes should be deployed in the same fashion. This will make it massively easier for the Site Reliability Engineering team to manage the infrastructure and have no doubt what to use when deploying any piece of software. 

**Simple, is always easier, and often times better.** 

**Consistency over Correctness**. If I deploy one application with helm, I'd like to deploy everything with helm just to ensure that there is no doubt when deploying anything new. Working over a consistent codebase is always easier, then one that switches back and forth creating more ambiguity. 


This post will explore deploying applications with Kustomize, going over two different examples. 
1. Creating a simple Kubernetes application.
2. Using the helm plugin to create an application that uses Helm, with Kustomize.
But first I will cover why I like Kustomize in the first place.

## Why Kustomize?

It is simple and easy with minimal overhead to deploy a single application to a single cluster. 
It makes extending an application and applying patches to multiple clusters really simple, no real additional configuration is necessary, aside from a few potentially semi-duplicate manifests in directories prefixed with their cluster-name. It is still possible to deploy helm applications due to the Helm integration, and while doing so, you still have all the benefits from using Kustomize. 

## Deploying an Application with Kustomize


I will walk through two different examples. The first one will include deploying a simple application with Kustomize across multiple clusters, and the second one will include deploying a helm application using Kustomize across multiple clusters.

From my last blog post, the directory structure in our gitops repository looks similar to:
```
gitops/
|-birdbox/
|--base/
|  |--kustomization.yaml
|  |--deploy.yaml
|--overlays/
|--|--dev/
|  |---kustomization.yaml
|  |---ingress.yaml
|  |---secrets.yaml
|  |---service-account-patch.yaml
|--|--prd/
|  |---kustomization.yaml
|  |---ingress.yaml
|  |---secrets.yaml
|  |---service-account-patch.yaml
```

Now to deploy this application, all we need to do is point ArgoCD's ApplicationSet at the `kustomization.yaml` in the `{{cluster}}` overlay. It will use that `kustomization.yaml` and grab the one from the `base` directory and apply all of those manifests. 
To give a more concrete example, the `kustomization.yaml` in the `dev` directory is:
```kustomization.yaml
---
namespace: birdbox
resources:
  - ../../base
  - ./secrets.yaml
  - ./ingress.yaml
patchesStrategicMerge: # this is deprecated
  - ./service-account-patch.yaml
images:
  - name: org/birdbox
    newName: image-repo:birdbox
    newTag: "2AB9Dd4FF"
# add annotation to all resources for 'dev'
commonAnnotations:
  environment: dev
  language: golang
  repo: birdbox
```


This is really nice, it shows us exactly what image should be deployed, what resources we are patching, and the annotations that will be applied to the objects that are going to be deployed. This is easy to debug too because Kustomize comes bundled with `kubectl` .  This means we can hop in the `{{cluster}}` directory, in this case `dev` and run `kubectl kustomize . > outputs.yaml` This will give us an `outputs.yaml` that has all of our manifests from this directory and the subsequent `base` directory, that we can verify prior to deployment. 

As of now, for simplicity I prefer updating these manifests via CI/CD PRs updating:
```
images:
  - name: org/birdbox
    newName: image-repo:birdbox
    newTag: "2AB9Dd4FF"
```
the `newTag` field to point to the new image. This makes it so we do not have to worry about versioning and the tags are directly linkable to a commit in our application repository. 


## Kustomize with Helm


In the case of an application like DataDog, or Grafana Loki, it might be a lot simpler to use helm, which seems to be the normal way to deploy most applications that are bundled up from the various vendors. With the Helm Plugin and Kustomize though, we can stick to our consistent approach and use Kustomize for this too.

In this case our Kustomize manifest will look like:
```
---
namespace: datadog

resources:
  - ../../base
  - ./secrets.yaml

helmCharts:
  - name: datadog
    namespace: datadog  # this sets the namespace for {{ .Relase.Namespace }} in the helm chart
    includeCRDs: true
    valuesFile: datadog-values.yaml
    releaseName: datadog
    version: datadog-3.81.0
    repo: https://helm.datadoghq.com
```

This ends up following our previous pattern. Gives us a nice way to deploy our custom `values.yaml` and ends up being pretty readable!

And again, it is really simple to debug with:
```
kubectl kustomize .  --enable-helm > outputs.yaml
```


I want to clarify something that has bugged me while writing these blog posts. The `secrets.yaml` file that has popped up are not raw secrets. It is an object that references secrets for our cloud provider to download and sync them into our cluster.


## Conclusion

By using Kustomize, we can ensure that all of our manifests are deployed in a consistent manner. It gives us the ability to deal with raw kubernetes manifests and provides an easy enough interface to patch objects so that I do not feel like I am missing out when configuring manifests. Of course, we do not get the full benefit of helm templates, but I still prefer simple manifests that can be referenced and then patched when necessary. Kustomize also offers a nice easily debuggable approach that is consistent across all of our applications. 





