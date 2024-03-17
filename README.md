# progressive-delivery
Progressive delivery with flux, istio and flagger

## Introduction

This repository holds a demo for a platform built on kubernetes. It is built using gitops principles:

- Flux - GitOps engine for Continuous delivery 
- Istio - service mesh
- Flagger - progressive delivery


## Prerequisites 

### Kubernetes cluster

This demo will be made using [k3d](https://k3d.io/#installation) cluster with three nodes:
```
k3d cluster create staging -p 8080:80@loadbalancer --agents 3
k3d cluster create production -p 8080:80@loadbalancer --agents 3
```

### flux CLI
```
curl -s https://fluxcd.io/install.sh | sudo bash
```

### GitHub account

```
export GITHUB_TOKEN=<YOUR-TOKEN>
export GITHUB_USER=<YOUR_GITHUB_USER>
```

## Repository structure

This repository follows [Flux's Monorepo structure](https://fluxcd.io/flux/guides/repository-structure/#repository-structure). It has also a directory named `applications` where the code for applications used for the demo are stored.

```
├── applications
├── apps
│   ├── base
│   │   ├── namespace-a
│   │   └── namespace-b
│   ├── production 
│   └── staging
│       ├── patch-file-1
│       └── patch-file-2
├── infrastructure
│   ├── base
│   │   ├── _sources
│   │   ├── manifests
│   │   │   ├── namespace-a
│   │   │   └── namespace-b
│   │   └── releases
│   │       ├── namespace-a
│   │       └── namespace-b
│   ├── production 
│   └── staging
│       ├── manifests
│       │   ├── namespace-a
│       │   └── namespace-b
│       └── releases
│           ├── namespace-a
│           └── namespace-b
└── clusters
    ├── production
    └── staging
```

- `applications` - is used to store application code for creating test images
- `apps` - is used to configure the deployment of applications
- `infrastracture` - is used to configure the deployment of services that might be used by multiple applications, like monitoring,networking, etc.
  - `base` - is used to store configurations that will be shared by all clusters
    - `_sources` - is used to store the sources of the helm releases. Useful because multiple HRs share sources.
    - `releases` - is used to store the helm releases
    - `manifests` - is used to store k8s manifests of CRDs that are configured by the HRs stored in `releases`. For that reason, this folder is applied after releases.
  - `staging`- is used to patch the configuration stored in 'base' to the specific staging values. 
- `clusters` - is used to define the clusters

The `apps` folder depend on the `infrastructure` for istio-proxy to be injected into the pods, for that reason flux will apply the kustomizations in the following order:

- infrastructure-releases
- infrastructure-manifests
- apps

## Bootstrap

### Staging
```
flux bootstrap github \
  --components-extra=image-reflector-controller,image-automation-controller \
  --context=k3d-staging \
  --owner=$GITHUB_USER \
  --repository=progressive-delivery \
  --branch=main \
  --path=clusters/staging \
  --read-write-key \
  --personal \
  --token-auth
```

Flux will automatically install:
- install Istio using the Istio base, istiod and gateway Helm charts
- install Flagger, Prometheus and Grafana
- creates the Istio public gateway 

### Access ghcr.io container registry

kubectl create -n applications secret docker-registry k8s-ghcr \
--docker-server=https://ghcr.io \
--docker-username=eduardodbr \
--docker-password=<redacted> \
--docker-email=eduardodbr@hotmail.com

## Applications

The [applications](applications/) directory stores application code that is used to test tools and scenarios.

- [echo-version](applications/echo-version/) is a [benthos](https://benthos.dev) app used as a simple API that echoes a version.

## Progressive delivery

This platform does progressive delivery using Istio, Flagger and FluxCD. Whenever a new application version is pushed to the container registry, flux automatically detects the change and applies it to the cluster if the new version matches a set of policies. When that new version is applied, flagger detects that change and starts the canary deployment by changing istio's virtual service routing weights. If the canary is successful, flagger replaces the current primary version with the canary one that has the new version.
