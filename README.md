## Deploying Apache Kafka on Kubernetes with Strimzi

![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.34+-326CE5?logo=kubernetes&logoColor=white)
![Kafka](https://img.shields.io/badge/Apache_Kafka-4.x-231F20?logo=apachekafka&logoColor=white)
![Strimzi](https://img.shields.io/badge/Strimzi-1.1.0-orange)
![Kind](https://img.shields.io/badge/Kind-Local_Cluster-0094F5)
![License](https://img.shields.io/badge/License-MIT-green)

This guide demonstrates how to deploy an **Apache Kafka cluster** on a local Kubernetes environment using **Kind** and the **Strimzi Cluster Operator**.

The deployment uses **Kafka Node Pools** to separate controller and broker roles, making it closer to a production-style architecture while remaining lightweight for local development and testing.

---

### Architecture

<img width="738" height="383" alt="kafka excalidraw" src="https://github.com/user-attachments/assets/e0328c8c-db08-42a0-a3fb-57e60bf43da5" />
---

### Prerequisites

Ensure the following tools are installed:

- Docker
- Kind
- kubectl

Verify the installation:

```bash
docker --version
kind --version
kubectl version --client
```

---

### Create the Kind Cluster

Create a Kubernetes cluster using the provided configuration.

```bash
kind create cluster --config cluster-config.yaml --name example
```

Verify the cluster is running:

```bash
kubectl get nodes -o wide
```

Expected output:

- 1 Control Plane
- 2 Worker Nodes

---

### Install the Strimzi Cluster Operator

Create the Kafka namespace.

```bash
kubectl create namespace kafka
```

Install the Strimzi Cluster Operator.

```bash
curl -L https://github.com/strimzi/strimzi-kafka-operator/releases/download/1.1.0/strimzi-cluster-operator-1.1.0.yaml \
| sed 's/namespace: myproject/namespace: kafka/g' \
| kubectl create -f - -n kafka
```

Wait until the operator is available.

```bash
kubectl wait deployment/strimzi-cluster-operator \
    -n kafka \
    --for=condition=Available \
    --timeout=180s
```

---

### Deploy the Kafka Cluster

Deploy the Kafka controller node pool.

```bash
kubectl apply -f kafka-controller.yaml -n kafka
```

Deploy the Kafka broker node pool.

```bash
kubectl apply -f kafka-broker.yaml -n kafka
```

Deploy the Kafka cluster.

```bash
kubectl apply -f kafka-cluster.yaml -n kafka
```

Wait for Kafka to become ready.

```bash
kubectl wait kafka/demo \
    --for=condition=Ready \
    --timeout=300s \
    -n kafka
```

---

### Validate the Deployment

Verify the Kafka node pools.

```bash
kubectl get kafkanodepool -n kafka
```

View the running pods.

```bash
kubectl get pods -n kafka -o wide
```

Check the persistent volumes.

```bash
kubectl get pvc -n kafka
```

---

### Create a Kafka Topic

Deploy the topic resource.

```bash
kubectl apply -f kafka-topic.yaml -n kafka
```

Verify the topic.

```bash
kubectl get kafkatopic -n kafka
```

---

### Test the Kafka Cluster

#### Start a Producer

```bash
kubectl -n kafka run kafka-producer \
    -ti \
    --image=quay.io/strimzi/kafka:1.1.0-kafka-4.3.0 \
    --rm=true \
    --restart=Never \
    -- \
    bin/kafka-console-producer.sh \
    --bootstrap-server demo-kafka-bootstrap:9092 \
    --topic test
```

Type a few messages and press **Enter** after each one.

Example:

```
Hello Kafka
Testing Strimzi
Kubernetes Rocks
```

---

### Start a Consumer

Open another terminal and run:

```bash
kubectl -n kafka run kafka-consumer \
    -ti \
    --image=quay.io/strimzi/kafka:1.1.0-kafka-4.3.0 \
    --rm=true \
    --restart=Never \
    -- \
    bin/kafka-console-consumer.sh \
    --bootstrap-server demo-kafka-bootstrap:9092 \
    --topic test \
    --from-beginning
```

You should see the messages produced from the previous step.

---

#### Cleanup

Delete the Kafka resources.

```bash
kubectl -n kafka delete $(kubectl get strimzi -o name -n kafka)
```

Verify that the persistent volume claims have been removed.

```bash
kubectl get pvc -n kafka
```

Delete the Kind cluster.

```bash
kind delete cluster --name example
```

---

#### Troubleshooting

Check the Strimzi operator logs.

```bash
kubectl logs deployment/strimzi-cluster-operator -n kafka
```

Describe the Kafka custom resource.

```bash
kubectl describe kafka demo -n kafka
```

View all Kafka-related resources.

```bash
kubectl get all -n kafka
```

Check Kubernetes events.

```bash
kubectl get events -n kafka --sort-by=.metadata.creationTimestamp
```
