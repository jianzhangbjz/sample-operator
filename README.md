Create an ansible-based operator by following: https://github.com/Xia-Zhao-rh/multi-arch-operators/blob/master/nginx-operator-24644/steps/steps.txt

1. Create Bundle image.
```yaml
[cloud-user@preserve-olm-env2 sample-operator]$  make bundle IMG=quay.io/olmqe/sample-operator@sha256:b4bd0b50ee0b1630d7b5da4d8c1c604fc154b1cea420c34d42615f9da2e9d770
operator-sdk generate kustomize manifests -q
cd config/manager && /data/jian/sample-operator/bin/kustomize edit set image controller=quay.io/olmqe/sample-operator@sha256:b4bd0b50ee0b1630d7b5da4d8c1c604fc154b1cea420c34d42615f9da2e9d770
/data/jian/sample-operator/bin/kustomize build config/manifests | operator-sdk generate bundle -q --overwrite --version 0.0.1  
INFO[0000] Creating bundle.Dockerfile                   
INFO[0000] Creating bundle/metadata/annotations.yaml    
INFO[0000] Bundle metadata generated suceessfully       
operator-sdk bundle validate ./bundle
INFO[0000] All validation tests have completed successfully
```
2. Add the bundle image to an exist index image.
```yaml
[cloud-user@preserve-olm-env2 sample-operator]$ opm index add -f quay.io/olmqe/learn-operator-index:v1 -b quay.io/olmqe/sample-operator-bundle@sha256:3487d4327c469d549d948d01ada0cd28a39e44941603c19e2cce27f2ce559300 -t quay.io/olmqe/learn-operator-index:v1
...
```
