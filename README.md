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

* [What is Istio?](https://istio.io/docs/concepts/what-is-istio/)

Skimmed through the other docs, more of reference material then required to get started, will read later.

## Prerequisites

Make sure on your laptop the following ports *__are not in use__* on localhost -- they will conflict with Istio `ingressgateway` which is exposed to the Dockerhost localhost interface:

* `80/TCP`
* `443/TCP`
* `15020/TCP`
* `15029/TCP`
* `15030/TCP`
* `15031/TCP`
* `15032/TCP`
* `15443/TCP`
* `31400/TCP`

## Setting up local k8s cluster

* Reset Docker for Desktop to factory defaults (or use `docker system prune`)
* Followed the tuning steps described to setup Istio on Kubernets for the [Docker Desktop](https://istio.io/docs/setup/kubernetes/platform-setup/docker/) platform.
* Enabled Kubernetes
* Installed Kubernetes cli via `brew install kubectl`
* Select the Docker for Desktop k8s context via `kubectl config use-context docker-for-desktop`

## Download

~~~bash
# Download release
$ curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.2.2 sh -

# Move to the Istio package directory.
$ cd istio-1.2.2
~~~

The installation directory contains:

* Installation YAML files for Kubernetes in `install/kubernetes`
* Sample applications in `samples/`
* The `istioctl` client binary in the `bin/` directory. `istioctl` is used when manually injecting Envoy as a sidecar proxy.

~~~bash
# Add the istioctl client to your PATH
$ export PATH=$PWD/bin:$PATH
~~~

### Enabling auto-completion (zsh)

The `istioctl` auto-completion file is located in the `tools/` directory.

~~~bash
# Enable auto-complete
$ source tools/_istioctl
~~~

## Install

Install all the Istio [Custom Resource Definitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#customresourcedefinitions) (CRDs) using `kubectl apply`, and wait a few seconds for the CRDs to be committed in the Kubernetes API-server:

~~~bash
# Kick off the command from the Istio directory...
$ for i in install/kubernetes/helm/istio-init/files/crd*yaml; do
   kubectl apply -f $i
done
~~~

Next, deploy the strict mutual TLS demo profile; this variant will enforce mutual TLS authentication between all clients and servers.

Since we use a fresh Kubernetes cluster where all workloads will be Istio-enabled we can use this option. All newly deployed workloads will have Istio sidecars installed - and that is what we want!

~~~bash
# Kick off the command from the Istio directory...
$ kubectl apply -f install/kubernetes/istio-demo-auth.yaml
~~~

## Verify your installation

Ensure the following Kubernetes services are deployed and verify they all have an appropriate `CLUSTER-IP` except the `jaeger-agent` service:

~~~bash
$ kubectl get svc -n istio-system

NAME                     TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                                                                                      AGE
grafana                  ClusterIP      10.100.172.192   <none>        3000/TCP                                                                                                                                     5m
istio-citadel            ClusterIP      10.104.4.119     <none>        8060/TCP,15014/TCP                                                                                                                           5m
istio-egressgateway      ClusterIP      10.97.185.113    <none>        80/TCP,443/TCP,15443/TCP                                                                                                                     5m
istio-galley             ClusterIP      10.99.219.55     <none>        443/TCP,15014/TCP,9901/TCP                                                                                                                   5m
istio-ingressgateway     LoadBalancer   10.107.252.27    localhost     15020:32352/TCP,80:31380/TCP,443:31390/TCP,31400:31400/TCP,15029:32456/TCP,15030:32334/TCP,15031:32023/TCP,15032:31309/TCP,15443:30845/TCP   5m
istio-pilot              ClusterIP      10.97.2.48       <none>        15010/TCP,15011/TCP,8080/TCP,15014/TCP                                                                                                       5m
istio-policy             ClusterIP      10.98.112.115    <none>        9091/TCP,15004/TCP,15014/TCP                                                                                                                 5m
istio-sidecar-injector   ClusterIP      10.96.231.10     <none>        443/TCP                                                                                                                                      5m
istio-telemetry          ClusterIP      10.104.184.141   <none>        9091/TCP,15004/TCP,15014/TCP,42422/TCP                                                                                                       5m
jaeger-agent             ClusterIP      None             <none>        5775/UDP,6831/UDP,6832/UDP                                                                                                                   5m
jaeger-collector         ClusterIP      10.109.205.133   <none>        14267/TCP,14268/TCP                                                                                                                          5m
jaeger-query             ClusterIP      10.101.244.136   <none>        16686/TCP                                                                                                                                    5m
kiali                    ClusterIP      10.106.42.5      <none>        20001/TCP                                                                                                                                    5m
prometheus               ClusterIP      10.104.18.191    <none>        9090/TCP                                                                                                                                     5m
tracing                  ClusterIP      10.108.157.38    <none>        80/TCP                                                                                                                                       5m
zipkin                   ClusterIP      10.100.176.134   <none>        9411/TCP
~~~

Ensure corresponding Kubernetes pods are deployed and have a `STATUS` of `Running`:

~~~bash
$ kubectl get pods -n istio-system

NAME                                      READY     STATUS      RESTARTS   AGE
grafana-55998bf4dd-2qlmn                  1/1       Running     0          8m
istio-citadel-69fc5c9c8f-gqztd            1/1       Running     0          8m
istio-cleanup-secrets-1.2.2-v4vdh         0/1       Completed   0          8m
istio-egressgateway-57ff8c9c8d-4b5xv      1/1       Running     0          8m
istio-galley-c97f568fc-w8jdj              1/1       Running     0          8m
istio-grafana-post-install-1.2.2-pjlm2    0/1       Completed   0          8m
istio-ingressgateway-78ccbccb64-gjj5r     1/1       Running     0          8m
istio-pilot-57cb75cd-6vvgp                2/2       Running     0          8m
istio-policy-869fdf9ddc-f5bss             2/2       Running     6          8m
istio-security-post-install-1.2.2-2pzf4   0/1       Completed   0          8m
istio-sidecar-injector-5b6dd98c9c-rg76k   1/1       Running     0          8m
istio-telemetry-855f87cf8d-wqzrw          2/2       Running     4          8m
istio-tracing-79c7955c98-hmshb            1/1       Running     0          8m
kiali-7ff7b959f-mv82m                     1/1       Running     0          8m
prometheus-69bddf498b-4xvb7               1/1       Running     0          8m
~~~

## Deploy our applications

This is where we...

* Explore Istio via the demo app [BookInfo](./istio-bookinfo.md)
* Figure out how we can [map Instio on CBX microservices](./cbx-on-istio.md).

## Uninstall

The uninstall deletes the RBAC permissions, the istio-system namespace, and all resources hierarchically under it. It is safe to ignore errors for non-existent resources because they may have been deleted hierarchically.

Uninstall the demo profile corresponding to the mutual TLS mode you enabled:

~~~bash
# Delete demo setup Istio infra
$ kubectl delete -f install/kubernetes/istio-demo-auth.yaml
~~~

If desired, delete the Istio CRDs:

~~~bash
# Delete CRDs
$ for i in install/kubernetes/helm/istio-init/files/crd*yaml; do
   kubectl delete -f $i;
done
~~~
