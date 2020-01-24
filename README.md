# Prereqs - Setting up Helm 3 and the Datadog Agent

## install helm 3

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

## deploy datadog

```
helm install datadog-agent --set datadog.apiKey=<DATADOG_API_KEY> stable/datadog
```

# METHOD 1 - Deploy the jmx version of the Datadog Agent and enable the jmx integration using K8s annotations

### Note: These metrics will be prefixed with `jvm.` in Datadog and are not tomcat specific metrics.  The list of metrics can be found [here](https://docs.datadoghq.com/integrations/java/?tab=host#data-collected).

1. Modify datadog-values-jmx.yaml and update with your API key.  Then run:

```
helm upgrade -f datadog-values-jmx.yaml datadog-agent stable/datadog
```

2. Deploy java pod using annotations method

```
kubectl create -f java-app-project-annotations.yaml
```

# METHOD 2 - Deploy Datadog Agent with jmx and use tomcat configmap

### Note: These metrics will be prefixed with `tomcat.` in Datadog.  The list of metrics can be found [here](https://docs.datadoghq.com/integrations/tomcat/#data-collected).

1. If you previously deployed the java application with K8s annotations delete it.

```
kubectl delete pod java-web
```

2. Update datadog-values-jmx-tomcat-configmap.yaml with your Datadog API key and modify the ad_identifiers with the __short image__ name of the container running tomcat and the jmx remote port.  Example running short image `java_web_project` with jmx remote port listening on 7199:

```
confd:
  tomcat.yaml: |-
    ad_identifiers:
      - java_web_project
    init_config:
      is_jmx: true
      collect_default_metrics: true
      conf:
        - include:
            type: ThreadPool
            attribute:
              maxThreads:
                alias: tomcat.threads.max
                metric_type: gauge
              currentThreadCount:
                alias: tomcat.threads.count
                metric_type: gauge
              currentThreadsBusy:
                alias: tomcat.threads.busy
                metric_type: gauge
        - include:
            type: GlobalRequestProcessor
            attribute:
              bytesSent:
                alias: tomcat.bytes_sent
                metric_type: counter
              bytesReceived:
                alias: tomcat.bytes_rcvd
                metric_type: counter
              errorCount:
                alias: tomcat.error_count
                metric_type: counter
              requestCount:
                alias: tomcat.request_count
                metric_type: counter
              maxTime:
                alias: tomcat.max_time
                metric_type: gauge
              processingTime:
                alias: tomcat.processing_time
                metric_type: counter
        - include:
            j2eeType: Servlet
            attribute:
              processingTime:
                alias: tomcat.servlet.processing_time
                metric_type: counter
              errorCount:
                alias: tomcat.servlet.error_count
                metric_type: counter
              requestCount:
                alias: tomcat.servlet.request_count
                metric_type: counter
        - include:
            type: Cache
            attribute:
              accessCount:
                alias: tomcat.cache.access_count
                metric_type: counter
              hitsCounts:
                alias: tomcat.cache.hits_count
                metric_type: counter
        - include:
            type: JspMonitor
            attribute:
              jspCount:
                alias: tomcat.jsp.count
                metric_type: counter
              jspReloadCount:
                alias: tomcat.jsp.reload_count
                metric_type: counter
    instances:
      - host: "%%host%%"
        port: "7199"
```

Then run:

```
helm upgrade -f datadog-values-jmx-tomcat-configmap.yaml datadog-agent stable/datadog
```

3.  Deploy the java app without K8s annotations

```
kubectl create -f java-app-project.yaml
```

# Checking Datadog Agent Status

Check the configuration of the datadog-agent pods

```
kubectl exec datadog-agent-<UID> agent configcheck
```

If the agent has autodiscovered the tomcat / jmx integration you will see something like the following.

### NOTE: Only the agents running on the same nodes as the java pod(s) will pick up the integration.

```
=== tomcat check ===
Configuration provider: kubernetes
Configuration source: kubelet:docker://dc00d0d6541e74fcceddddfe179e2424c70c5e5e3f89923338e8f1fe5050e288
Instance ID: tomcat:30f8cf07ea9fe86f
host: 10.40.0.8
port: "7199"
tags:
- short_image:java_web_project
- docker_image:dogdemo/java_web_project:latest
- kube_namespace:default
- pod_phase:running
- kube_container_name:java-web-project
- image_tag:latest
- image_name:dogdemo/java_web_project
~
Init Config:
{"collect_default_metrics":"true","is_jmx":"true"}
Auto-discovery IDs:
* docker://dc00d0d6541e74fcceddddfe179e2424c70c5e5e3f89923338e8f1fe5050e288
===
```

Check the status of the datadog-agent pods.

```
kubectl exec datadog-agent-<UID> agent status
```

### NOTE: Only the agents running on the same node as the java pod(s) will pick up the integration.

```
========
JMXFetch
========
  Initialized checks
  ==================
    tomcat
      instance_name : tomcat-10.40.0.8-7199
      message :
      metric_count : 19
      service_check_count : 0
      status : OK
  Failed checks
  =============
    no checks
```
