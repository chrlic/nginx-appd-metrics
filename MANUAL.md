# Manual instrumentation of NGINX

This document describes, how to instrument Nginx for OpenTelemetry and how to setup OpenTelemetry collector to send metrics data to AppDynamics.

## Deploying the OpenTelemetry collector

OpenTelemetry collector must be deployed first.

There are three manifests involved:

* config map describing the OpenTelemetry collector's configuration
* deployment describing the OpenTelemetry collector workload
* service exposing the collector within the cluster, in our case, to Nginx. Multiple Nginx(s) can share single collector

example of both is in the file `nginx/manual/collector/d-svc-collector.yaml`

### Setting up the config map with collector's configuration

There are following placeholders, which need to be replaced by a specific value:
* `<CONTROLLER-EVENT-SERVICE-URL>` - something like https://fra-ana-api.saas.appdynamics.com, the correct value depends on the AWS hosting site of the controller. The URL for the target controller can be found in the documentation at [Saas IP ranges](https://docs.appdynamics.com/appd/24.x/24.11/en/cisco-appdynamics-essentials/getting-started/saas-domains-and-ip-ranges), look for *Analytics*
* `<CONTROLLER-GLOBAL-ACCOUNT-NAME>` - this can be found in the controller under Settings (now under user login icon) -> Admin -> License - tab Account (on the top line), field *Global Account Name*
* `<CONTROLLER-EVENT-SERVICE-API-KEY>` - this needs to be likely created in the target controller. Go to Analytics -> Configuration -> API Keys tab. Click *Add* button, give the key a name and optionally description. Then assign appropriate access rights to the key. Select *Custom Analytics Events Permissions* and select all three boxes: *Can Manage Schema*, *Can Query all Custom Analytics Events*, *Can Publish all Custom Analytics Events*. Once you click the *Create* button, API Key will be displayed. This is the value you need. Before closing the window with the API key displayed, make sure you have copied the value. It will not be accessible again, new key would have to be created.

### Setting up the deployment and service

Adjust the naming if necessary, other that that, no changes are needed

### Deploying the collector

Deploy the OpenTelemetry collector to the cluster by running 

`kubectl -n <namespace> apply -f nginx/manual/collector/d-svc-collector.yaml`

## Instrument Nginx manually

In the directory `nginx/manual/nginx`, there are three example files showing the instrumentation of the Nginx:

* `cm-nginx-config.conf` - the config map containing example of the nginx.conf Nginx configuration file
* `cm-otel-config.conf` - the config map containing the opel.conf configuration file setting up parameters for the OpenTelemetry libraries
* `d-svc-nginx.yaml` - the example file with the instrumented Nginx deployment and service

### Modifying the `nginx.conf` in the config map (`cm-nginx-config.conf` as an example)

What needs to be done:

* load instrumentation libraries into Nginx process
* load configuration file with OpenTelemetry-related parameters from a `otel.conf` configuration file

The existing `nginx.conf` file needs to be modified in this way:

1) Insert this line `load_module /opt/opentelemetry-webserver/agent/WebServerModule/Nginx/1.24.0/ngx_http_opentelemetry_module.so;` as the first line in the config. See example in the `cm-nginx-config.conf`, the section is marked by the comment. **THE VERSION (1.24.0) IN THE PATH MUST CORRESPOND TO THE NGINX VERSION**
2) Insert this line `include /opt/opentelemetry-webserver/agent/nginx-config/otel.conf;` in the *http* directive as also shown and commented in the example file.

### Modifying the Nginx deployment or stateful set (`d-svc-nginx.yaml` as an example)

This is the most difficult part of the whole instrumentation process. What is being done:

* *initContainer* is added with the image containing the OpenTelemetry libraries for Nginx. In the example image `ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-apache-httpd:1.0.4` is used which contains support for Nginx versions 1.24.0 and 1.25.3. This is currently the latest image available.
* *volumes* `otel-conf` and `otel-module` are added. The volume `otel-conf` contains the content of the config map defined in the file `cm-otel-config.yaml`. This config map does not have to be changed in any way you there is not a need to adjust some of the parameters (it generally is not). The volume `otel-module` is an empty directory volume, which will serve as a volume carrying the Nginx loadable module and the `otel.conf` file into the Nginx pod.
* *volumeMount* `otel-module` is added to the Nginx *container* in the *pod template* of the original manifest, which maps the OpenTelemetry modules and configuration to the Nginx pod's filesystem. 
* add a directory with OpenTelemetry modules to the loader path via setting up a environmental variable LD_LIBRARY_PATH to the Nginx *container* in the *pod template* of the original manifest. Without this, Nginx process would not be able to load them.

How to do that:
1) Add the *initContainers* section from the example file `d-svc-nginx.yaml` to the original Nginx manifest. Simply copy paste this section from the example file. Only change the *env* variable setting for `K8S_DEPLOYMENT_NAME` so, that it reflects the deployment/stateful set name. 
   
```yaml
        env:
        - name: K8S_DEPLOYMENT_NAME
          value: nginx
          # ^^^^^ this should correspond to deployment name ^^^^^
```

2) Add the two *volumes* `otel-conf` and `otel-module` to the *volumes* section. There's very likely already one volume containing the `nginx.conf` configuration (which by this time should contain the two added lines as described above).

```yaml
      - name: otel-conf
        configMap:
          name: otel-conf
          items:
            - key: otel.conf
              path: otel.conf
      - name: otel-module
        emptyDir: {}
```

3) Add *volumeMount* to the Nginx *container* section

```yaml
        volumeMounts:
          - name: otel-module
            mountPath: /opt/opentelemetry-webserver
            readOnly: false
```

4) Add the environmental variable LD_LIBRARY_PATH to the Nginx *container* section

```yaml
        env:
          - name: LD_LIBRARY_PATH
            value: /opt/opentelemetry-webserver/agent/sdk_lib/lib
```

### Deploying the instrumented Nginx

Deploy the manifests to the kubernetes (use your manifest names):

```yaml
kubectl -n <namespace> apply -f cm-nginx-config.yaml
kubectl -n <namespace> apply -f cm-otel-config.yaml
kubectl -n <namespace> apply -f d-svc-nginx.yaml
```

## Test the setup!

run some transactions to Nginx and see section **Working with the data in AppDynamics** in `README.md` how to work with the resulting data.