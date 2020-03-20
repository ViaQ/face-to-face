# Evaluation of Loki as a low retention store

## Motivation

Originally, Elasticsearch was introduced for storing logs for a given short amount of time which at the same time had fantastic querying capabilities. Unfortunately, Elasticsearch has a lot more to offer and is therefore quite “heavy” + difficult to operator without the necessary experience. We have been in many cases, where there were issues with either customers using more than expected or not as expected; but at the same time we struggle with it’s complexity. All that, just to have a low-retention storage for logs. For providing a better integration for log exploration inside the console, we would also need a storage available all the time and Elasticsearch is just too heavy.

## Evaluation criteria

1. Low retention: e.g. 7 days to 30 days
- Amount of storage we need?
- What does the writing path look like? (memory -> persistent disk)
- What are the stores? (No exclusive cloud providers)
- What does the metadata model look like?
2. Multitenancy
- How is it provided by the solution?
- What does it look like on the storage level?
3. K8S native
- Are the components containerized (stateless vs stateful)?
- Do we need an operator for the store?
4.Correlations with metrics
- Labels?
- Time-series?
- Query/Filter
5. Does it have a query language?
- How complicated is it?

## Preliminary documentation assessment

| Topic                     | Notes/Results                                                                                                                                                   |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Low-Retetion              | Chunk based format of compressed logs. Boils down to how fast ingesters can write to a cassandra. Sizing on Loki Component Replica vs. Cassandra cluster nodes? |
| Multi-Tenancy             | Available based on vendor HTTP header. N/A storage level isolation?                                                                                             |
| K8s Native Model          | Distributor/Ingester/Store/TableManager: Probably as Statefulsets. Need 1x Stateful set for ring storage (etcd/consul) and 1x for Index (e.g. Cassandra)        |
| Signal Correlation        | Based on same k8s metadata if used with Promtail format.                                                                                                        |
| Query/Filter Capabilities | TBA                                                                                                                                                             |
| Clients/Plugins           | [fluent-plugin-grafana-loki](https://github.com/grafana/loki/blob/master/docs/clients/fluentd/README.md#fluentd), Needs fluentd >=1.9.0 <2.0                    |
|                           | [fluentbit](https://github.com/grafana/loki/blob/master/cmd/fluent-bit/README.md)                                                                               |

## Spike

The spike targets a mimimalistic installation to collect and store logs based on [cluster-logging-operator](https://github.com/openshift/cluster-logging-operator). The following sections describe how to apply the accompanying patches and provide for each cluster-logging component an step-by-step installation procedure. For a shift demonstration the following assumptions apply:
- Deployment of a collector-only cluster-logging-instance: This implies that the cluster-logging facility provides only the fluentd and log forwarding facility.
- Use of the tech-preview log forwarding feature to forward logs to an unmanaged - not cluster-logging-operator reconciled - Loki installation.
- Loki in cluster installation, but manually managed by an administrator.

1. To use the vendored fluent-plugin-grafana-loki plugin, apply the following [openshift/origin-aggregated-logging#1856](https://github.com/openshift/origin-aggregated-logging/pull/1856)
2. To add support to cluster-logging-operator, apply the following PR [openshift/cluster-logging-operator#428](https://github.com/openshift/cluster-logging-operator/pull/428)
3. Run E2E tests in cluster-logging-operator againt a dev cluster:
   `IMAGE_FLUENTD_IMAGE=image-registry.openshift-image-registry.svc:5000/openshift/origin-logging-fluentd:latest make test-e2e-local`

## Results
TBD

## Next steps
TBD.
