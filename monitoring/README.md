# OpenShift Logging Monitoring

## What?

The goal for this project was to 1) review existing metrics and dashboards exposed by our current solution, and investigate with some improvements; as well as 2) try get the current set of dashboards running inside the console.

Participants: Lukas Vlcek, Christian Heidenreich

## Why?

Not only our customers, but also support continuous to struggle with getting the right data to troubleshoot OpenShift Logging + Elasticseach/Kibana. Monitoring is such an essential tool to help with that, that we need to improve the status quo.

## Outcome

* We have a basic set of Elasticsearch and fluentd metrics.
* We have a basic set of Elasticsearch dashboards, but nothing for fluentd.
* We got the Elasticsearch dashboards running with some modifications for it to be rendered inside the console.

### Elasticsearch dashboards into the console

Note: This works on OpenShift 4.4. The way the ConfigMap looks like may change in the future.

1. Deploy entire logging stack as per our documentation.
2. Make sure everything is running and you can see the ES metrics inside Prometheus.
3. Deploy the `grafana-dashboard-elasticsearch.yaml` into the `openshift-config-managed` namespace.
4. Go to "Monitoring" -> "Dashboards" inside the admin perspective and switch to the Elasticsearch dashboard.

## Next steps

* Clean up current ES dashboards and add it to the deployment for ES so that it will render inside the OpenShift console (https://issues.redhat.com/browse/LOG-674).
* Explore https://github.com/justwatchcom/elasticsearch_exporter and if it does provide more visibility into Elasticsearch. It comes with metrics, alerting rules and dashboards for Elasticsearch (https://issues.redhat.com/browse/LOG-675).

