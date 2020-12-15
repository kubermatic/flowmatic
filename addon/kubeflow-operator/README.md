# KubeFlow Addon for Kubermatic Kubernetes Platform
This is a KubeFlow [addon for Kubermatic Kubernetes Platform](https://docs.kubermatic.com/kubermatic/master/advanced/addons/).

The KubeFlow addon provides the following config option:
- `IstioIngressGatewayServiceType`  (string): this is the type of istioingressgateway service, it can be set to `ClusterIP`, `NodePort` or `LoadBalancer`.

## Example Addon Resources
For the KubeFlow addon:
```yaml
apiVersion: kubermatic.k8s.io/v1
kind: Addon
metadata:
  name: kubeflow-operator
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
  name: kubeflow-operator
  variables: {"IstioIngressGatewayServiceType":"LoadBalancer"}
```

## Addon Maintenance
The manifest.yaml file contains three parts:
- A namespace to deploy KubeFlow operator.
- Manifest of KubeFlow operator, which can be generate by following [KubeFlow operator page](https://github.com/kubermatic/kfctl/blob/master/operator.md#deployment-instructions)
- A KfDef which is used to config KubeFlow installation.
