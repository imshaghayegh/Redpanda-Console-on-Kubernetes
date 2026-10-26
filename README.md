# Redpanda Console on Kubernetes — Connecting to Strimzi Kafka

این مخزن نحوه نصب **Redpanda Console** و اتصال آن به یک کلاستر **Apache Kafka مدیریت‌شده با Strimzi** را توضیح می‌دهد. پنل وب **بدون صفحه ورود (بدون Username/Password)** اجرا می‌شود، ولی اتصال Console به Kafka از **mTLS** و در صورت فعال بودن Authorization از **ACL** استفاده می‌کند.

> ⚠️ **محیط آزمایشی:** چون خود پنل وب Login ندارد، `NodePort` را با Firewall / ACL شبکه فقط برای IPهای مجاز باز کنید. این تنظیمات را بدون افزودن کنترل دسترسی مناسب به اینترنت یا شبکه عمومی متصل نکنید.

## مشخصات محیط نمونه

| مورد | مقدار |
| --- | --- |
| Kubernetes namespace | `kafka` |
| Kafka CR (Strimzi) | `prod-kafka` |
| Kafka bootstrap | `prod-kafka-kafka-bootstrap.kafka.svc.cluster.local:9093` |
| Kafka authentication | TLS client certificate (mTLS) |
| Console web port | `8080` |
| Console NodePort | `30080` |
| KafkaUser | `redpanda-console` |
| Image | مسیر Registry داخلی، قابل تغییر در Manifest |

## ساختار مخزن

```text
.
├── README.md
└── k8s/
    ├── 01-kafka-user.yaml
    ├── 02-console-configmap.yaml
    └── 03-console-deployment-service.yaml
```

## ۱. پیش‌نیازها

- کلاستر Kafka و Strimzi Operator نصب و آماده باشند.
- در `Kafka` CR، Listener روی پورت `9093` با `tls: true` و `authentication.type: tls` تعریف شده باشد.
- در Kafka CR بخش `entityOperator.userOperator: {}` فعال باشد و User Operator سالم اجرا شود.
- برای کاربران دارای ACL، `spec.kafka.authorization.type: simple` نیز فعال باشد.
- Image Console از Registry داخلی برای نودهای Kubernetes قابل Pull باشد. مسیر نمونه `registry.internal.example/redpandadata/console:v3.13.1` را در Manifest به مسیر **واقعی و در دسترس** خود تغییر دهید و وجود Tag را بررسی کنید.

```bash
kubectl get kafka prod-kafka -n kafka
kubectl get pods -n kafka
kubectl get kafka prod-kafka -n kafka -o jsonpath='{.spec.kafka.listeners}{"\n"}{.spec.kafka.authorization}{"\n"}'
```

**تفاوت دو نوع دسترسی:** `KafkaUser` یک هویت ماشینی برای دسترسی Console به Kafka است. برای ورود به **صفحه وب Console** نام کاربری یا گذرواژه تعریف نمی‌کنیم. ساخت KafkaUser به معنی فعال‌کردن Login در پنل وب نیست.

## ۲. ساخت KafkaUser و Secretهای TLS

فایل [`k8s/01-kafka-user.yaml`](k8s/01-kafka-user.yaml) را بررسی کنید. نمونه با `authentication: tls` و ACLهای صرفاً خواندنی نوشته شده است. **اگر کلاستر شما `authorization: simple` ندارد، بخش `authorization` را از KafkaUser حذف کنید**؛ اما برای امنیت مناسب، بهتر است Authorization در کلاستر فعال باشد.

```bash
kubectl apply -f k8s/01-kafka-user.yaml
kubectl wait -n kafka kafkauser/redpanda-console --for=condition=Ready --timeout=180s
kubectl get kafkauser redpanda-console -n kafka
kubectl get secret redpanda-console -n kafka
kubectl get secret prod-kafka-cluster-ca-cert -n kafka
```

وقتی User Operator کاربر TLS را پردازش کند، معمولاً Secret با نام `redpanda-console` و کلیدهای `user.crt` و `user.key` تولید می‌شود. Secret دیگر (`prod-kafka-cluster-ca-cert`) حاوی `ca.crt` برای اعتبارسنجی سرور Kafka است. هیچ کلید خصوصی، خروجی `-o yaml` Secret یا محتوای Secret را در Git Commit نکنید.

نمایش **صرفاً نام کلیدها**:

```bash
kubectl get secret redpanda-console -n kafka \
  -o go-template='{{range $k,$v := .data}}{{printf "%s\n" $k}}{{end}}'
```

### اگر KafkaUser به Ready نرسید

```bash
kubectl get kafkauser redpanda-console -n kafka \
  -o jsonpath='{range .status.conditions[*]}{.reason}{" : "}{.message}{"\n"}{end}'
kubectl logs -n kafka deployment/prod-kafka-entity-operator -c user-operator --tail=150
kubectl get kafka prod-kafka -n kafka
kubectl get pods -n kafka -o wide
```

- `ScramShaCredentialsCache is not ready!`: ابتدا سلامت **Broker/Controller** و ارتباط User Operator با Kafka را بررسی کنید؛ این خطا می‌تواند با آماده‌نبودن Kafka مرتبط باشد.
- `ImagePullBackOff` روی Broker/Controller: ابتدا Imageهای Kafka و دسترسی به Registry داخلی را اصلاح کنید. Imageهای Kafka را در `spec.kafka.image` از **Kafka CR** تنظیم کنید، نه در Deployment کنسول.
- `InvalidResourceException` یا خطای Authorization: بین `KafkaUser.spec.authorization` و `Kafka.spec.kafka.authorization` سازگاری برقرار کنید.
- `topic-operator` در CrashLoop است: این کانتینر با `user-operator` متفاوت است؛ لاگ هر دو را جداگانه بررسی کنید.

## ۳. تنظیم اتصال Console به Kafka

فایل [`k8s/02-console-configmap.yaml`](k8s/02-console-configmap.yaml) تنظیمات اتصال TLS و آدرس Bootstrap را نگه می‌دارد:

```bash
kubectl apply -f k8s/02-console-configmap.yaml
```

مسیرهای `/etc/console/certs/ca.crt`، `/etc/console/certs/user.crt` و `/etc/console/certs/user.key` در Deployment از Secretهای Kubernetes Mount می‌شوند. نیازی به نوشتن Username/Password در ConfigMap نیست.

## ۴. Deployment و NodePort

در [`k8s/03-console-deployment-service.yaml`](k8s/03-console-deployment-service.yaml) آدرس Image را مطابق Registry داخلی خود اصلاح کنید؛ مقدار `registry.internal.example/...` فقط **Placeholder** است.

```bash
kubectl apply -f k8s/03-console-deployment-service.yaml
kubectl rollout status -n kafka deployment/redpanda-console --timeout=180s
kubectl get deployment,pods,svc -n kafka | grep redpanda
kubectl logs -n kafka deployment/redpanda-console --tail=100
```

IP یکی از نودهای در دسترس را پیدا کنید:

```bash
kubectl get nodes -o wide
```

سپس در مرورگر:

```text
http://<NODE-IP>:30080
```

برای تست جایگزین بدون NodePort:

```bash
kubectl port-forward -n kafka svc/redpanda-console 8080:8080
# http://localhost:8080
```

## ۵. اعتبارسنجی اتصال

```bash
kubectl get pods -n kafka
kubectl get endpointslice -n kafka -l kubernetes.io/service-name=redpanda-console
kubectl logs -n kafka deployment/redpanda-console --tail=150
kubectl get kafkauser redpanda-console -n kafka
```

در پنل Console بررسی کنید که Topicها و Consumer Groupها قابل مشاهده‌اند. تنظیم ACLهای نمونه فقط برای مشاهده است؛ عملیات ایجاد/حذف/نوشتن پیام ممکن است با `TopicAuthorizationException` رد شوند.

## ۶. عیب‌یابی رایج

| نشانه | بررسی |
| --- | --- |
| Console: `CreateContainerConfigError` | نام Secretها، کلیدهای `ca.crt`، `user.crt`، `user.key` و ConfigMap |
| Console: `ImagePullBackOff` | مسیر واقعی Image، Tag، DNS، دسترسی Registry یا `imagePullSecrets` |
| Kafka: `ImagePullBackOff` | `Kafka.spec.kafka.image`، سلامت Registry و Operator |
| Console: `TLS handshake` / `certificate` | فعال‌بودن mTLS روی Listener 9093 و Certificateهای Mount شده |
| Console: `TopicAuthorizationException` | ACLهای KafkaUser و فعال‌بودن `authorization: simple` |
| Console: `connection refused` / `timeout` | `prod-kafka-kafka-bootstrap`، Ready بودن Kafka، NetworkPolicy |
| KafkaUser: `NotReady` | `status.conditions` و لاگ کانتینر `user-operator` |
| NodePort باز نمی‌شود | پورت `30080`، Firewall، kube-proxy و Endpointهای Service |

دستور بررسی Eventها:

```bash
kubectl get events -n kafka --sort-by=.lastTimestamp | tail -40
```

## ۷. نکات عملیاتی و امنیتی

- این پنل **بدون احراز هویت وب** است؛ محدودیت شبکه برای NodePort ضروری است.
- فایل‌های Certificate و Key را در Git نگه ندارید؛ از Kubernetes Secret استفاده کنید.
- Registry خصوصی ممکن است به `imagePullSecrets` احتیاج داشته باشد.
- نسخه Tag Image را در محیط خود تأیید کنید؛ از `latest` برای استقرار تکرارپذیر استفاده نکنید.
- تغییر ConfigMapی که با `subPath` Mount شده، معمولاً نیازمند Restart کردن Deployment برای اعمال تغییرات است:

```bash
kubectl rollout restart deployment/redpanda-console -n kafka
```

## مراجع

- [Redpanda Console: Configure Console](https://docs.redpanda.com/streaming/current/console/config/configure-console/)
- [Redpanda Console: Kubernetes Installation](https://github.com/redpanda-data/console/blob/master/docs/installation.md)
- [Strimzi: Configure Kafka Users](https://github.com/strimzi/strimzi-kafka-operator/blob/main/documentation/modules/operators/proc-configuring-kafka-user.adoc)
