# Istio demo app: BookInfo

My notes deploying this demo app and assessing Istio capabilities using the [Bookinfo example](https://istio.io/docs/examples/bookinfo/) - read through this first 2 chapters so you understand what the application is about.

## Prepare you k8s namespace

We are going to deploy the bookinfo application into its own namespace...

~~~bash
# List namespaces
$ kubectl get namespace

NAME              STATUS   AGE
default           Active   20m
docker            Active   19m
istio-system      Active   2m42s
kube-node-lease   Active   20m
kube-public       Active   20m
kube-system       Active   20m

# Create namespace for demo app
$ kubectl create namespace bookinfo
~~~

## Deploy the demo app

Change directory to the root of the Istio installation.

~~~bash
# The default Istio installation uses automatic sidecar injection.
# Label the namespace that will host the application with istio-injection=enabled:
$ kubectl label namespace bookinfo istio-injection=enabled

# Deploy the demo application using the kubectl command:
$ kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml \
   -n bookinfo

# Confirm all services are correctly defined and running:
$ kubectl get services -n bookinfo

NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
details       ClusterIP   10.102.130.187   <none>        9080/TCP   43s
kubernetes    ClusterIP   10.96.0.1        <none>        443/TCP    3d
productpage   ClusterIP   10.97.28.9       <none>        9080/TCP   43s
ratings       ClusterIP   10.105.45.86     <none>        9080/TCP   43s
reviews       ClusterIP   10.101.117.31    <none>        9080/TCP   43s

# Confirm all pods are running:
$ kubectl get pods -n bookinfo

NAME                              READY   STATUS    RESTARTS   AGE
details-v1-5544dc4896-wwfkh       2/2     Running   0          2m20s
productpage-v1-7868c48878-l529f   2/2     Running   0          2m20s
ratings-v1-858fb7569b-7bpv2       2/2     Running   0          2m20s
reviews-v1-796d4c54d7-5h47n       2/2     Running   0          2m20s
reviews-v2-5d5d57db85-jqn4n       2/2     Running   0          2m20s
reviews-v3-77c6b4bdff-p8ckn       2/2     Running   0          2m20s
~~~

## Confirm running demo app

To confirm that the Bookinfo application is running, send a request to it by a `curl` command from some pod, for example from `ratings`:

~~~bash
# Query product page from the ratings pod:
$ kubectl exec -it $(kubectl get pod -l app=ratings -n bookinfo -o \
   jsonpath='{.items[0].metadata.name}') \
   -c ratings -n bookinfo -- curl productpage:9080/productpage | \
   grep -o "<title>.*</title>"

<title>Simple Bookstore App</title>
~~~

## Determining the ingress IP and port

Now that the Bookinfo services are up and running, you need to make the application accessible from outside of your Kubernetes cluster, e.g., from a browser. An Istio Gateway is used for this purpose.

~~~bash
# Define the ingress gateway for the application:
$ kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml \
   -n bookinfo

# Confirm the gateway has been created:
$ kubectl get gateway -n bookinfo

NAME               AGE
bookinfo-gateway   21s
~~~

The Docker-for-Desktop environment has an external load balancer, hence we follow the following steps to obtain the ingress IP and ports:

~~~bash
# Set the ingress IP and ports:
$ export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway \
   -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

$ export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway \
   -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')

$ export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway \
   -o jsonpath='{.spec.ports[?(@.name=="https")].port}')

# Check the variables (all should have a value)
$ echo $INGRESS_HOST
localhost

$ echo $INGRESS_PORT
80

$ echo $SECURE_INGRESS_PORT
443

# Next, Set GATEWAY_URL:
$ export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT && echo $GATEWAY_URL
localhost:80
~~~

To confirm that the Bookinfo application is accessible from outside the cluster (hence via your MacBook's localhost interface...), run the following curl command:

~~~bash
# Get now the page via the Istio gateway
$ curl -s http://$GATEWAY_URL/productpage | grep -o "<title>.*</title>"
~~~

You can also point your browser to [http://$GATEWAY_URL/productpage](http://localhost:80/productpage) to view the Bookinfo web page. If you refresh the page several times, you should see different versions of reviews shown in `productpage`, presented in a round robin style (red stars, black stars, no stars), since we haven’t yet used Istio to control the version routing.

## Apply default destination rules

Before you can use Istio to control the Bookinfo version routing, you need to define the available versions, called subsets, in [destination rules](https://istio.io/docs/concepts/traffic-management/#destination-rules).

Run the following command to create default destination rules for the Bookinfo services:

~~~bash
# We are using mutual TLS, hence...
$ kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml \
  -n bookinfo
~~~

Wait a few seconds for the destination rules to propagate.

You can display the destination rules with the following command:

~~~bash
# Show rules
$ kubectl get destinationrules -o yaml \
   -n bookinfo
~~~

## Let's explore

Now we have setup basic rules for Istio’s features for traffic routing, fault injection, rate limiting, etc. let's see what we can actually do...

First, let's tackle [telemetry](https://en.wikipedia.org/wiki/Telemetry) to "see" what we pretty much get out of the box:

* [How to visualize your mesh](./visualize-mesh-with-kiali.md) with [Kiali](https://www.kiali.io)
* [Perform distributed tracing](./distributed-tracing.md)
* [Dig up some metrics](./metrics.md)
* [Inspect the logs](./inspect-logs.md)

Next, let's dive into traffic management, security and policies...

* a
* b
* c

## Cleanup

When you’re finished experimenting with the Bookinfo sample, uninstall and clean it up using the following instructions corresponding to your Istio runtime environment.

~~~bash
# Delete the routing rules and terminate the application pods
$ ./samples/bookinfo/platform/kube/cleanup.sh  #-- provide the bookinfo space name!

# Confirm shutdown
$ kubectl get virtualservices    #-- there should be no virtual services
$ kubectl get destinationrules   #-- there should be no destination rules
$ kubectl get gateway            #-- there should be no gateway
$ kubectl get pods               #-- the Bookinfo pods should be deleted
~~~
