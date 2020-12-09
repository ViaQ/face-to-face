# Virtual Face to Face 2020/12 - Self Healing Elasticsearch

## Who

* Eric Wolinetz - @ewolinetz
* Brett Jones - @blockloop

## What

### Exposing metrics of components and consuming in operator for making tuning suggestions

####	As a developer for Logging Exploration, I want to consume metrics for my log storage (such as storage usage and flow rate) so that my operator can make tuning recommendations so users can act on them.

This hackathon item originally started as a means to consume metrics within the operator for making recommendations (e.g. phase 4 operator task), however after some discussion we argued that the operator already was phase 4 due to the metrics and alerting rules it created and it would be better to let Alert Manager maintain state for alerts and let EO consume and act on alerts when they were firing.

## Design

### Alerts to look for

Initially we set out to select four alerts that we could check their state to determine if the operator should scale up the Elasticsearch cluster it was managing.

We settled on:
1. [ElasticsearchJVMHeapUseHigh](https://github.com/openshift/elasticsearch-operator/blob/master/files/prometheus_alerts.yml#L86)
    ```
    "expr": |
      sum by (cluster, instance, node) (es_jvm_mem_heap_used_percent) > 75
    "for": "10m"
    ```
1. [ElasticsearchNodeDiskWatermarkReached](https://github.com/openshift/elasticsearch-operator/blob/master/files/prometheus_alerts.yml#L35)
    ```
     "expr": |
      sum by (instance, pod) (
        round(
          (1 - (
            es_fs_path_available_bytes /
            es_fs_path_total_bytes
          )
        ) * 100, 0.001)
      ) > on(instance, pod) es_cluster_routing_allocation_disk_watermark_low_pct
    "for": "5m"
    ```
1. [ElasticsearchDiskSpaceRunningLow](https://github.com/openshift/elasticsearch-operator/blob/master/files/prometheus_alerts.yml#L116)
    ```
    expr: |
      sum(predict_linear(es_fs_path_available_bytes[6h], 6 * 3600)) < 0
    for: 1h
    ```
1. [ElasticsearchWriteRequestsRejectionJumps](https://github.com/openshift/elasticsearch-operator/blob/master/files/prometheus_alerts.yml#L25)
    ```
    "expr": |
      round( writing:reject_ratio:rate2m * 100, 0.001 ) > 5
    "for": "10m"
    ```

### Consuming the alerts

The next step was to come up with a mechanism to consume these alerts from Alert Manager and make sense of them to know if we need to scale.

``` alertmanager/api.go
type Alerts struct {
	HeapHigh            bool
	LowWatermark        bool
	DiskAvailabilityLow bool
	WriteRejections     bool
}

type API interface {
	Alerts() (*Alerts, error)
}

func (c *Client) Alerts() (*Alerts, error) {
	resp, err := c.allAlerts()

	if err != nil {
		return nil, err
	}

	res := &Alerts{}

	for _, alert := range resp.Alerts {
		raw, ok := alert.Labels[alertNameLabel]
		if !ok {
			continue
		}
		alertName, ok := raw.(string)
		if !ok {
			continue
		}
		if alert.Status.State != "active" {
			continue
		}

		switch strings.ToLower(alertName) {
		case "elasticsearchjvmheapusehigh":
			res.HeapHigh = true
		case "elasticsearchnodediskwatermarkreached":
			res.LowWatermark = true
		case "elasticsearchdiskspacerunninglow":
			res.DiskAvailabilityLow = true
		case "elasticsearchwriterequestsrejectionjumps":
			res.WriteRejections = true
		}
	}

	return res, nil
}

func (c *Client) allAlerts() (*GetAlertsResponse, error) {
	uri := c.baseURL + "/api/v1/alerts"

	req, err := http.NewRequest(http.MethodGet, uri, nil)
	if err != nil {
		return nil, kverrors.Wrap(err, "failed to create request")
	}

	req.Header.Set("Authorization", fmt.Sprintf("Bearer %s", c.token))

	res, err := c.httpClient.Do(req)
	if err != nil {
		return nil, kverrors.Wrap(err, "failed to GET endpoint")
	}
	defer func() { _ = res.Body.Close() }()
	var alerts *GetAlertsResponse
	if err := json.NewDecoder(res.Body).Decode(&alerts); err != nil {
		return nil, kverrors.Wrap(err, "failed to decode alerts message")
	}

	return alerts, nil
}
```

### Making sense of the alerts

Once we had a simplified means to check if these alerts were firing, we needed to react to these alerts and scale up.

As part of this hackathon, to avoid needing to make API changes we made assumptions about the topology of the cluster and manipulated the CR. Also to avoid the cluster being scaled up to infinity we set an upper bounds of five nodes that we can scale up to.

``` hack/cr.yaml
---
apiVersion: "logging.openshift.io/v1"
kind: "Elasticsearch"
metadata:
  name: "elasticsearch"
spec:
  managementState: "Managed"
  nodeSpec:
    resources:
      limits:
        memory: 1Gi
      requests:
        cpu: 100m
        memory: 1Gi
  nodes:
  - nodeCount: 1
    roles:
    - master
    storage: {}
  - nodeCount: 1
    roles:
    - client
    - data
    storage: {}
  redundancyPolicy: ZeroRedundancy
  indexManagement:
    policies:
    - name: infra-policy
      pollInterval: 59m
      phases:
        hot:
          actions:
            rollover:
              maxAge:   20m
        delete:
          minAge: 50m
    - name: app-policy
      pollInterval: 59m
      phases:
        hot:
          actions:
            rollover:
              maxAge:   20m
        delete:
          minAge: 50m
    mappings:
    - name:  infra
      policyRef: infra-policy
      aliases:
      - infra
      - logs.infra
    - name:  app
      policyRef: app-policy
      aliases:
      - app
      - logs.app
```

``` k8shandler/self_heal.go
const max_node_count = 5

func (er *ElasticsearchRequest) ReconfigureCR() error {

	httpClient := &http.Client{
		Transport: &http.Transport{
			Proxy: http.ProxyFromEnvironment,
			DialContext: (&net.Dialer{
				Timeout:   30 * time.Second,
				KeepAlive: 30 * time.Second,
				DualStack: true,
			}).DialContext,
			MaxIdleConns:          100,
			IdleConnTimeout:       90 * time.Second,
			TLSHandshakeTimeout:   10 * time.Second,
			ExpectContinueTimeout: 1 * time.Second,
			TLSClientConfig: &tls.Config{
				InsecureSkipVerify: true,
			},
		},
	}

	amClient := alertmanager.NewClient("https://alertmanager-main-openshift-monitoring.apps.<user>.devcluster.openshift.com",
		httpClient, "<kube-admin token>")
	alerts, err := amClient.Alerts()

	if err != nil {
		return err
	}

	if alerts == nil {
		return nil
	}

	// 0. This hackathon demo requires that masters are not also data nodes
	// 1. evaluate alerts and adjust the cr we know about
	// 2. compare the adjusted cr to what is currently out there -- it may not be the same
	// 3. if different, patch it *
	// * there a lot of nuances that can happen but for the sake of this demo its going to be a primitive change and assume only the node counts need to change

	// queue up several different changes at once
	cluster := er.cluster

	// scale up data/ingest
	if alerts.HeapHigh || alerts.DiskAvailabilityLow || alerts.LowWatermark {
		for index, node := range cluster.Spec.Nodes {
			if isDataNode(node) {
				if cluster.Spec.Nodes[index].NodeCount < max_node_count {
					log.Info("scaling up data nodes!", "alerts", alerts)
					cluster.Spec.Nodes[index].NodeCount += 1
				}
			}
		}
	}

	// scale up master based on number of data nodes
	dataCount := getDataCount(cluster)
	masterCount := getMasterCount(cluster)

	if dataCount >= 3 && masterCount < 3 {
		for index, node := range cluster.Spec.Nodes {
			if isMasterNode(node) {
				log.Info("Scaling up master nodes!")
				cluster.Spec.Nodes[index].NodeCount = 3
			}
		}
	}

	currentCR := &v1.Elasticsearch{}
	objectKey := types.NamespacedName{
		Name:      cluster.Name,
		Namespace: cluster.Namespace,
	}

	if err := er.client.Get(context.TODO(), objectKey, currentCR); err != nil {
		return err
	}

	// check if the master and data node counts already match
	different := false
	for _, currentNode := range currentCR.Spec.Nodes {
		for _, desiredNode := range cluster.Spec.Nodes {
			if (isMasterNode(currentNode) && isMasterNode(desiredNode)) ||
				(isDataNode(currentNode) && isDataNode(desiredNode)) {

				if currentNode.NodeCount < desiredNode.NodeCount {
					different = true
				}
			}
		}
	}

	if different {
		// update the CR
		err := retry.RetryOnConflict(retry.DefaultRetry, func() error {
			if err := er.client.Get(context.TODO(), objectKey, currentCR); err != nil {
				return err
			}

			for currentIndex, currentNode := range currentCR.Spec.Nodes {
				for _, desiredNode := range cluster.Spec.Nodes {
					if (isMasterNode(currentNode) && isMasterNode(desiredNode)) ||
						(isDataNode(currentNode) && isDataNode(desiredNode)) {

						if currentNode.NodeCount < desiredNode.NodeCount {
							currentCR.Spec.Nodes[currentIndex].NodeCount = desiredNode.NodeCount
						}
					}
				}
			}

			if err := er.client.Update(context.TODO(), currentCR); err != nil {
				return err
			}
			return nil
		})
		return err
	}

	return nil
}
```

## Current Gaps

Currently the operator needs to poll Alert Manager which means there is an up to thirty second gap from when an alert fires to when the operator is aware that the alert fired. Ideally, the operator could be notified by Alert Manager and could act upon it immediately.

## Next Steps and Recommendations

The end goal is for a customer to not need to be responding to alerts, so next steps would be for us to continue hardening and improving upon our alerts to ensure that they fire when it is appropriate. We currently have static durations that we check for and likely should update this to be based on our curation schedule.

We would also need to change the API for the elasticsearch CRs to remove tuning knobs since customers would be relying on the operator to scale and tune the cluster.

There has been discussions about what this API might look like as part of the API.Next presentation given during the face to face. We could make the buy-in for phase 5 being the use of this api.

While doing investigation for how we should split up the nodes in the `hack/cr.yaml` file, the elastic.co docs recommend for large clusters avoiding letting the masters act as coordinating and data nodes. For within our topology that would mean creating a separate node that only uses role `master` (`client` is used to determine which nodes are fronted by our elasticsearch service, and in the context of the elastic.co does this is the same as an `ingest` node).

## Refs
1. [EO hackathon working branch](https://github.com/ewolinetz/elasticsearch-operator/commits/hackathon_dec_2020)
1. [Hackathon working doc](https://docs.google.com/document/d/1X5qdFG8mTIK-aZ9JQHDT1XHu1xabo1qtvVdjgW6xaOA/edit#)
1. [API.Next slides](https://docs.google.com/presentation/d/1Mf0mxSAdUz3zfzs3VP9agqI6ysoj2Tdx-RichEXJSRA/edit)
1. [ES node roles](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/modules-node.html)