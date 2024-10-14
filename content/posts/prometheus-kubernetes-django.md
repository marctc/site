---
author: "Marc Tudurí"
title: "Using Prometheus for monitoring Django applications in Kubernetes"
date: 2018-10-08
tags: ["prometheus","kubernetes", "monitoring"]
description: "Using Prometheus for monitoring Django applications in Kubernetes"
---

### Django application deployment architecture

Developing web applications with [Django](https://www.djangoproject.com/) is very easy and fun: if you just follow the [tutorial](https://docs.djangoproject.com/en/2.1/intro/tutorial01/) in a few hours you can have up and running your own website. However, deploying a Django site can be hard and you can end up running a site with a terrible performance if you don't follow the recommended guidelines to deploy a Django site to a production environment.

There are many options to deploy a Django application but you will finish using a [WSGI](http://wsgi.org/) (Web Server Gateway Interface) which is a Python specification that describes how a webserver communicates with web applications. One of the most popular and most used WSGI implementations is [uWSGI](https://uwsgi-docs.readthedocs.io/en/latest/) and it is the [recommended way](https://docs.djangoproject.com/en/2.1/howto/deployment/wsgi/) to deploy a Django application by the Django site.

Another key element to deploy a Django application is to use a reverse proxy in front of the WSGI application which provides a better way to route the inbound traffic to the site, serve static files, TLS termination and increase the traffic performance. We will end up with a schema like this:

```
the web client <-> the web server <-> the socket <-> uWSGI <-> Python
```

A common approach is to use [NGINX](https://nginx.org/) as a reverse proxy. [This tutorial](https://uwsgi-docs.readthedocs.io/en/latest/tutorials/Django_and_nginx.html)  explains with a very good level of detail how to set up a Django application with uWSGI and NGINX.

There are a lot of approaches to deploying Django applications in a cloud environment using Docker. As we deploy many of our applications in [Kubernetes](https://kubernetes.io) we developed a [Helm](http://helm.sh) chart in order to automatize the deployment of new sites. You can take a look at these charts [here](https://github.com/APSL/kubernetes-charts).

### Monitoring architecture

Once we have our Django application architecture for live environments it is strongly suggested to add a monitor system to our production infrastructure. Our idea is to have an [observable](https://medium.com/@copyconstruct/monitoring-and-observability-8417d1952e1c) system, so to achieve that, we need to get metrics of **each** piece of the Django application architecture: uWSGI, NGINX and Django server. There are many options for collecting metrics: Graphite, InfluxDB, StatsD but for our architecture, we will choose Prometheus as a monitoring system.

[Prometheus](https://prometheus.io/) works as pull model (in contrast of InfluxDB, which is push model) which means that the metrics are collected periodically from an HTTP endpoint and sent to a time series database. Pull model works very well for online-serving systems, i.e a website, because you don't need to know if metrics are properly sent to the metric storage: you decouple the delivery of metrics from the main application winning in simplicity and performance. However, this work for our uses cases so probably sometimes you should need to use a push model and send the metrics explicitly. For that purpose, you can use InfluxDB ([TICK Stack](https://www.influxdata.com/time-series-platform/)) as mentioned before or keep using Prometheus with the [Pushgateway](https://prometheus.io/docs/practices/pushing/) module.

On the other hand, a great advantage of Prometheus is the existence of [Prometheus exporters](https://prometheus.io/docs/instrumenting/exporters/). Many of open source software projects include natively its support for exposing metrics in Prometheus format. If a software or library doesn't have it,  you will need an exporter which can be attached as a running process to any software that doesn't export Prometheus metrics by default. This will expose all the metrics on HTTP endpoint  (typically `/metrics`) at a [specific port](https://github.com/prometheus/prometheus/wiki/Default-port-allocations).

Finally, another one of the great advantages of Prometheus, besides it is open source, is that it belongs to the [CNCF](https://www.cncf.io/) ecosystem as [graduated project](https://www.cncf.io/announcement/2018/08/09/prometheus-graduates/) for monitoring in cloud-native environments and works [very good](https://www.weave.works/blog/prometheus-kubernetes-perfect-match/) in Kubernetes environments, so is becoming to a de-facto standard solution for monitoring distributed system. Also, the Prometheus [exposition format](https://prometheus.io/docs/instrumenting/exposition_formats/) is evolving into an open standard: [OpenMetrics](https://openmetrics.io/)

You can read more about how Prometheus work [here](https://prometheus.io/docs/introduction/overview/) and why important companies like Weavework [choose](https://www.weave.works/technologies/monitoring-kubernetes-with-prometheus/) Prometheus as a monitoring system.

To install Prometheus you can use a [Helm chart](https://github.com/helm/charts/tree/master/stable/prometheus) or with the [Prometheus Operator](https://github.com/coreos/prometheus-operator) which seems the best option to deploy a Prometheus instance to Kubernetes.


### Setting up a Django application to work with Prometheus

The following steps suppose that you have your application properly dockerized and prepared to run in Kubernetes environment. A few changes to the configuration of uWSGI, NGINX and Django are needed to provide a proper monitoring.

#### Enable uWSGI monitoring

uWSGI provides by default a [stats module](https://uwsgi-docs.readthedocs.io/en/latest/StatsServer.html) which opens a port and expose metrics in JSON format:

```json
{
  "workers": [{
    "id": 1,
    "pid": 31759,
    "requests": 0,
    "exceptions": 0,
    "status": "idle",
    "rss": 0,
    "vsz": 0,
    "running_time": 0,
    "last_spawn": 1317235041,
    "respawn_count": 1,
    "tx": 0,
    "avg_rt": 0,
    "apps": [{
      "id": 0,
      "modifier1": 0,
      "mountpoint": "",
      "requests": 0,
      "exceptions": 0,
      "chdir": ""
    }]
  }
```

This endpoint will be used by a Prometheus exporter to scrape and expose it in Prometheus format. We need to change the `uwsgi.ini` file to open the port:

```ini
stats = :1717
stats-http = true
```

#### Enable NGINX monitoring

Similar to uWSGI, NGINX provides a [stub status module](http://nginx.org/en/docs/http/ngx_http_stub_status_module.html) which provides a basic status information:

```
Active connections: 291 
server accepts handled requests
 16630948 16630948 31070465 
Reading: 6 Writing: 179 Waiting: 106 
```
To enable this config, add these lines to your NGINX config:

```nginx
server {
    listen 80 default_server;
    ...
    location /nginx/status {
        stub_status on;
        access_log off;
    }
}
```

If you need more detailed metrics about NGINX you can use [NGINX virtual host traffic status module](https://github.com/vozlt/nginx-module-vts) instead.

#### Enable Django monitoring

This is the most important while monitoring our Django application. For that purpose, we will use [django-prometheus](https://github.com/korfuri/django-prometheus). This library allows us to expose metrics that comes from the processing of incoming and outbound requests (like the total requests of a specific view) without adding to much code. If you need to instrument your application placing business logic metrics you can also use this library as includes the Python client for Prometheus.

To enable basic Prometheus monitoring with Django, follow these steps:

* Install the package: `pip install django-prometheus`
* Add the library to `INSTALLED_APPS` list in `settings.py`:

```python
    INSTALLED_APPS = [
        ...
        'django_prometheus',
    ]
```
* Add the required middlewares in `MIDDLEWARE` list in `settings.py`. The order is important so `PrometheusBeforeMiddleware` needs to be the first one in the list and `PrometheusAfterMiddleware` needs to be the last:

```python
    MIDDLEWARE = [
        'django_prometheus.middleware.PrometheusBeforeMiddleware',
        ...
        'django_prometheus.middleware.PrometheusAfterMiddleware',
    ]
```

* Add this url pattern in `urls.py`:

```python
  urlpatterns = [
     ...
     path('', include('django_prometheus.urls')),
  ]
```

* Finally, it's recommended restrict the access the provided endpoint in `urls.py` to avoid to be public. We can disable it in the NGINX config:

```nginx

   server {
        listen 8000 default_server;
       ...

        location /metrics {
            deny  all;
            access_log off;
            error_log off;
        }
   }
```

A common [issue](https://github.com/korfuri/django-prometheus/issues/34) that happends running the `./manage.py collectstatic` command and it is not documented yet could be fixed adding this to the`settings.py` file.

```python
PROMETHEUS_EXPORT_MIGRATIONS = False
```

### Setting up the Kubernetes Deployment

Once we have the new Docker images for the changes previously introduced it's time to modify our Kubernetes deployment as Prometheus should be able to gather all exposed metrics. For that purpose, we will use the above mentioned Prometheus exporters.

Thanks to the Prometheus community there are a lot of exporters for a lot of software projects. So we will use existing exporters for uWSGI and NGINX that will be added as [sidecar containers](https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns/) of our application deployment:

* uWSGI: [https://github.com/timonwong/uwsgi_exporter](https://github.com/timonwong/uwsgi_exporter)
* NGINX: [https://github.com/discordianfish/nginx_exporter](https://github.com/discordianfish/nginx_exporter). But if you need more metrics your probably should use [https://github.com/hnlq715/nginx-vts-exporter](https://github.com/hnlq715/nginx-vts-exporter).

Hence, the result Deployment should look as follows:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: myapp
  namespace: mynamespace
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      tier: web
  strategy:
    type: Recreate
  template:
    metadata:
      annotations:
        prometheus.io/scrape: 'true'    
      labels:
        app: myapp
        tier: web
    spec:
      affinity: {}
      containers:
      ...
      # uwsgi exporter
      - args:
        - --stats.uri=http://localhost:1717
        image: timonwong/uwsgi-exporter
        imagePullPolicy: Always
        name: uwsgi-exporter
        resources:
          requests:
            cpu: 10m
            memory: 10Mi
          limits:
            cpu: 100m
            memory: 25Mi
        livenessProbe:
          httpGet:
            path: /-/healthy
            port: 9117
      # nginx exporter
      - args:
        - --nginx.scrape_uri=http://localhost/nginx/status
        image: fish/nginx-exporter
        imagePullPolicy: Always
        name: nginx-exporter
        resources:
          requests:
            cpu: 10m
            memory: 10Mi
          limits:
            cpu: 100m
            memory: 25M
        ports:
        livenessProbe:
          httpGet:
            path: /-/healthy
            port: 9113
```

Since all containers in the Pod shares the same [namespace network](https://kubernetes.io/docs/concepts/workloads/pods/pod/#resource-sharing-and-communication) the exposed endpoints of uWSGI and NGINX will be visible for each exporter.

Notice that we didn't specify anything for Django metrics as is by default exposed a `localhost:<UWSGI_PORT>/metrics`.

Look that we used `prometheus.io/scrape: 'true'` tag. This enables [Prometheus Kubernetes service discover](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#%3Ckubernetes_sd_config%3E) to find out which Deployments have metrics endpoints to scrape.

The problem doing this is that Prometheus will try to fetch `/metrics` and scrape all the open HTTP ports of all the containers of the tagged Deployment causing that the Prometheus scraper throws errors when trying to scrape HTTP servers that don't expose metrics (uWSGI, NGINX,...). 

The solution is change the `prometheus.yaml` config file and add these lines to `scrape_configs` section:

```yaml
global:
   ...
   scrape_configs:
      - job_name: 'kubernetes-pods-containers'
        kubernetes_sd_configs:
        - role: pod
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
          target_label: __address__
        - source_labels: [__meta_kubernetes_pod_container_port_name]
          action: keep                    
          regex: metrics 
```

The lines above will config Prometheus to scrape only those endpoints of containers of the Deployment that have an explicit `port` section in its specification: 

```yaml
      - args:
        - --stats.uri=http://localhost:1717
        image: timonwong/uwsgi-exporter
        imagePullPolicy: Always
        name: uwsgi-exporter
        resources:
          requests:
            cpu: 10m
            memory: 10Mi
          limits:
            cpu: 100m
            memory: 25Mi
        livenessProbe:
          httpGet:
            path: /-/healthy
            port: 9117
        ports:
        - name: "metrics"
          containerPort: 9117
```

Now we have our Django application fully monitored with Prometheus. Is also highly recommended to monitor all the external services and external backends such as databases, cache systems, load balancers, etc.

### Integration with Stackdriver

It is possible to have a Prometheus installed in your cluster and at the same time send your metrics to [Stackdriver](https://cloud.google.com/stackdriver/) which is an excellent option if you use Kubernetes under Google Cloud. Just follow [this steps](https://cloud.google.com/monitoring/kubernetes-engine/prometheus) without changing anything of the explained before.

#### Alternative option

An alternative way is instead of to deploy a full Prometheus installation you can just put a special side-car container with an image of [prometheus-to-sd](https://github.com/GoogleCloudPlatform/k8s-stackdriver/tree/master/prometheus-to-sd) which is a little piece of software that translates the metrics of your Prometheus exporters and send it to Stackdriver. You just need to add this container to your Deployment:

```yaml
      - name: prometheus-to-sd
        image: gcr.io/google-containers/prometheus-to-sd:latest
        command:
        - /monitor
        - --source=:http://localhost:9117/metrics # uwsgi
        - --source=:http://localhost:9113/metrics # nginx
        - --source=:http://localhost:8080/metrics # django
        - --stackdriver-prefix=custom.googleapis.com
        - --pod-id=$(POD_ID)
        - --namespace-id=$(POD_NAMESPACE)
        env:
        - name: POD_ID
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace    
```
You need to add a line with `--source=:` and the path for every exporter that you want to collect the metrics.

You have to keep in mind that both solutions are **experimental**.  There also some [discussions](https://github.com/Stackdriver/stackdriver-prometheus/issues/15) of which solution is better.

### Example usage of a real app

It is interesting to show some of the most important metrics of each exporter that could be useful to use to monitoring and alerting.  If you are not familiar with Prometheus exporting format this will be interesting for you too.

#### NGINX metrics

* Average response time (milliseconds):

```
# HELP http_request_duration_microseconds The HTTP request latencies in microseconds.
# TYPE http_request_duration_microseconds summary
http_request_duration_microseconds{handler="prometheus",quantile="0.5"} 2769.075
http_request_duration_microseconds{handler="prometheus",quantile="0.9"} 3520.802
http_request_duration_microseconds{handler="prometheus",quantile="0.99"} 8228.981
http_request_duration_microseconds_sum{handler="prometheus"} 1.5934673549999993e+06
http_request_duration_microseconds_count{handler="prometheus"} 408
```

* Connections processed by NGINX:

```
# HELP nginx_connections_current Number of connections currently processed by nginx
# TYPE nginx_connections_current gauge
nginx_connections_current{state="active"} 1
nginx_connections_current{state="reading"} 0
nginx_connections_current{state="waiting"} 0
nginx_connections_current{state="writing"} 1
```

#### uWSGI metrics

Metrics that could be obtained of each running uWSGI worker.

* Transmited bytes:

```
# HELP uwsgi_worker_transmitted_bytes_total Worker transmitted bytes.
# TYPE uwsgi_worker_transmitted_bytes_total counter
uwsgi_worker_transmitted_bytes_total{stats_uri="http://localhost:1717",worker_id="1"} 2.0374868e+07
```

* Average response time (in seconds):

```
# HELP uwsgi_worker_average_response_time_seconds Average response time in seconds.
# TYPE uwsgi_worker_average_response_time_seconds gauge
uwsgi_worker_average_response_time_seconds{stats_uri="http://localhost:1717",worker_id="1"} 0.093864
```

* Harakiri count:

```
# HELP uwsgi_worker_harakiri_count_total Total number of harakiri count.
# TYPE uwsgi_worker_harakiri_count_total counter
uwsgi_worker_harakiri_count_total{stats_uri="http://localhost:1717",worker_id="1"} 0
```

#### Django metrics

Metrics exposed from the middlewares of Prometheus Django package.

* Response by status code:

```
# HELP django_http_responses_total_by_status Count of responses by status.
# TYPE django_http_responses_total_by_status counter
django_http_responses_total_by_status{status="200"} 630.0
django_http_responses_total_by_status{status="404"} 103.0
django_http_responses_total_by_status{status="301"} 3.0
```

* Total number of requests by view:

```
# HELP django_http_requests_total_by_view_transport_method Count of requests by view, transport, method.
# TYPE django_http_requests_total_by_view_transport_method counter
django_http_requests_total_by_view_transport_method{method="POST",transport="https",view="product:add-to-cart"} 4.0
django_http_requests_total_by_view_transport_method{method="GET",transport="https",view="cart:cart-summary"} 67.0
```

* Histogram of latencies by view:

```
# HELP django_http_requests_latency_seconds_by_view_method Histogram of request processing time labelled by view.
# TYPE django_http_requests_latency_seconds_by_view_method histogram
django_http_requests_latency_seconds_by_view_method_bucket{le="0.005",method="GET",view="cart:cart-summary"} 0.0
...
django_http_requests_latency_seconds_by_view_method_bucket{le="0.5",method="GET",view="cart:cart-summary"} 67.0
django_http_requests_latency_seconds_by_view_method_bucket{le="0.75",method="GET",view="cart:cart-summary"} 67.0
django_http_requests_latency_seconds_by_view_method_bucket{le="1.0",method="GET",view="cart:cart-summary"} 67.0
...
django_http_requests_latency_seconds_by_view_method_bucket{le="+Inf",method="GET",view="cart:cart-summary"} 67.0
```

### Conclusions

It may seem that follow these guidelines will cost you a lot of work and those metrics are only a few that are provided by default, so you **should** instrument your application setting the proper metrics according to your business requirements. That's the real effort: change your mind and do this job from the beginning. 

The robustness and simplicity of Prometheus is a game changer in the world of monitoring applications and it could save a lot of time and money of operational tasks if you use it properly.