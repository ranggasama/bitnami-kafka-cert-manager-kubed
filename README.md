# Bitnami - Kafka Kubernetes Installation

## Pre-requisites

### Add repo bitnami

```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### Config Syncer (previously Kubed) operator

[Kubed Installation Guide](https://appscode.com/products/kubed/v0.12.0/setup/install/)

### Cert-Manager

Prepare certificate using Cert-Manager

```shell
kubectl apply -f cert/bootstrap-ca.yaml
kubectl apply -f cert/certificates.yaml
```

### Prepare Config syncer

```shell
# Create namespace kafka and label it "need=root-secret"
kubectl create ns kafka
kubectl label namespace kafka need=root-secret

# annotate root-secret so that it replicate to namespace with label "need=root-secret"
kubectl -n cert-manager annotate secrets root-secret kubed.appscode.com/sync="need=root-secret" --overwrite
```

## Installation

```shell
helm install -n kafka kafka bitnami/kafka \
--version 16.2.10 \
-f values-custom.yaml
```

## Validate Installation

```shell
# Create kafka client pod and execute shell
kubectl apply -f kafka-client.yaml
kubectl -n kafka exec -it kafka-client -- bash

# produce queue message to "test" topic:
kafka-console-producer.sh \
  --producer.config /tmp/client.properties \
  --broker-list kafka-0.kafka-headless.kafka.svc.cluster.local:9092,kafka-1.kafka-headless.kafka.svc.cluster.local:9092,kafka-2.kafka-headless.kafka.svc.cluster.local:9092 \
  --topic test

# read queue message from "test" topic::
kafka-console-consumer.sh \
    --consumer.config /tmp/client.properties \
    --bootstrap-server kafka.kafka.svc.cluster.local:9092 \
    --topic test \
    --from-beginning

# List topics:
kafka-topics.sh \
  --command-config /tmp/client.properties \
  --bootstrap-server kafka-0.kafka-headless.kafka.svc.cluster.local:9092,kafka-1.kafka-headless.kafka.svc.cluster.local:9092,kafka-2.kafka-headless.kafka.svc.cluster.local:9092 --list
```

## Access kafka via UI

Run deploment using container image "provectuslabs/kafka-ui", with environment properties below and mounting root-secret secrets to "/certs" directory (with mode 444)

```properties
KAFKA_CLUSTERS_0_ZOOKEEPER=kafka-zookeeper-0.kafka-zookeeper-headless:2181,kafka-zookeeper-1.kafka-zookeeper-headless:2181,kafka-zookeeper-2.kafka-zookeeper-headless:2181
KAFKA_CLUSTERS_0_PROPERTIES_SSL_TRUSTSTORE_TYPE=PEM
KAFKA_CLUSTERS_0_PROPERTIES_SSL_TRUSTSTORE_LOCATION=/certs/ca.crt
KAFKA_CLUSTERS_0_PROPERTIES_SECURITY_PROTOCOL=SSL
KAFKA_CLUSTERS_0_PROPERTIES_CLIENT_DNS_LOOKUP=use_all_dns_ips
KAFKA_CLUSTERS_0_NAME=local
KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=kafka-0.kafka-headless:9092,kafka-1.kafka-headless:9092,kafka-2.kafka-headless:9092
```

## Clean-up

```shell
helm -n kafka delete kafka
kubectl delete pvc -n kafka --all
kubectl delete pv $(kubectl get pv | tail -n+2 | awk '$5 == "Released" {print $1}')
# Delete leftover folder from NFS server
ssh root@<NAS IP> "rm -rf /nfs/kafka-data-kafka-*"
```
