# Metrics

Let's see how we can...

* add a new metric to be collected via Istio
* inspect the metric via Prometheus
* render the metric via Grafana
* explore the Kiali integration

...and understand the mechanics behind the example metric added in the chapter.

## Collecting a new metric

Using the YAML displayed below, we are going to add to Istio's engine a new metric collection definition:

~~~yaml
# Configuration for metric instances
apiVersion: config.istio.io/v1alpha2
kind: instance
metadata:
  name: doublerequestcount
  namespace: istio-system
spec:
  compiledTemplate: metric
  params:
    value: "2" # count each request twice
    dimensions:
      reporter: conditional((context.reporter.kind | "inbound") == "outbound", "client", "server")
      source: source.workload.name | "unknown"
      destination: destination.workload.name | "unknown"
      message: '"twice the fun!"'
    monitored_resource_type: '"UNSPECIFIED"'
---
# Configuration for a Prometheus handler
apiVersion: config.istio.io/v1alpha2
kind: handler
metadata:
  name: doublehandler
  namespace: istio-system
spec:
  compiledAdapter: prometheus
  params:
    metrics:
    - name: double_request_count # Prometheus metric name
      instance_name: doublerequestcount.instance.istio-system # Mixer instance name (fully-qualified)
      kind: COUNTER
      label_names:
      - reporter
      - source
      - destination
      - message
---
# Rule to send metric instances to a Prometheus handler
apiVersion: config.istio.io/v1alpha2
kind: rule
metadata:
  name: doubleprom
  namespace: istio-system
spec:
  actions:
  - handler: doublehandler
    instances: [ doublerequestcount ]
~~~

We will dive into the details later, first apply this metric definition via:

~~~bash
# Load definition so Istio will generate
# and collect automatically; objects are created
# in istio-system namespace
$ kubectl apply -f samples/bookinfo/telemetry/metrics.yaml

instance.config.istio.io/doublerequestcount created
handler.config.istio.io/doublehandler created
rule.config.istio.io/doubleprom created
~~~

Since we still have the traffic generator running (as explained in [Visualise mesh with Kiali --> generating a service graph](./visualize-mesh-with-kiali.md), we can directly jump into viewing the newly collected metrics!

## Prometheus dashboard

Let's access the dashboard - similar as what we did for [Kiali](./visualize-mesh-with-kiali.md) and [Jaeger](./distributed-tracing.md) we open another terminal and run a port-forwarding process:

~~~bash
# Keep this running inside a terminal
$ kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=prometheus -o jsonpath='{.items[0].metadata.name}') 9090:9090
~~~

Open your browser to the [Prometheus console](http://localhost:9090).

Next, in the input text box enter the expression:  `istio_double_request_count` and click execute.

The details pane is quite similar to:

~~~text
istio_double_request_count{destination="details-v1",instance="172.17.0.12:42422",job="istio-mesh",message="twice the fun!",reporter="client",source="productpage-v1"}   8
istio_double_request_count{destination="details-v1",instance="172.17.0.12:42422",job="istio-mesh",message="twice the fun!",reporter="server",source="productpage-v1"}   8
istio_double_request_count{destination="istio-policy",instance="172.17.0.12:42422",job="istio-mesh",message="twice the fun!",reporter="server",source="details-v1"}   4
istio_double_request_count{destination="istio-policy",instance="172.17.0.12:42422",job="istio-mesh",message="twice the fun!",reporter="server",source="istio-ingressgateway"}   4
~~~

## Understanding the metrics configuration

The added Istio metric collection configuration instructed Mixer to automatically generate and report a new metric for all traffic within the mesh.

The added configuration controlled three pieces of Mixer functionality:

1. Generation of instances (in this example, metric values) from Istio attributes
1. Creation of handlers (configured Mixer adapters) capable of processing generated instances
1. Dispatch of instances to handlers according to a set of rules

The metrics configuration directs Mixer to send metric values to Prometheus. It uses three stanzas (or blocks) of configuration:

1. instance
1. handler
1. rule

### Instance configuration

Please read the YML definition below:

~~~yaml
# Configuration for metric instances
apiVersion: config.istio.io/v1alpha2
kind: instance
metadata:
  name: doublerequestcount
  namespace: istio-system
spec:
  compiledTemplate: metric
  params:
    value: "2" # count each request twice
    dimensions:
      reporter: conditional((context.reporter.kind | "inbound") == "outbound", "client", "server")
      source: source.workload.name | "unknown"
      destination: destination.workload.name | "unknown"
      message: '"twice the fun!"'
    monitored_resource_type: '"UNSPECIFIED"'
~~~

The kind: `instance` stanza of configuration defines a schema for generated metric values (or instances) for a new metric named `doublerequestcount`. This instance configuration tells Mixer how to generate metric values for any given request, based on the attributes reported by Envoy (and generated by Mixer itself).

For each instance of `doublerequestcount`, the configuration directs Mixer to supply a `value of 2` for the instance. Because Istio generates an instance for each request, this means that this metric records a value equal to twice the total number of requests received.

A set of `dimensions` are specified for each `doublerequestcount` instance. Dimensions provide a way to slice, aggregate, and analyze metric data according to different needs and directions of inquiry. For instance, it may be desirable to only consider requests for a certain destination service when troubleshooting application behavior.

The configuration instructs Mixer to populate values for these dimensions based on attribute values and literal values. For instance, for the `source` dimension, the new configuration requests that the value be taken from the `source.workload.name` attribute. If that attribute value is not populated, the rule instructs Mixer to use a default value of `"unknown"`. For the message dimension, a literal value of `"twice the fun!"` will be used for all instances.

### Handler configuration

Please read the YML definition below:

~~~yaml
# Configuration for a Prometheus handler
apiVersion: config.istio.io/v1alpha2
kind: handler
metadata:
  name: doublehandler
  namespace: istio-system
spec:
  compiledAdapter: prometheus
  params:
    metrics:
    - name: double_request_count # Prometheus metric name
      instance_name: doublerequestcount.instance.istio-system # Mixer instance name (fully-qualified)
      kind: COUNTER
      label_names:
      - reporter
      - source
      - destination
      - message
~~~

The kind: `handler` stanza of configuration defines a handler named `doublehandler`. The handler `spec` configures how the Prometheus adapter code translates received metric instances into Prometheus-formatted values that can be processed by a Prometheus backend. This configuration specified a new Prometheus metric named `double_request_count`. The Prometheus adapter prepends the istio_ namespace to all metric names, therefore this metric will show up in Prometheus as `istio_double_request_count`. The metric has three labels matching the dimensions configured for `doublerequestcount` instances.

Mixer instances are matched to Prometheus metrics via the `instance_name` parameter. The `instance_name` values must be the fully-qualified name for Mixer instances (example: `doublerequestcount.instance.istio-system`).

### Rule configuration

Please read the YML definition below:

~~~yaml
# Rule to send metric instances to a Prometheus handler
apiVersion: config.istio.io/v1alpha2
kind: rule
metadata:
  name: doubleprom
  namespace: istio-system
spec:
  actions:
  - handler: doublehandler
    instances: [ doublerequestcount ]
~~~

The kind: `rule` stanza of configuration defines a new rule named `doubleprom`. The rule directs Mixer to send all `doublerequestcount` instances to the `doublehandler` handler. Because there is no match clause in the rule, and because the rule is in the configured default configuration namespace (`istio-system`), the rule is executed for all requests in the mesh.
