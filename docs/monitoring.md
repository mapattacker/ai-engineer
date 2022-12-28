# Monitoring

After model deploying, it is essential to monitor the system as life-real scenario is different from initial model training and testing. In essence, they can be summarize into 3 parts.

<figure>
  <img src="https://github.com/mapattacker/ai-engineer/blob/master/images/monitoring.png?raw=true" />
  <figcaption>3 Key Metrics to Monitor.<a href="https://www.udemy.com/course/testing-and-monitoring-machine-learning-model-deployments/"> Source</a></figcaption>
</figure>


Performance Metrics
 - Latency
 - Memory Footprint

Input Data
 - Drift in distribution of features
 - Kolmogorov-Smirnov (K-S) test
 - aka Data Drift

Model
 - aka Concept / Target Drift

## Libraries

Both [scikit-multiflow](https://scikit-multiflow.readthedocs.io/en/stable/api/api.html#module-skmultiflow.drift_detection) & [River](https://riverml.xyz/latest/api/drift/ADWIN/) have a list of algorithms for detecting concept drifts.


## Monitoring Systems

### Prometheus

[Prometheus](https://prometheus.io/) is the go-to open source metrics monitoring platform for ML systems.

### Grafana

[Grafana](https://prometheus.io/docs/visualization/grafana/) can be integrated with Prometheus to display the metrics in interactive visualizations.

### Kibana

[Kibana](https://www.elastic.co/kibana) is a free and open user interface that lets you visualize your logs, or model input distributions. This helps to detect concept or data drifts.

It can also create alerts. AWS has an integrated service called [Amazon Elasticsearch Service](https://aws.amazon.com/elasticsearch-service/) which sets up all the infrastructure ready for use.

### Evidently

[Evidently](https://docs.evidentlyai.com) helps evaluate and monitor machine learning models in production. It generates interactive reports or JSON profiles from pandas DataFramesor csv files. 

You can use visual reports for ad hoc analysis, debugging and team sharing, and JSON profiles to integrate Evidently in prediction pipelines or with other visualization tools.