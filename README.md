# Quickstart OpenSearch-k8s-operator on AKS within 10 minutes

## Introduction

Walking through the Quickstart that is part of the [Operator User guide](https://github.com/Opster/opensearch-k8s-operator/blob/main/docs/userguide/main.md) doesn't mention some AKS details, which will result in Pod failures. This frustated some of my colleagues, so I have tried to deploy OpenSearch with the OpenSearch-k8s-Operator on AKS. Below the quickstart how to deploy and use the Operator successfull on Azure provided AKS clusters.

## AKS Deployment

### Deployment steps

By default most settings for Kubelet and Linux OS are set to default. Most use cases this seems okay, but with OpenSearch this can give you some challenges. Opensearch clusters require some important settings for Production workloads regarding memory related kernel settings, which is further described in the [Opensearch documentation](https://opensearch.org/docs/latest/install-and-configure/install-opensearch/index/#important-settings).

Important is that we provide a custom node configuration to our AKS deployment, especially for the `vm.swappiness` and `vm.max_map_count` settings.
To provide custom settings we can use an additional input that is provided by an Azure extension called `aks-preview`. For simplicity we are using the `AZ CLI using the Cloud Shell`, but also possible in a Bicep resource definition. 

Below the sample of our JSON input we will use.

```
./linuxosconfig.json
{
 "sysctls": {
  "vmMaxMapCount": 262144,
  "vmSwappiness": 0
 }
}
```

More information about custom node settings can be found on [Microsoft Learn](https://learn.microsoft.com/en-us/azure/aks/custom-node-configuration).

Below the command to create a default cluster with a custom node configuration. In some cases install or update the extension that will be used.

```
az extension add --name aks-preview
az aks create --name OpenSearchAKSCluster --resource-group rg-opensearch-testdrive --linux-os-config ./linuxosconfig.json
```

### Validate Kernel settings

After a succesfull creation we can connect to the cluster with Cloud Shell by using `kubectl`.

```
Requesting a Cloud Shell.Succeeded. 
Connecting terminal...

az aks get-credentials --resource-group rg-opensearch-testdrive --name OpenSearchAKSCluster
kubectl get nodes
```

For looking into the AKS node I will execute a special debug feature and use `cat` in the node to validate the kernel loaded settings.

```
kubectl debug node/aks-nodepool1-1234567-vmss000000 -it --image=mcr.microsoft.com/dotnet/runtime-deps:6.0
cat /proc/sys/vm/swappiness
0
cat /proc/sys/vm/max_map_count
262144
```

## Opensearch-k8s-operator deployment

### Helm walkthrough

Great. We are ready to deploy the OpenSearch operator. Let's add the repo and execute the installation.

```
helm repo add opensearch-operator https://opster.github.io/opensearch-k8s-operator/
helm repo update
helm install opensearch-operator opensearch-operator/opensearch-operator
```

The following is expected.

```
NAME: opensearch-operator
LAST DEPLOYED: Thu Apr 27 08:37:33 2023
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

### Validate Operator objects

We can use the following command to validate if the `CRD` Objects are well created.

```
kubectl get crd | grep opensearch
```

Also check the actual operator controller manager pod is running:

```
kubectl get pods
NAME                                                     READY   STATUS      RESTARTS   AGE
opensearch-operator-controller-manager-977b5b4bb-s9g69   2/2     Running     0          74m
```

## Create sample OpenSearch Cluster

Great we are now at the stage we can actually deploy an OpenSearch cluster. 

We will create a cluster with three (3) OS nodes {roles: data, cluster manager} with one (1) Dashboard instance.
Copy the following one-liner and execute this.

```
kubectl apply -f - <<EOF
apiVersion: opensearch.opster.io/v1
kind: OpenSearchCluster
metadata:
  name: qensus-labs-testdrive
  namespace: default
spec:
  general:
    serviceName: qensus-labs-testdrive
    version: 2.3.0
  dashboards:
    enable: true
    version: 2.3.0
    replicas: 1
    resources:
      requests:
         memory: "512Mi"
         cpu: "200m"
      limits:
         memory: "512Mi"
         cpu: "200m"
  nodePools:
    - component: nodes
      replicas: 3
      diskSize: "5Gi"
      nodeSelector:
      resources:
         requests:
            memory: "2Gi"
            cpu: "500m"
         limits:
            memory: "2Gi"
            cpu: "500m"
      roles:
        - "cluster_manager"
        - "data"
EOF
```

You can watch the cluster becomes available using `kubectl get pods`. Status must become *Running*

```
kubectl get pods
NAME                                                     READY   STATUS      RESTARTS   AGE
qensus-labs-testdrive-dashboards-69487b4bbd-d6zdp        1/1     Running     0          71m
qensus-labs-testdrive-nodes-0                            1/1     Running     0          71m
qensus-labs-testdrive-nodes-1                            1/1     Running     0          70m
qensus-labs-testdrive-nodes-2                            1/1     Running     0          68m
```

## Create an Ingress for external access

Last step is to expose the OpenSearch dashboard using an Ingress.

### Deploy a basic Ingress using Nginx.

Just follow the following [Microsoft Learn article](https://learn.microsoft.com/en-us/azure/aks/ingress-basic)

Follow should be self-explainable, since previously we also used Helm.

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --create-namespace \
  --namespace ingress-basic \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz
```

Now validate if an Nginx ingress service has been created with an external IP address. Ingress can expose both HTTP and HTTPS traffic.

```
kubectl get svc -n ingress-basic

NAME                                 TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.0.208.120   40.38.160.93   80:31527/TCP,443:31074/TCP   52m
```

### Creating Ingress object for opensearch-dashboards

Now just apply this YAML.

```
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: opensearch-dashboards
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: qensus-labs-testdrive-dashboards
            port:
              number: 5601
EOF
```

Et voila, we have a cluster within 10 minutes !!!  And don't forget to update your password asap

<img src="https://raw.githubusercontent.com/avwsolutions/quickstart-aks-opensearch-k8s-operator/main/cluster-opensearch-welcome-msg.png" alt="architecture setup">
