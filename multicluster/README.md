# Istio Multicluster

## Introduction

This repository contains configuration files and instructions on configuring a multi-primary multicluster Istio environment.

## Installation

Cluster 1:

``` 
kubectl create ns istio-system

kubectl label ns istio-system topology.istio.io/network=network1

kubectl create secret generic cacerts -n istio-system --from-file=certs/cluster1/ca-cert.pem --from-file=certs/cluster1/ca-key.pem --from-file=certs/cluster1/root-cert.pem --from-file=certs/cluster1/cert-chain.pem 

helm install istio-base <istio-directory>/manifests/charts/base --namespace istio-system

helm install istiod <istio-directory>/manifests/charts/istio-control/istio-discovery/ -n istio-system --values istio-values-cluster1.yaml

helm install istio-ingress <istio-directory>/manifests/charts/gateways/istio-ingress/ -n istio-system --values istio-values-cluster1.yaml

helm install istio-egress <istio-directory>/manifests/charts/gateways/istio-egress/ -n istio-system --values istio-values-cluster1.yaml

istioctl x create-remote-secret --name=cluster1 >> secret-cluster1.yaml

istioctl x create-remote-secret --name=cluster2 >> secret-cluster2.yaml

kubectl apply -f secret-cluster2.yaml

kubectl apply -f expose-services.yaml
```

Cluster 2:

```
kubectl create ns istio-system

kubectl label ns istio-system topology.istio.io/network=network1

kubectl create secret generic cacerts -n istio-system --from-file=certs/cluster2/ca-cert.pem --from-file=certs/cluster2/ca-key.pem --from-file=certs/cluster2/root-cert.pem --from-file=certs/cluster2/cert-chain.pem 

helm install istio-base <istio-directory>/manifests/charts/base --namespace istio-system

helm install istiod <istio-directory>/manifests/charts/istio-control/istio-discovery/ -n istio-system --values istio-values-cluster2.yaml

helm install istio-ingress <istio-directory>/manifests/charts/gateways/istio-ingress/ -n istio-system --values istio-values-cluster2.yaml

helm install istio-egress <istio-directory>/manifests/charts/gateways/istio-egress/ -n istio-system --values istio-values-cluster2.yaml

kubectl apply -f secret-cluster1.yaml

kubectl apply -f expose-services.yaml
```

## Verify installation

Cluster 1:

```
kubectl create ns sample 

kubectl label ns sample istio-injection=enabled

kubectl apply -l service=helloworld -n sample -f <istio-directory>/samples/helloworld/helloworld.yaml

kubectl apply -n sample -l version=v1 -f /Users/jimerson/istio/istio-1.9.5-am1/samples/helloworld/helloworld.yaml 

```

Cluster 2:

```
kubectl create ns sample 

kubectl label ns sample istio-injection=enabled

kubectl apply -l service=helloworld -n sample -f <istio-directory>/samples/helloworld/helloworld.yaml

```
