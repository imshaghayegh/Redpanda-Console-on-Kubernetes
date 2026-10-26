# Redpanda Console on Kubernetes — Connecting to Strimzi Kafka

This repository demonstrates how to deploy **Redpanda Console** on Kubernetes and connect it to an **Apache Kafka cluster managed by Strimzi**. The Console web interface runs **without a username/password login**, while its connection to Kafka uses **mutual TLS (mTLS)** and, when enabled, **Kafka ACLs**.

> **Lab environment / security warning:** Because the web UI has no authentication, restrict access to its NodePort using a firewall or network controls. Never expose this setup directly to the public internet.

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

Never commit private keys, secret contents, or decoded certificates containing private keys to Git.

### If the KafkaUser does not become Ready

```bash
kubectl get kafkauser redpanda-console -n kafka \
  -o jsonpath='{range .status.conditions[*]}{.reason}{" : "}{.message}{"\n"}{end}'

kubectl logs -n kafka deployment/prod-kafka-entity-operator \
  -c user-operator --tail=150

kubectl get kafka prod-kafka -n kafka
kubectl get pods -n kafka -o wide
```

Common causes include:

- **`ScramShaCredentialsCache is not ready!`** — Check whether the Kafka brokers and controllers are Ready and whether the User Operator can reach Kafka. This error may occur while the operator's Kafka-related cache is not initialized.
- **`ImagePullBackOff` on brokers/controllers** — Fix Kafka image availability and private-registry access first. The Kafka image belongs in `Kafka.spec.kafka.image`, not in the Console deployment.
- **`InvalidResourceException` / authorization errors** — Confirm that the KafkaUser authorization configuration is compatible with the Kafka cluster's authorization configuration.
- **Topic Operator crash loops** — The Topic Operator and User Operator are separate containers. Inspect their logs separately; a Topic Operator failure does not by itself prove the User Operator is failing.

Do not manually create the KafkaUser TLS Secret unless you are deliberately replacing the Strimzi-managed certificate workflow.

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

No SASL username or password is required because Kafka authentication uses mTLS. No web login is configured for the Console.

## 4. Deploy Redpanda Console and expose it via NodePort

Edit [`k8s/03-console-deployment-service.yaml`](k8s/03-console-deployment-service.yaml) and **replace the example image reference** with the path and tag of an image that actually exists in your internal registry:

```yaml
image: registry.internal.example/redpandadata/console:v3.13.1
```

For example, use your organization's verified private-registry path; do not assume a pull-through proxy supports arbitrary paths or image tags. If the registry requires credentials, configure an appropriate Kubernetes `imagePullSecret` for the Pod or its ServiceAccount.

Apply the deployment and service:

```bash
kubectl apply -f k8s/03-console-deployment-service.yaml
kubectl rollout status -n kafka deployment/redpanda-console --timeout=180s
kubectl get deployment,pods,svc -n kafka | grep redpanda
kubectl logs -n kafka deployment/redpanda-console --tail=100
```

The Service uses `type: NodePort` with port `30080`. Find the IP address of a reachable Kubernetes node:

```bash
kubectl get nodes -o wide
```

Then open:

```text
http://<NODE-IP>:30080
```

For local testing without exposing the NodePort, you can instead use port forwarding:

```bash
kubectl port-forward -n kafka svc/redpanda-console 8080:8080
# Open http://localhost:8080
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

## 6. Troubleshooting

| Symptom | What to check |
| --- | --- |
| Console: `CreateContainerConfigError` | ConfigMap and Secret names; presence of `ca.crt`, `user.crt`, and `user.key` keys |
| Console: `ImagePullBackOff` | Actual image path and tag, registry reachability, TLS trust, and `imagePullSecrets` |
| Kafka: `ImagePullBackOff` | `Kafka.spec.kafka.image`, Kafka operator reconciliation, and private-registry availability |
| Console: TLS handshake / certificate errors | Kafka listener authentication, mounted certificates, and CA trust |
| Console: `TopicAuthorizationException` | KafkaUser ACLs and Kafka's authorization configuration |
| Console: connection refused / timeout | Kafka bootstrap Service, broker readiness, DNS, and NetworkPolicies |
| KafkaUser: `NotReady` | `status.conditions` and `user-operator` container logs |
| NodePort not reachable | Port `30080`, firewall rules, Kubernetes networking, and Service endpoints |

Inspect recent events:

```bash
kubectl get events -n kafka --sort-by=.lastTimestamp | tail -40
```

## 7. Operations and security notes

- **The web UI is unauthenticated.** Restrict NodePort access to trusted IP ranges, preferably within an isolated lab network.
- Keep private keys in Kubernetes Secrets; never commit them to Git.
- Internal registries may require `imagePullSecrets` or trusted CA configuration on the Kubernetes nodes.
- Verify image version compatibility and pin a known tag instead of using `latest`.
- Because the ConfigMap is mounted using `subPath`, updating it typically requires restarting the Console deployment:

```bash
kubectl rollout restart deployment/redpanda-console -n kafka
```

- If the KafkaUser is `NotReady`, resolve Kafka or User Operator issues **before** deploying the Console; a missing Secret will prevent the Console Pod from starting correctly.

## References

- [Redpanda Console — Configuration](https://docs.redpanda.com/streaming/current/console/config/configure-console/)
- [Redpanda Console — GitHub](https://github.com/redpanda-data/console)
- [Strimzi Kafka Operator — Documentation](https://strimzi.io/documentation/)
