# Proof of Concept: Istio

Notes, ideas, scripts during Istio evaluation, including playing around with the setup of Istio.

## Why use Istio

Istio makes it easy to create a network of deployed services with load balancing, service-to-service authentication, monitoring, and more, with few (Zipkin related...) or no code changes in service code. You add Istio support to services by deploying a special sidecar proxy throughout your environment that intercepts all network communication between microservices, then configure and manage Istio using its control plane functionality, which includes:

* Automatic load balancing for HTTP, gRPC, WebSocket, and TCP traffic.
* Fine-grained control of traffic behavior with rich routing rules, retries, failovers, and fault injection.
* A pluggable policy layer and configuration API supporting access controls, rate limits and quotas.
* Automatic metrics, logs, and traces for all traffic within a cluster, including cluster ingress and egress.
* Secure service-to-service communication in a cluster with strong identity-based authentication and authorization.

Istio is designed for extensibility and meets diverse deployment needs.

## Reading

Hit the link below and you get a good overview - enough to get you to understand the BookIn with the 

* [What is Istio?](https://istio.io/docs/concepts/what-is-istio/)

Skimmed through the other docs, more of reference material then required to get started, will read later.

## Explore Istio

Here I follow 3 tracks:

1. Follow the [user guide of Istio using BookInfo example](./istio-kubernetes-userguide.md) based upon Kubernetes using Docker for Desktop
1. Reading through the book [Introducing Istio Mesh for microservices](https://www.goodreads.com/book/show/40361132-introducing-istio-service-mesh-for-microservices) while [playing with Istio on OpenShift](./minishift-istio-install.md)
1. Figure out how we can [map Instio on CBX microservices](./cbx-on-istio.md)



