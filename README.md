# Kubernetes Gateway API tutorial

Learn basic concepts and how to use the Kubernetes Gateway API.

## Goal

Setup a maintenance page only specific users may pass during.

## Prerequisites

* kubectl
* kind
  * kubernetes cluster v1.25.x

---

## Prepare a k8s cluster with kind

```shell
kind create cluster --name gateway-example
kind get kubeconfig --name gateway-example
kubectl config set-context kind-gateway-example
```

## Install Gateway API custom resource definitions (CRD)

Based on: `01-standard-install.yaml`

```shell
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml
```

The result should look like this:

```shell
kubectl get crd | grep "gateway.networking"

NAME                                        CREATED AT
gatewayclasses.gateway.networking.k8s.io    2023-12-16T10:35:26Z
gateways.gateway.networking.k8s.io          2023-12-16T10:35:26Z
httproutes.gateway.networking.k8s.io        2023-12-16T10:35:26Z
referencegrants.gateway.networking.k8s.io   2023-12-16T10:35:27Z
```

## Install a gateway controller of your choice

### NGINX Gateway Fabric

#### First install its specific NGINX Gateway Fabric CRDs

Based on: `02-crds.yaml`

```shell
kubectl apply -f https://github.com/nginxinc/nginx-gateway-fabric/releases/download/v1.1.0/crds.yaml
```

The result should look like this:

```shell
kubectl get crd  | grep "nginx"

nginxgateways.gateway.nginx.org             2023-12-16T10:42:24Z
```

#### Install NGINX Gateway Fabric

Note: This creates the namespace `nginx-gateway` and deploy "ngf" in it.

Based on: `03-nginx-gateway.yaml`

```shell
kubectl apply -f https://github.com/nginxinc/nginx-gateway-fabric/releases/download/v1.1.0/nginx-gateway.yaml
```

Now we are able to work with instances of the new nginx specific custom resource definitions.
The result should look like this:

```shell
kubectl -n nginx-gateway get nginxgateways.gateway.nginx.org 

NAME                   AGE
nginx-gateway-config   82s
```

#### Expose the NGINX Gateway Fabric

##### Option 1 - as a NodePort service

We chose this option since we use a local kubernetes cluster provided through kind we do not yet have a loadbalancer such as metallb in place.

Based on: `04-nodeport.yaml`

```shell
kubectl apply -f https://raw.githubusercontent.com/nginxinc/nginx-gateway-fabric/v1.0.0/deploy/manifests/service/nodeport.yaml
```

The result should look like this:

```shell
kubectl -n nginx-gateway get svc

NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
nginx-gateway   NodePort   10.96.177.241   <none>        80:32310/TCP,443:30606/TCP   4s
```

##### Option 2 - as a Loadbalancer Service

Coming soon...

---

## Deploy some workload

This will create two pods (a and b) and 2 services (a and b) in the `default` namespace.

```shell
kubectl apply -f 05-workload.yaml
```

## Publish the workload

The following resources will be applied to the `default` namespace.

### Create a gateway

Based on: `06-gateway.yaml`

```shell
kubectl apply -f 06-gateway.yaml
```

The result looks like this:

```shell
kubectl -n default get gateway

NAME               CLASS   ADDRESS   PROGRAMMED   AGE
combined-gateway   nginx             True         8s
```

### Create a ruleset for our workloads

Based on: `07-httproute.yaml`

Still the missing link is to create a ruleset - an `httpRoute` - to publish the newly created workload pods which makes them available through the gateway controller.
The `httpRoute` can be roughly compared to the former resource `kind: ingress`.

So we make use of the new CRD `httproutes.gateway.networking.k8s.io` and create a k8s resource based on it. 

```shell
kubectl apply -f 07-httproute.yaml
```

The result looks like this:

```shell
kubectl -n default get httproutes.gateway.networking.k8s.io
 
NAME        HOSTNAMES       AGE
service-a   ["localhost"]   5s
service-b   ["localhost"]   5s
```

---

## Verify

* Create a port-forward to the NodePort service of the gateway controller:
  ```shell
  kubectl -n nginx-gateway port-forward svc/nginx-gateway 8080:80
  ```

* Visit these URLs
  * http://localhost:8080/service-a
  * http://localhost:8080/service-b

---

# Info, Links, Reference, Credits

Inspired by this blogpost https://benchkram.de/blog/dev/replace-kubernetes-ingress-with-gateway-api

https://github.com/kubernetes-sigs/gateway-api

https://gateway-api.sigs.k8s.io/guides/#installing-gateway-api

https://docs.nginx.com/nginx-gateway-fabric/

https://docs.nginx.com/nginx-gateway-fabric/installation/installing-ngf/manifests/

https://github.com/nginxinc/nginx-gateway-fabric/blob/main/README.md#technical-specifications

https://github.com/nginxinc/nginx-gateway-fabric/tree/v1.1.0/deploy/manifests

#### in question

- only deployment is currently supported - waiting for ds
- how to create HA
