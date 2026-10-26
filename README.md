# Redpanda Console on Kubernetes — Connecting to Strimzi Kafka

This repository demonstrates how to deploy **Redpanda Console** on Kubernetes and connect it to an **Apache Kafka cluster managed by Strimzi**. The Console web interface runs **without a username/password login**, while its connection to Kafka uses **mutual TLS (mTLS)** and, when enabled, **Kafka ACLs**.


## Example environment

| Setting | Value |
| --- | --- |
| Kubernetes namespace | `kafka` |
| Strimzi Kafka resource | `prod-kafka` |
| Kafka bootstrap address | `prod-kafka-kafka-bootstrap.kafka.svc.cluster.local:9093` |
| Kafka authentication | TLS client certificate (mTLS) |
| Console container port | `8080` |
| Console NodePort | `30080` |
| KafkaUser | `redpanda-console` |
| Container image | Configurable private-registry image |

## Repository structure

```text
.
├── README.md
└── k8s/
    ├── 01-kafka-user.yaml
    ├── 02-console-configmap.yaml
    └── 03-console-deployment-service.yaml
```

## 1. Prerequisites

- A running Kubernetes cluster with the Strimzi Kafka Operator installed.
- A healthy Kafka cluster named `prod-kafka` in the `kafka` namespace.
- A Kafka listener on port `9093` configured with `tls: true` and `authentication.type: tls`.
- `entityOperator.userOperator: {}` enabled in the Kafka resource, with the User Operator running successfully.
- If you want ACL enforcement, `spec.kafka.authorization.type: simple` enabled in the Kafka resource.
- A Redpanda Console container image accessible from your Kubernetes nodes. The deployment manifest uses the **placeholder** `registry.internal.example/redpandadata/console:v3.13.1`; replace it with an actual image path and verified tag in your private registry.

Verify the existing Kafka installation:

```bash
kubectl get kafka prod-kafka -n kafka
kubectl get pods -n kafka
kubectl get kafka prod-kafka -n kafka \
  -o jsonpath='{.spec.kafka.listeners}{"\n"}{.spec.kafka.authorization}{"\n"}'
```

**Important distinction:** `KafkaUser` is a machine identity used by Redpanda Console when communicating with Kafka. It does **not** enable username/password authentication for users who open the Console web page.

## 2. Create the KafkaUser and TLS secrets

Review [`k8s/01-kafka-user.yaml`](k8s/01-kafka-user.yaml). The example uses TLS client authentication and read-oriented ACLs.

**If your Kafka cluster does not have `authorization: simple` enabled, remove the `authorization` section from the KafkaUser manifest.** For secure environments, Kafka authorization should remain enabled and appropriate ACLs should be configured.

Apply the KafkaUser resource:

```bash
kubectl apply -f k8s/01-kafka-user.yaml
kubectl wait -n kafka kafkauser/redpanda-console \
  --for=condition=Ready --timeout=180s
kubectl get kafkauser redpanda-console -n kafka
```

Once the User Operator successfully reconciles the TLS user, Strimzi creates a Kubernetes Secret named `redpanda-console`, typically containing `user.crt` and `user.key`. The `prod-kafka-cluster-ca-cert` Secret contains the cluster CA certificate (`ca.crt`) used to verify Kafka's TLS identity.

Verify both secrets:

```bash
kubectl get secret redpanda-console -n kafka
kubectl get secret prod-kafka-cluster-ca-cert -n kafka
```

List **only the key names**, without displaying secret values:

```bash
kubectl get secret redpanda-console -n kafka \
  -o go-template='{{range $k,$v := .data}}{{printf "%s\n" $k}}{{end}}'
```

## 3. Configure the Console-to-Kafka connection

The file [`k8s/02-console-configmap.yaml`](k8s/02-console-configmap.yaml) defines the Kafka bootstrap address and mTLS certificate paths.

```bash
kubectl apply -f k8s/02-console-configmap.yaml
```

The deployment mounts the following files from Kubernetes Secrets:

| Mounted file | Source |
| --- | --- |
| `/etc/console/certs/ca.crt` | `prod-kafka-cluster-ca-cert` Secret |
| `/etc/console/certs/user.crt` | `redpanda-console` Secret |
| `/etc/console/certs/user.key` | `redpanda-console` Secret |


## 4. Deploy Redpanda Console and expose it via NodePort

Edit [`k8s/03-console-deployment-service.yaml`](k8s/03-console-deployment-service.yaml) 

```yaml
image: registry.internal.example/redpandadata/console:v3.13.1
```

Apply the deployment and service:

```bash
kubectl apply -f k8s/03-console-deployment-service.yaml
kubectl rollout status -n kafka deployment/redpanda-console --timeout=180s
kubectl get deployment,pods,svc -n kafka | grep redpanda
kubectl logs -n kafka deployment/redpanda-console --tail=100
```


```text
http://<NODE-IP>:30080
```

## 5. Verify connectivity

```bash
kubectl get pods -n kafka
kubectl get endpointslice -n kafka \
  -l kubernetes.io/service-name=redpanda-console
kubectl logs -n kafka deployment/redpanda-console --tail=150
kubectl get kafkauser redpanda-console -n kafka
```

In Redpanda Console, verify that you can browse Kafka topics and consumer groups. The sample ACLs are oriented toward read-only inspection; operations such as creating or deleting topics, producing messages, or changing topic settings can fail with an authorization error. Additional Console features may require additional permissions.

## References

- [Redpanda Console — Configuration](https://docs.redpanda.com/streaming/current/console/config/configure-console/)
- [Redpanda Console — GitHub](https://github.com/redpanda-data/console)
- [Strimzi Kafka Operator — Documentation](https://strimzi.io/documentation/)
