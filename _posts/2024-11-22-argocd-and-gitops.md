---
layout: post
title:  "GitOps Across Clusters - How ArgoCD and Kustomize Makes It Simple"
date:   2024-11-22 00:00:00 +0000
categories: SRE, DevOps, Kubernetes, CloudNative
---

Working with Kubernetes is fun and rewarding, wait... did I get that right? Overwhelming and complex... Just Kidding, I really do enjoy working with Kubernetes. 

If a Site Reliability Engineer were to manage several clusters and lots of different applications in those clusters, it could quickly become stressful. 

Even just thinking about having to run `kubectl apply` manually makes me sweat. 

Instead, everything inside of our various Kubernetes clusters is done automatically via ArgoCD's continuous polling of our GitOps repository. 

To better understand why GitOps is an optimal solution, we should know the four principles that sort of make up the idea of GitOps. 
1. It's Declarative, your end state is defined.
2. Application manifests are versioned and their history is stored
3. Application manifests are continuously pulled 
4. State is continuously reconciled

Now to get to the fun part. 

## What is ArgoCD and why use it?
[ArgoCD](https://argo-cd.readthedocs.io/en/stable/#why-argo-cd)
>Why Argo CD?
>Application definitions, configurations, and environments should be declarative and version controlled. Application deployment and lifecycle management should be automated, auditable, and easy to understand

It embodies the principles governing GitOps and becomes a great tool for the job. It automatically polls our gitops repository and continuously reconciles it, making sure that the desired state is the current state in our Kubernetes cluster. This makes it a lot easier for anyone operating within the cluster to understand the desired state. 

## What is Kustomize and why use it?
[Kustomize](https://kustomize.io/)
>Kustomize introduces a template-free way to customize application configuration that simplifies the use of off-the-shelf applications.

Kustomize makes it simple to define your application manifest in a `base` directory and then define either patches or additional components in an `overlay` directory.
I'll go over more examples later in this post. 

## Why not helm? 

I don't love templating yaml manifests, in fact, I don't think anyone finds it enjoyable! I also do not want to have to account for new fields in kubernetes manifests as they become available. 
Applying helm charts is easy enough, but in my experience it becomes frustrating maintaining helm charts. At first, the Helm chart starts as a general-purpose application chart, but over time, more `if` statements are added until the entire template becomes difficult to read.

I have not yet had that experience with Kustomize.

## How To Effectively use ArgoCD to Deploy Across Multiple Clusters
Argo has two main resources that can be used to deploy applications. One is an `Application` and the other is an `ApplicationSet`. I tend to prefer the `ApplicationSet` resource. Pairing ArgoCD with Kustomize makes it incredibly easy to set up and maintain multiple applications across multiple clusters. 

To go off on an example, say we want to deploy an application called BirdBox in our Kubernetes clusters (dev, uat, and prd). We could:
Create 3 `Application` resources and configure ArgoCD to deploy each of them.
or
Make 1 `ApplicationSet` and use a few variables to determine the cluster and path to apply the application automatically across the clusters.

To get into how we define these `ApplicationSet` resources it will be easier to describe our gitops repository layout first. 

We use Kustomize for all of the applications we support and the structure typically is:
```
gitops
├── README.md
├── birdbox
│   ├── base
│   │   ├── kustomization.yaml
│   │   ├── deploy.yaml
│   │   ├── service-account.yaml
│   │   └── service.yaml
│   └── overlays
│       ├── dev
│       │   ├── kustomization.yaml
│       │   ├── ingress.yaml
│       │   ├── secrets.yaml
│       │   ├── service-account-patch.yaml
│       │   ├── hpa-deploy.yaml
│       └── uat
│       │   ├── kustomization.yaml
│       │   ├── ingress.yaml
│       │   ├── secrets.yaml
│       │   ├── service-account-patch.yaml
│       │   ├── hpa-deploy.yaml
│       └── prd
│       │   ├── kustomization.yaml
│       │   ├── ingress.yaml
│       │   ├── secrets.yaml
│       │   ├── service-account-patch.yaml
│       │   ├── hpa-deploy.yaml
```

Now, back to the `ApplicationSet` resource. 
```
---
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: birdbox
  namespace: argocd
spec:
  ignoreApplicationDifferences:
    - jsonPointers:
        - /spec/syncPolicy
  generators:
    - list:
        elements:
          - cluster: dev
            url: https://xxx.xxx.xxx.xxx.xxx.com
          - cluster: uat
            url: https://xxx.xxx.xxx.xxx.xxx.com
          - cluster: prd
            url: https://xxx.xxx.xxx.xxx.xxx.com
  template:
    metadata:
      name: '{{cluster}}-birdbox'
    spec:
      project: default
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
      source:
        repoURL: https://github.com/${ORG}/gitops.git
        targetRevision: main
        path: birdbox/overlays/{{cluster}}
      destination:
        server: '{{url}}'
        namespace: birdbox
```

This ApplicationSet will dynamically create the application across the various clusters: `{{cluster}}-birdbox`
We can use the name of the `{{cluster}}` to dynamically apply the correct path for the application, which matches our gitops application structure.

To be more specific. In our `dev` cluster, this `ApplicationSet` will look in our `gitops` repository under the path: `birdbox/overlays/dev` and that `kustomization.yaml` manifest will reference the `kustomization.yaml` manifest located in the base of the application directory. This makes it so we only have to define our app manifests once, and then we can define more cluster specific manifests in their own directories. In the above example, we apply a `service-account-patch.yaml` to patch an annotation on our service accounts to link them to the IAM Roles for Service Accounts (IRSA). We also tend to keep `ingress.yaml` defined at the environment layer, due to differing Ingress-nginx annotations and the lack of desire to patch everything. 


ApplicationSet's also provide an easy way to ignore certain differences in applications:
```
  ignoreApplicationDifferences:
    - jsonPointers:
        - /spec/syncPolicy
```
This allows us to quickly disable application auto syncing in the ArgoCD UI in case of an emergency where we need to either patch something, while we work on a more declarative fix. It also helps in the case that ArgoCD might start thrashing if some how the application state does not match what is in the GitOps repository. 

## ApplicationSets to rule Applications
Earlier I mentioned that I never want to have to run `kubectl apply`, and while that is true, there is one manifest that still needs applied manually: the `Application` that maintains the other `ApplicationSets`. This resource is an `Application`
```
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: applicationset-controller
  namespace: argocd
spec:
  project: default
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
  destination:
    name: ''
    namespace: argocd
    server: https://kubernetes.default.svc
  source:
    repoURL: https://github.com/${ORG}/gitops.git
    targetRevision: main
    path: argocd/appsets
```

Now, with this resource, whenever anyone is to PR a new `ApplicationSet` to our gitops repository, it will automatically sync that `ApplicationSet` to ArgoCD, and then to the various clusters. This enables us to implement well-scoped Role-Based Access Controls (RBAC) to ensure our compliance with the four principles of GitOps. The only exception is for production related emergencies, which usually involve a breakglass situation.

In an attempt to keep these short and sweet I will cover how to deploy new application images and go further in depth in to our Kustomize manifests in the next post! 
