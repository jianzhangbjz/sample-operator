### Background
The `sample-operator` was failed to install due to the `gcr.io/kubebuilder/kube-rbac-proxy` image pull error. Details as follows. 
To fix this issue, we need to build a new `kube-rbac-proxy` image and recreate the `quay.io/olmqe/learn-operator-index:v25` catalog source index image.
```go
I0307 09:11:20.740927 108278 helper.go:183] $oc get [csv -n e2e-test-default-dvh5r], the returned resource:
NAME                     DISPLAY           VERSION   RELEASE   REPLACES                PHASE
learn-operator.v0.0.3    Learn Operator    0.0.3               learn-operator.v0.0.2   Succeeded
sample-operator.v0.0.1   Sample Operator   0.0.1                                       Installing
I0307 09:11:23.741301 108278 client.go:761] Running 'oc --kubeconfig=/tmp/kubeconfig-4153306646 get pods -n e2e-test-default-dvh5r'
I0307 09:11:23.919271 108278 helper.go:183] $oc get [pods -n e2e-test-default-dvh5r], the returned resource:
NAME                                                 READY   STATUS             RESTARTS   AGE
sample-operator-controller-manager-879794b5d-bj76f   1/2     ImagePullBackOff   0          5m35s 

  Normal   Pulling         2m36s (x5 over 5m15s)  kubelet            Pulling image "gcr.io/kubebuilder/kube-rbac-proxy@sha256:db06cc4c084dd0253134f156dddaaf53ef1c3fb3cc809e5d81711baa4029ea4c"
  Warning  Failed          2m35s (x5 over 5m14s)  kubelet            Failed to pull image "gcr.io/kubebuilder/kube-rbac-proxy@sha256:db06cc4c084dd0253134f156dddaaf53ef1c3fb3cc809e5d81711baa4029ea4c": unable to pull image or OCI artifact: pull image err: initializing source docker://gcr.io/kubebuilder/kube-rbac-proxy@sha256:db06cc4c084dd0253134f156dddaaf53ef1c3fb3cc809e5d81711baa4029ea4c: reading manifest sha256:db06cc4c084dd0253134f156dddaaf53ef1c3fb3cc809e5d81711baa4029ea4c in gcr.io/kubebuilder/kube-rbac-proxy: manifest unknown: Requested entity was not found.; artifact err: get manifest: build image source: reading manifest sha256:db06cc4c084dd0253134f156dddaaf53ef1c3fb3cc809e5d81711baa4029ea4c in gcr.io/kubebuilder/kube-rbac-proxy: manifest unknown: Requested entity was not found.
  Warning  Failed          2m35s (x5 over 5m14s)  kubelet            Error: ErrImagePull
  Normal   BackOff         10s (x21 over 5m5s)    kubelet            Back-off pulling image "gcr.io/kubebuilder/kube-rbac-proxy@sha256:db06cc4c084dd0253134f156dddaaf53ef1c3fb3cc809e5d81711baa4029ea4c"
  Warning  Failed          10s (x21 over 5m5s)    kubelet            Error: ImagePullBackOff
```
### Steps
1. Build the `kube-rbac-proxy` image
```console
jiazha-mac:goproject jiazha$ git clone https://github.com/openshift/kube-rbac-proxy.git
Cloning into 'kube-rbac-proxy'...
...
jiazha-mac:kube-rbac-proxy jiazha$ podman build --platform linux/amd64,linux/arm64,linux/ppc64le,linux/s390x  --manifest quay.io/olmqe/kube-rbac-proxy:v4.22 . -f Dockerfile.ocp 
[linux/amd64] [1/2] STEP 1/6: FROM registry.ci.openshift.org/ocp/builder:rhel-9-golang-1.25-openshift-4.22 AS builder
...
...
jiazha-mac:kube-rbac-proxy jiazha$ podman manifest push quay.io/olmqe/kube-rbac-proxy:v4.22
Getting image list signatures
...
```
2. Replace the old `kube-rbac-proxy` image with this new one: `quay.io/olmqe/kube-rbac-proxy:v4.22` for the sample-operator.
3. Update the `installMode` of `sample-operator` to support `multi-namespaces`.
4. Build the `sample-operator` image.
```console
jiazha-mac:sample-operator jiazha$ podman build --platform linux/amd64,linux/arm64,linux/ppc64le,linux/s390x  --manifest quay.io/olmqe/sample-operator:v4.22 .
[linux/s390x] STEP 1/6: FROM quay.io/operator-framework/ansible-operator:v1.19.0
Trying to pull quay.io/operator-framework/ansible-operator:v1.19.0...
...
jiazha-mac:sample-operator jiazha$ podman manifest push quay.io/olmqe/sample-operator:v4.22
Getting image list signatures
...
```
5. Generate the `bundle.Dockerfile`
```console
$ make bundle IMG=quay.io/olmqe/sample-operator@sha256:e83e6ae660fe7c94ec61616c622dce6640bf912667ede0c17881e10ad8812be3
```
6. Build the bundle image
```console
jiazha-mac:sample-operator jiazha$ podman build --platform linux/amd64,linux/arm64,linux/ppc64le,linux/s390x  --manifest quay.io/olmqe/sample-operator-bundle:v4.22 -f bundle.Dockerfile 
[linux/s390x] STEP 1/14: FROM registry.access.redhat.com/ubi8/ubi-minimal:latest
...
jiazha-mac:sample-operator jiazha$ podman manifest push quay.io/olmqe/sample-operator-bundle:v4.22
Getting image list signatures
...
```
7. Update the kube-rbac-proxy, sample-operator and bundle images in the `catalog/sample-operator/catalog.json` file.
```console
jiazha-mac:sample-operator jiazha$ tree catalog
catalog
├── learn
│   └── catalog.json
└── sample-operator
    └── catalog.json

3 directories, 2 files
```
8. Build the index image.
```console
jiazha-mac:sample-operator jiazha$ podman build --platform linux/amd64,linux/arm64,linux/ppc64le,linux/s390x  --manifest quay.io/olmqe/learn-operator-index:v25 . -f catalog.Dockerfile
[linux/s390x] [1/2] STEP 1/3: FROM registry.redhat.io/openshift4/ose-operator-registry-rhel9:v4.19 AS builder
...
jiazha-mac:sample-operator jiazha$ podman manifest push quay.io/olmqe/learn-operator-index:v25
Getting image list signatures
...
```
