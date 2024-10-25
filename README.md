# Mirror Maker Demo

## Pre-requisites

Install the Red Hat Streams for Apache Kafka operator. This can be done within your target namespace(s) (ie, 'streams' & 'dc2-streams'), or globally across all namespaces.

Install the Prometheus operator. This can be done within your target namespace (ie, 'streams'), or globally across all namespaces.

Install the Grafana operator. This can be done within your target namespace (ie, 'streams'), or globally across all namespaces.

You can download the `oc` CLI for your platform via the OpenShift web console. Click on the <img height="24" src="img/question-mark.png"/> -> "Command Line Tools" link in the upper-right corner. Unzip the `oc` CLI executable into the "${PROJECT_ROOT}/bin" directory.

## Installation

__Environment__

```
#
# Set your env variables.
export PROJECT_ROOT="$(pwd)"
export PATH="${PROJECT_ROOT}/bin:${PATH}"

export DOMAIN="apps.cluster-b47nb.b47nb.sandbox3205.opentlc.com"
```

__Prometheus/Grafana__

```
#
# Create the monitoring project. Do this as a regular user.
cd "${PROJECT_ROOT}/monitoring"
oc new-project monitoring

#
# Create/configure the Prometheus server. These steps must be completed as cluster-admin.
cd "${PROJECT_ROOT}/monitoring"
oc -n monitoring apply -f ./prometheus-additional-scrape-secret.yaml
oc -n monitoring apply -f ./prometheus.yaml
oc -n monitoring apply -f ./prometheus-strimzi-pod-monitor.yaml
# End cluster-admin steps.


#
# Create/configure the Grafana server.
cd "${PROJECT_ROOT}/monitoring"
oc -n monitoring apply -f ./grafana.yaml
oc -n monitoring expose service grafana-service
oc -n monitoring apply -f ./grafana-datasource.yaml
oc -n monitoring apply -f './grafana-*-dashboard.yaml'
```

__Red Hat Streams for Apache Kafka__

```
#
# Create/configure DC1 the Kafka cluster.
cd "${PROJECT_ROOT}/dc1"
oc new-project dc1
oc -n dc1 apply -f ./kafka-metrics-configmap.yaml
oc -n dc1 apply -f ./kafka-cluster.yaml
oc -n dc1 get secret dc1-cluster-cluster-ca-cert -o jsonpath='{.data.ca\.crt}' | base64 -d > ${PROJECT_ROOT}/tls/dc1-ca.crt
oc -n dc1 apply -f ./kafka-topics.yaml

#
# Create/configure DC1 the Kafka cluster.
cd "${PROJECT_ROOT}/dc2"
oc new-project dc2
oc -n dc2 apply -f ./kafka-metrics-configmap.yaml
oc -n dc2 apply -f ./kafka-cluster.yaml
oc -n dc2 create secret generic dc1-cluster-cluster-ca-cert --from-file=ca.crt=${PROJECT_ROOT}/tls/dc1-ca.crt
cat ./kafka-mirror-maker-2.yaml | envsubst | oc -n dc2 apply -f-

```

## Testing

```
#
# Open a terminal window for DC1 and start a producer.
oc -n dc1 run kafka-producer -ti --image=registry.redhat.io/amq-streams/kafka-37-rhel9:2.7.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --bootstrap-server dc1-cluster-kafka-bootstrap:9092 --topic demo.footopic

#
# Open a terminal window for DC2 and start a consumer.
oc -n dc2 run kafka-consumer -ti --image=registry.redhat.io/amq-streams/kafka-37-rhel9:2.7.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server dc2-cluster-kafka-bootstrap:9092 --topic dc1-cluster.demo.footopic --from-beginning
```
