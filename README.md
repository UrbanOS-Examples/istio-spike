### Deploying Istio Common Infrastructure
first edit the ACM cert in the istio-demo-config.yaml file

then run the k8s manifest generator to make some manifests
```sh
istioctl manifest generate -f istio-demo-config.yaml 2>&1 > istio-demo-manifest.yaml
```

then run a k8s apply to spin up istio
```sh
k apply -f istio-demo-manifest.yaml
```

you should see an ELB in AWS that has the right cert set up for port 443

you then have to apply a fix so istio's proxies don't mess with AWS ELB proxy headers (will result in a redirect loop otherwise)
```sh
k -n istio-system apply -f istio-envoy-filter-elb-fix.yaml
```

now deploy a specially configured external dns setup so host names can be configured
```sh
k -n cluster-infra apply -f istio-external-dns-security.yaml
k -n cluster-infra apply -f istio-external-dns.yaml
```

### Deploying specific apps 
First, the namespace needs to be configured for istio to auto inject proxies for each app in question. Re-deploying or deleting the pods injects a proxy sidecar used for outbound service discovery
```sh
kubectl label namespace discovery istio-injection=enabled
k -n discovery scale deployment/discovery-ui --replicas=0
k -n discovery scale deployment/discovery-ui --replicas=1
k -n discovery scale deployment/discovery-api --replicas=0
k -n discovery scale deployment/discovery-api --replicas=1
```

Istio versions of discovery API and UI can be deployed by applying a set of gateways and virtual services for each. Note that you probably don't need a different gateway per app, but the external-dns setup above is only set to listen on gateways to that is how this demo is configured.
```sh
k -n discovery apply -f istio-discovery-ing.yaml
```

Each app should now be available
```sh
curl https://discovery-ui-istio.dev.internal.smartcolumbusos.com
curl https://discovery-api-istio.dev.internal.smartcolumbusos.com/api/v1/dataset/search
```
