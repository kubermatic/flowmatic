# KubeFlow Addon for Kubermatic Kubernetes Platform
This is a KubeFlow [addon for Kubermatic Kubernetes Platform](https://docs.kubermatic.com/kubermatic/master/advanced/addons/).
Currently, we have 2 KubeFlow addons:
- `kubeflow` is the addon with manifest on KubeFlow master branch.
- `kubeflow.v1.2.0` is the addon with manifest on KubeFlow v1.2-branch. For deploying this addon, extra tweaking of apiserver flags is needed to enable [service account token volume projection](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#service-account-token-volume-projection) for istio.

The KubeFlow addon provides the following optional config options:
- `ExposeLoadBalancer` (boolean): If true, the Kubeflow dashboard will be exposed via a `LoadBalancer` service instead of a `NodePort`.
- `EnableTLS` (boolean): If true, TLS will be enabled, and a certificate will be automatically issued for the specified `DomainName`.
- `NVIDIAOperator` (boolean): If true, NVIDIA GPU Operator will be installed. Also installs Node Feature Discovery for Kubernetes.
- `DomainName` (text): Domain name that will be used to access the KubeFlow dashboard. Make sure to set up you DNS accordingly.
- `OIDCProviderURL` (text): URL of external OIDC provider, e.g. Kubermatic Dex instance. Make sure that `DomainName` is specified if you are using this. If not provided, static user authentication will be used.
- `OIDCSecret` (text): Secret string shared between the OIDC provider and KubeFlow. If not provided, [this default](https://github.com/kubeflow/manifests/blob/master/istio/oidc-authservice/base/params.env#L5) is used (insecure!).

## Example Addon Resources
For the Kubeflow addon:
```yaml
apiVersion: kubermatic.k8s.io/v1
kind: Addon
metadata:
  name: kubeflow
  namespace: cluster-bjg9qmhctj
  ownerReferences:
  - apiVersion: kubermatic.k8s.io/v1
    blockOwnerDeletion: true
    controller: true
    kind: Cluster
    name: bjg9qmhctj
    uid: 30c13f8b-235e-4512-8673-d9b0f3a41f27
spec:
  cluster:
    kind: Cluster
    name: bjg9qmhctj
    uid: 30c13f8b-235e-4512-8673-d9b0f3a41f27
  name: kubeflow
  variables:
    ExposeLoadBalancer: true
    EnableTLS: true
    DomainName: kubeflow.mydomain.io
    OIDCProviderURL: https://dev.kubermatic.io/dex
    OIDCSecret: NQh4P9fIDlEyI6EMKW66TLKLdcIStT4C02
```

## DNS Setup
If using `ExposeLoadBalancer=true`, to access the Kubeflow dashboard, finish the DNS setup of you installation. In order to do that, retrieve the external IP / DNS name allocated for the istio-ingressgateway service:
```bash
$ kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP                                                                  PORT(S)                                                                                                                                      AGE
istio-ingressgateway   LoadBalancer   10.240.28.214   a286f5a47e9564e43ab4165039e58e5e-1598660756.eu-central-1.elb.amazonaws.com   15020:31655/TCP,80:31380/TCP,443:31390/TCP,31400:31400/TCP,15029:32743/TCP,15030:30831/TCP,15031:32599/TCP,15032:30819/TCP,15443:31158/TCP   21m
```
Now setup your DNS at your domain provider to point to the allocated `EXTERNAL-IP`, e.g. using a `CNAME` DNS entry.

## OIDC Provider Configuration
The plugin allows pointing to an external OIDC provider of user's choice. To have OIDC working, the provider itself needs to be configured as well - e.g. in case it is a Dex instance, a `staticClients` entry needs to be added:
```yaml
  staticClients:
    - RedirectURIs:
        - http://kubeflow.mydomain.io/login/oidc
      id: kubeflow-oidc-authservice
      name: kubeflow-oidc-authservice
      secret: NQh4P9fIDlEyI6EMKW66TLKLdcIStT4C02
```

## TLS Configuration
If TLS is enabled, the addon will automatically issue a certificate for the specified `DomainName` using cert-manager and Let's Encrypt via ACME. However, due to some Istio-related issues, Istio configuration needs to be manually adapted with a new `Gateway` and `VirtualService` to let ACME challenge pass to the solver pods.

Check the specification of an ingress that was automatically created for the acme-solver and use its output to configure the `Gateway` and `VirtualService`:
```shell
$ kubectl get ingress -n istio-system -o yaml
```

Gateway configuration (does not need to be modified):
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: challenge
  namespace: kubeflow
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - '*'
    port:
      name: http
      number: 80
      protocol: HTTP
```

VirtualService configuration (modify `hosts`, `prefix` and `host`):
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: challenge
  namespace: kubeflow
spec:
  gateways:
  - challenge
  hosts:
  - 'kubeflow.mydomain.io'
  http:
  - match:
    - method:
        exact: GET
      uri:
        prefix: /.well-known/acme-challenge/PQILQ5DARUGg0woJ19lLhNfU-HThRNndmb_epggYv78
    route:
    - destination:
        host: cm-acme-http-solver-frnh4.istio-system.svc.cluster.local
        port:
          number: 8089
```

After the certificate is successfully issued, the `Gateway` and `VirtualService` can be removed. This can be verified as follows:
```shell
$ kubectl get certificaterequests -n istio-system
NAME                                    READY   AGE
istio-ingressgateway-certs-3283355345   True    12m
```

## Addon Maintenance
The manifest.yaml file contains three parts:
- A namespace to deploy KubeFlow operator.
- Manifest of KubeFlow operator, which can be generated by following [KubeFlow operator page](https://github.com/kubermatic/kfctl/blob/master/operator.md#deployment-instructions)
- A KfDef which is used to config KubeFlow installation.
