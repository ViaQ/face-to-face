# Face to Face 2020/03 - Fluentbit -> Kafka -> Fluentd -> Elasticsearch

## What?
The purpose of this hackathon item was to investigate a new collector/aggregator solution for Cluster Logging.

## Why?
Currently Fluentd takes up decent amount of CPU and RAM <sup>[1](#comparison)</sup> on the host nodes in order to collect, enrich, and ship logs.

Fluentbit is a slimmer version of Fluentd (created by the same company) and is written in C. It does not have the same amount of plugins as Fluentd, however we felt it was best to just let it pull logs from the nodes and ship them to the aggregator as fast as possible.

Using Kafka lets us add reliability into our log delivery stack. Each of these components can be configured such that there are `ack`s for as messages are sent to the next component, ensuring that our messages are being persisted before we know we no longer require them. Leveraging Kafka also adds as an additional buffer for our logs to be stored in, easing the potential burden on our aggregator without as much worry for loss of logs.

Using Kafka also has the added benefit of decoupling our collectors from the destination. This lets us allow downtime of the destinations due to maintenaince, lack of connectivity, etc, and not impact the collectors. Currently we can only achieve this through limited buffering in the log collectors.


## How?

1. spin up OCP cluster with CLO and EO
1. Update clusterlogging/instance to be unmanaged
1. Delete fluentd ds
    ```
    oc delete ds fluentd
    ```
1. Install the strimzi operator through the console OperatorHub
1. create kafka cluster using strimzi operator:
    ```
    oc apply -f https://raw.githubusercontent.com/strimzi/strimzi-kafka-operator/0.16.2/examples/kafka/kafka-persistent-single.yaml -n openshift-logging
    ```
1. wait for kafka cluster to be ready:
    ```
    oc wait kafka/my-cluster --for=condition=Ready --timeout=300s -n openshift-logging
    ```
1. update cm/fluentd run.sh to add  `gem install fluent-plugin-kafka --no-document`:
    ```
    run.sh: |
        #!/bin/bash

        gem install fluent-plugin-kafka --no-document

        export MERGE_JSON_LOG=${MERGE_JSON_LOG:-false}
        CFG_DIR=/etc/fluent/configs.d
        ENABLE_PROMETHEUS_ENDPOINT=${ENABLE_PROMETHEUS_ENDPOINT:-"true"}
        OCP_OPERATIONS_PROJECTS=${OCP_OPERATIONS_PROJECTS:-"default openshift openshift- kube-"}
    ```
1. update cm/fluentd fluent.conf to look like:
    ```
    fluent.conf: |
        <source>
          @type kafka
          brokers my-cluster-kafka-bootstrap:9092
          topics infra, app, audit
        </source>

        # current gap with this filter is that we have to use ${tag} for parsing
        # also there is a time parsing bug if we have "kubernetes" => {} fields... from fluentbit
        #<filter app>
        #  @type kubernetes_metadata
        #  kubernetes_url "#{ENV['K8S_HOST_URL']}"
        #  cache_size "#{ENV['K8S_METADATA_CACHE_SIZE'] || '1000'}"
        #  watch "#{ENV['K8S_METADATA_WATCH'] || 'false'}"
        #  use_journal "#{ENV['USE_JOURNAL'] || 'nil'}"
        #  ssl_partial_chain "#{ENV['SSL_PARTIAL_CHAIN'] || 'true'}"
        #</filter>

        <match app>
          @type elasticsearch
          host elasticsearch.openshift-logging.svc.cluster.local
          port 9200
          scheme https
          ssl_version TLSv1_2
          index_name app
          user fluentd
          password changeme

          client_key '/var/run/ocp-collector/secrets/fluentd/tls.key'
          client_cert '/var/run/ocp-collector/secrets/fluentd/tls.crt'
          ca_file '/var/run/ocp-collector/secrets/fluentd/ca-bundle.crt'
          type_name _doc
        </match>

        <match infra>
          @type elasticsearch
          host elasticsearch.openshift-logging.svc.cluster.local
          port 9200
          scheme https
          ssl_version TLSv1_2
          index_name infra
          user fluentd
          password changeme

          client_key '/var/run/ocp-collector/secrets/fluentd/tls.key'
          client_cert '/var/run/ocp-collector/secrets/fluentd/tls.crt'
          ca_file '/var/run/ocp-collector/secrets/fluentd/ca-bundle.crt'
          type_name _doc
        </match>

        <match audit>
          @type elasticsearch
          host elasticsearch.openshift-logging.svc.cluster.local
          port 9200
          scheme https
          ssl_version TLSv1_2
          index_name audit
          user fluentd
          password changeme

          client_key '/var/run/ocp-collector/secrets/fluentd/tls.key'
          client_cert '/var/run/ocp-collector/secrets/fluentd/tls.crt'
          ca_file '/var/run/ocp-collector/secrets/fluentd/ca-bundle.crt'
          type_name _doc
        </match>

        <match *>
          @type elasticsearch
          host elasticsearch.openshift-logging.svc.cluster.local
          port 9200
          scheme https
          ssl_version TLSv1_2
          index_name catchall
          user fluentd
          password changeme

          client_key '/var/run/ocp-collector/secrets/fluentd/tls.key'
          client_cert '/var/run/ocp-collector/secrets/fluentd/tls.crt'
          ca_file '/var/run/ocp-collector/secrets/fluentd/ca-bundle.crt'
          type_name _doc
        </match>
    ```
1. Create fluentd deployment:
    ```
    oc create -f fluentd-deployment.yaml
    ```
1. create fluentbit cm:
    ```
    oc create configmap fluentbit --from-file=fluent-bit.conf=fluent-bit.conf --from-file=parsers.conf=parsers.conf
    ```
1. create fluentbit ds:
    ```
    oc create -f fluentbit.yaml
    ```

## Current Gaps

Currently the Fluentd k8s metadata plugin only correctly parses the container information from the tag which is based on the file path. There is an option to pull from a nested "kubernetes" field on the record, however this fails when trying to parse out the timestamp in the plugin. We would need to either fix this, or allow us to specify a field to parse out in lieu of using the tag in order to collect metadata at the aggregator level.

// the issue is that fluentd in-kafka automatically parses the time field to be a float instead of giving us an option to keep it as a string -- this would let us get around this error

As a more immediate work around we could keep collecting metadata at the collector level (in this case, Fluentbit [5]), however there would be less calls to the kubernetes service if this was at the aggregator level since we can independently scale Fluentd for aggregation separate from the number of nodes we are collecting from.

## Next Steps and Recommendations

There seemed to be a good performance increase over our current implementation of Fluentd -> Elasticsearch. I think some next steps would be to address the gap of the fabric8.io metadata plugin mentioned above, and have our performance team do a capacity test to figure out:
1. how well can fluentbit keep up with log ingestion for very noisy logs?
1. how many fluentd do we need per x rate of logs?
1. at what point do we need to horizontally scale our kafka pods?
1. verify/tune rate of log aggregation to prevent duplicate logs with multiple topic consumers

I believe we should eventually move to make this our default configuration. As long as a [new] consumer can send logs to `kafka` then we should be able to configure `fluentd` to write to different endpoints. I can see this being the future state of log forwarding as well.

Next steps will further be captured here: https://issues.redhat.com/browse/LOG-703

### Refs

1. https://strimzi.io/quickstarts/okd/
1. https://github.com/fluent/fluent-plugin-kafka
1. https://hub.docker.com/r/fluent/fluent-bit/
1. https://docs.fluentbit.io/manual/output/kafka
1. https://docs.fluentbit.io/manual/filter/kubernetes

---

<a name="comparison">1</a>
A quick and dirty comparison - each tailing a JSON log file and dumping into stdout:

                 VIRT    RES    SHR  S  %CPU  %MEM
    fluent-bit   25.8m   8.2m   5.0m S   1.0   0.1
    fluentd      414.3m  57.3m  8.8m S   5.7   0.4