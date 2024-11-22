
## Instrumenting Nginx with the OpenTelemetry operator

https://github.com/open-telemetry/opentelemetry-operator

### Install OpenTelemetry Operator using Helm

Add the repository

~~~
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
~~~

Create OpenTelemetry namespace

~~~
kubectl create namespace opentelemetry
~~~

Install the Helm chart

~~~
helm install --namespace opentelemetry opentelemetry-operator open-telemetry/opentelemetry-operator --values=nginx/auto/helm/values.yaml
~~~

The values.yaml file is modified to enable Nginx instrumentation via a feature flag, which is still considered as Alpha stage by the OpenTelemetry project

At this point, there should be a deployment *opentelemetry-operator* in the namespace *opentelemetry* with associated pod like *opentelemetry-operator-785bbd55fd-gff8f*.

### Start the OpenTelemetry collector

OpenTelemetry collector can be started standalone, but the easier method is using OpenTelemetry Operator to handle the collectors lifecycle. In the directory `nginx/auto/instrument` is a file `collector.yaml` with an example how to run a collector with all the features we need, i.e. getting metrics out of Nginx traces and sending them to AppDynamics event service (Analytics).

There are following parameters in the `collector.yaml` file that need to be changed based on the target AppDynamiccs controller environment:

* <CONTROLLER-ACCOUNT> - typically something like *company* in front of .saas.appdynamics.com
* <CONTROLLER-HOSTNAME> - typically something like *company.saas.appdynamics.com*
* <CONTROLLER-OTEL-ENDPOINT> - something like https://fra-sls-agent-api.saas.appdynamics.com, the correct URL for the target controller can be found on the OTel tab under Configuration submenu and then under Exporters tab on the *Configure OpenTelemetry collector for AppDynamics* screen.
* <CONTROLLER-OTEL-API-KEY> - on the same screen as above, on the bottom, there's a *Access Key* section. Click the *Show* button to get the access key value
* <CONTROLLER-EVENT-SERVICE-URL> - something like https://fra-ana-api.saas.appdynamics.com, the correct value depends on the AWS hosting site of the controller. The URL for the target controller can be found in the documentation at [Saas IP ranges](https://docs.appdynamics.com/appd/24.x/24.11/en/cisco-appdynamics-essentials/getting-started/saas-domains-and-ip-ranges), look for *Analytics*
* <CONTROLLER-GLOBAL-ACCOUNT-NAME> - this can be found in the controller under Settings (now under user login icon) -> Admin -> License - tab Account (on the top line), field *Global Account Name*
* <CONTROLLER-EVENT-SERVICE-API-KEY> - this needs to be likely created in the target controller. Go to Analytics -> Configuration -> API Keys tab. Click *Add* button, give the key a name and optionally description. Then assign appropriate access rights to the key. Select *Custom Analytics Events Permissions* and select all three boxes: *Can Manage Schema*, *Can Query all Custom Analytics Events*, *Can Publish all Custom Analytics Events*. Once you click the *Create* button, API Key will be displayed. This is the value you need. Before closing the window with the API key displayed, make sure you have copied the value. It will not be accessible again, new key would have to be created.

At this stage, time interval buckets can be also modified if needed.

With all the changes done to `collector.yaml`, OpenTelemetry collector can be started by a command in an appropriate namespace: 

~~~
kubectl -nnamespace apply -f nginx/auto/instrument/collector.yaml
~~~

Namespace might be any namespace, usually in the opentelemetry namespace or in the namespace of the application. The collector is published via a service and the URL of the service will be part of the configuration of the Instrumentation as described below in the guide. 

The collector used is a custom build containing all the necessary components built in. Source code can be made available upon request.

Verify the collector is running - in the appropriate namespace, there should be a deployment *appdynamics-custom-collector* and related pod like  *appdynamics-custom-collector-68579d5b58-gt26w*.

### Instrumenting Nginx

To instrument Nginx, there are two things to do:

1) Create *Instrumentation* resource with the instrumentation rules
2) Annotate the Nginx pod by a reference to that *Instrumentation* resource

#### Creating the Instrumentation resource

Sample Instrumentation resource is located at `nginx/auto/instrument/instrumentation.yaml`. There are a few items needing adjustment:

* *metadata.name* - this is the name of the object, which will later be referred to in the Nginx pod annotation
* *spec.exporter.endpoint* - this is the default URL of the OpenTelemetry collector Kubernetes service deployed earlier. If the collector runs in the same namenspace as the monitored Nginx, it can use the simple service names as in the example. If it runs in a different namespace, then the FQDN should have format <otel-collector-service-name>.<namespace-of-the-collector>.svc.cluster.local
* *spec.nginx.env* with name *OTEL_EXPORTER_OTLP_ENDPOINT* should use the same service URL as above, just the port 4318
* *spec.nginx.attrs* with name *NginxModuleServiceNamespace* should be set to the application name as it should appear in AppDynamics. Tier name will be set to the deployment name of the Nginx automatically and can be overridden by setting attribute *NginxModuleServiceName*.

Create the resource by running:

~~~
kubectl -n<namespace-of-the-Nginx> apply -f nginx/auto/instrument/instrumentation.yaml
~~~

#### Instrumenting Nginx

To instrument the Nginx, Nginx pod must be annotated as follows:

~~~~
instrumentation.opentelemetry.io/inject-nginx: "nginx1-instrumentation"
~~~~

where "nginx1-instrumentation" is the name of Instrumentaton CRD created in the previous chapter.

Example can be found at `nginx/auto/instrument/d-svc-nginx.yaml`. Note that the annotation is on the pod template spec level, not on the deployment level. 

Once a new Nginx pod is started via 

~~~
kubectl -n<namespace-of-the-Nginx> apply -f nginx/auto/instrument/d-svc-nginx.yaml
~~~

the pod should be automatically instrumented by OpenTelemetry agent and once requests are coming to it, metrics data should, with up to a minute delay, appear in AppDynamics.

#### Delete OpenTelemetry Operator (if needed)

~~~
helm delete --namespace=opentelemetry  opentelemetry-operator
~~~

## Working with the data in AppDynamics

In AppDynamics, metrics collected will appear in the Analytics event database. It is also possible to store them in the standard metric tree, this method is not described here at this time.

The data is stored in the analytics event database table defined in the configuration of the *appdynamics* exporter in the OpenTelemetry collector configuration. Currently, it is set to *mdotelmetrics*, but it can be changed - look for that string in the `collector.yaml` file.

There are two kinds of metrics produced by the collector:

1) Request counts, average response times, and error counts for each location given by the Nginx configuration
2) Histogram of response times for each location given by the Nginx configuration

Access to the data is via ADQL queries and their visual representations. 

#### Getting reponse times and request counts

To access the request counts, error counts, and response times, use query like:

`SELECT series(metricTimestamp, '1m') AS "Time", lbl_span_name AS "Location", avg(metricValue) AS "ResponseTime (ms)" FROM mdotelmetrics WHERE scp_name = "spanmetricsconnector" and metricName = "traces.span.metrics.avgDuration" and lbl_span_name LIKE "/api/*"`

field *metricName* can be:

* *traces.span.metrics.avgDuration* to get response times
* *traces.span.metrics.calls* to get request counts, both ok and in error. They can be further split by error code via value of the field *lbl_status_code*, sample values are *STATUS_CODE_OK* and *STATUS_CODE_UNSET*
  
The field *lbl_span_name* corresponds to the name of the location directive in the Nginx configuration

#### Working with histograms

To get histogram of response times over a time period, use ADQL query like:

`SELECT bucket AS "Response Time (ms)", lbl_span_name AS "Service", sum(metricValue) AS "Number of Requests" FROM mdotelmetrics WHERE scp_name = "spanmetricsconnector" and metricName = "traces.span.metrics.duration" and lbl_span_name LIKE "/*"` 

and choose vertical bar chart visualisation


