# Intro grafana operator in the PaaS

## enable grafana in the PaaS

In de spec part of the PaaS you can define the capabilities. In this case grafana.
It takes no parameters, so define it like this.

```
spec:
  capabilities: 
    grafana: {}
```

After you created a pull request, the grafana operator will create a grafana instance 
in a namespace called grafana within your PaaS context.

In the namespace grafana you can find a pod running a grafana instance, the service and the route.
Also necessary secrets a PVC and configmaps are created.

The PaaS operator will automatically create a datasource for prometheus in the grafana namespace.

If you use the search menu to search for an object of type GrafanaDatasource you will find a 
datasource called <namespace>-prometheus.


## grafana dashboard with predefined metrics
If you select the route of the grafana instance in the browser you will see a dashboard with predefined metrics.
You can login with your github credentials.



If you download a default dashboard from [grafana comunity dashboards](https://grafana.com/grafana/dashboards/?plcmt=oss-nav)

Under dashboards you can now select new in the top right corner, and then import. Edit the dashboard to get your specif namespace, datasource and such set and save it.


## create custom metrics for your application

The data you get by default is general openshift/kubernetes metrics. This will give you info on the running pods, nodes, etc.
If you want to get metrics from your application you need to expose them. In general there are components in your framework that do this. For example in spring boot you can use actuator.
Or in quarkus you can use smallrye metrics. These metrics are exposed by on a http endpoint. 

For my fotoshow app I have enabled smallrye metrics and exposed the metrics on /q/metrics.

If you now import the JVM (micrometer) dashboard from grafana.com and modifying the enviroment settings, you will not see the metrics from the application. So why not.

### enable prometheus scanning of your application

The default prometheus configuration does not scan your application. Since you can not edit the system wide prometheus configuration you have to enable this with a service monitor.

The service monitor is a kubernetes object that defines how prometheus should scan your application and will be picked up by the prometheus operator.

#### define a service monitor

create a file called service-monitor.yaml in the grafana namespace with the following content:

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    application: fotoshow
  name: java-app-monitor
  namespace: sam-sam2-test
spec:
  endpoints:
    - interval: 30s
      path: /q/metrics
      targetPort: 18181
  selector:
    matchLabels:
      app: fotoshow
```

the selector defines which pods should be scanned.
the endpoints defines the path where the metrics are exposed.

If you now apply the service monitor, prometheus will start scanning your application. The metrics will be available in grafana after a few minutes.

## gitops

Make sure to add the service monitor and dashboards to your gitops deploy-repo so they are deployed to your PaaS environment.

## links
https://medium.com/daemon-engineering/exposing-metrics-to-prometheus-with-service-monitors-326f38b2daf1
https://observability.thomasriley.co.uk/prometheus/configuring-prometheus/using-service-monitors/
https://grafana.com/grafana/dashboards/?plcmt=oss-nav




