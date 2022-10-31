# Istio Egress Gateway

## Introduction

This repository contains a set of default configuration files to get an Istio installation running with the following components:

 - Egress Gateway
 - Istio Dashboard 
 - Grafana
 - Prometheus
 - Kiali and Kiali UI


## Installation

``` 
kubectl create ns istio-system

helm install istio-base <istio-directory>/manifests/charts/base --namespace istio-system

helm install istiod <istio-directory>/manifests/charts/istio-control/istio-discovery/ -n istio-system --values istio-values.yaml

helm install istio-ingress <istio-directory>/manifests/charts/gateways/istio-ingress/ -n istio-system --values istio-values.yaml

helm install istio-egress <istio-directory>/manifests/charts/gateways/istio-egress/ -n istio-system --values istio-values

kubectl apply -f prometheus-service.yaml

helm install kiali-server kiali-server -n istio-system --values kiali-values.yaml --set auth.strategy=anonymous --set external_services.prometheus.url=http://aspen-mesh-metrics-collector.istio-system.svc.cluster.local:19090 --repo https://kiali.org/helm-charts

kubectl apply -f kiali-service.yaml

kubectl apply -f dashboard-service.yaml

kubectl create ns grafana

helm repo add grafana https://grafana.github.io/helm-charts && helm repo update && helm install grafana grafana/grafana -n grafana --values grafana-values.yaml

kubectl get secret --namespace grafana grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

kubectl label --overwrite namespace default istio-injection=enabled
```

## Egress gateway

There is a separate folder called [`ecommerce-microservices`](https://github.com/istio/se-workshop-demo-library/ecommerce-microservices) that contains a demonstration of microservices on Istio.  One of the microservices, `payment-service`, requires an egress gateway to make an API call to Stripe, a credit card processing API.  There are two files that will create the egress gateway for you:

```
kubectl apply -f stripe-tls-service-entry.yaml

kubectl apply -f stripe-tls-egress-gateway.yaml
```

To verify that the external traffic is going through the egress gateway, submit a couple of orders, and then look at the access logs for the egress gateway:

```
kubectl logs -l istio=egressgateway -n istio-system
```

You should see entries for https calls to api.stripe.com, which means that all external calls are being routed through the egress gateway.

