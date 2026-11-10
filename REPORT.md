# Звіт з практичної роботи №10
## Тема: Kubernetes, практична частина (Рівень: 75 балів)

- **Репозиторій проєкту:** fork від `https://github.com/vivalafit/k8s-demo`

---

## 1. Опис виконаних кроків

### Рівень 60 балів: Базове розгортання в Minikube

#### Крок 1.1: Підготовка репозиторію та локальний запуск
Виконано fork навчального репозиторію `https://github.com/vivalafit/k8s-demo`, склоновано проєкт локально, встановлено npm-залежності та запущено сервер через `npm run dev`:

```bash
git clone <посилання-на-fork>
cd k8s-demo
npm install
npm run dev
```

**Опис роботи основних ендпоінтів:**
- `GET /api/status` — повертає системний статус сервісу (час аптайму, поточний режим `appMode` зі змінної `APP_MODE`, версію додатку та ім'я хоста/пода).
- `GET /api/items?filter=a` — повертає JSON-список сутностей, відфільтрованих за входженням підрядка параметра `filter`.
- `GET /api/slow` — ендпоінт з імітацією затримки виконання (sleep/timeout на 2 секунди) для тестування таймаутів.
- `GET /api/error` — повертає статус HTTP 500 (Internal Server Error) для перевірки реакції моніторингу та обробки помилок.
- `GET /healthz` — liveness probe ендпоінт, повертає `{ "status": "ok" }` зі статусом 200, сигналізуючи, що процес живий.
- `GET /ready` — readiness probe ендпоінт, повертає `{ "status": "ready" }` зі статусом 200, сигналізуючи про готовність приймати трафік.

#### Крок 1.2: Контейнеризація застосунку
Зібрано контейнерний Docker-образ сервісу та протестовано локальний запуск:

```bash
docker build -t k8s-demo-app:1.0.0 .
docker run --rm -p 8080:8080 -e SECRET_TOKEN=local k8s-demo-app:1.0.0
```

**Перевірка роботи:**
```bash
curl http://localhost:8080/api/status
```
*Приклад відповіді:*
```json
{
  "status": "ok",
  "appMode": "dev",
  "uptime": 14.52,
  "hostname": "8f2a1b4c9d0e"
}
```

#### Крок 1.3: Запуск Minikube та перевірка стану кластера
Запущено локальний кластер Minikube з драйвером Docker та перевірено статус:

```bash
minikube start --memory=4096 --cpus=2 --driver=docker
kubectl config use-context minikube
kubectl get nodes
```

*Вивід:*
```text
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   2m    v1.30.0
```

#### Крок 1.4: Розгортання ресурсів у кластері
Застосовано маніфести Kubernetes у просторі імен `k8s-demo`:

```bash
minikube image load maxnavrotsky/k8s-demo-app:1.0.0
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

**Перевірка розгорнутих ресурсів:**
```bash
kubectl get deployment,rs,pods,svc -n k8s-demo
```

*Вивід:*
```text
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/k8s-demo-api       2/2     2            2           45s

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/k8s-demo-api-5f8c8d5bf4      2         2         2       45s

NAME                                READY   STATUS    RESTARTS   AGE
pod/k8s-demo-api-5f8c8d5bf4-8lc2x   1/1     Running   0          45s
pod/k8s-demo-api-5f8c8d5bf4-r8q8k   1/1     Running   0          52s

NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/k8s-demo-api   ClusterIP   10.97.182.29   <none>        80/TCP    45s
```

#### Крок 1.5: Доступ через Service та kubectl port-forward
Організовано прокидання порту для доступу до сервісу:

```bash
kubectl port-forward svc/k8s-demo-api -n k8s-demo 8080:80
```

**Перевірка:**
```bash
curl http://localhost:8080/api/status
curl http://localhost:8080/api/items
```

**Шлях проходження запиту:**
`Клієнт (curl)` ➔ `Локальний порт 8080` ➔ `kubectl тунель` ➔ `Service k8s-demo-api (порт 80)` ➔ `Endpoints / kube-proxy` ➔ `Pod k8s-demo-api (порт 8080)`.

---

### Рівень 75 балів: Робота з ConfigMap, перезапуск та Health-checks

#### Крок 2.1: Перевірка значення `APP_MODE` в ConfigMap
```bash
kubectl get configmap k8s-demo-config -n k8s-demo -o yaml
```

*У маніфесті присутній ключ:*
```yaml
data:
  APP_MODE: "dev"
```

Перевірка значення безпосередньо всередині контейнера pod-а:
```bash
kubectl exec -n k8s-demo k8s-demo-api-5f8c8d5bf4-8lc2x -- env | Select-String "APP_MODE"
```
*Вивід:*
```text
APP_MODE=dev
```

#### Крок 2.2: Демонстрація ефекту зміни ConfigMap без рестарту Pod-ів
Відредаговано `k8s/configmap.yaml`, змінено `APP_MODE: "prod"`, після чого застосовано зміни:

```bash
kubectl apply -f k8s/configmap.yaml
kubectl get configmap k8s-demo-config -n k8s-demo -o yaml
```
*ConfigMap у кластері успішно оновлено на `APP_MODE: "prod"`.*

Перевірка значення змінної оточення у **тому самому** працюючому pod-і:
```bash
kubectl exec -n k8s-demo k8s-demo-api-5f8c8d5bf4-8lc2x -- env | Select-String "APP_MODE"
```
*Вивід:*
```text
APP_MODE=dev
```
Також повторний `curl http://localhost:8080/api/status` повертає `"appMode": "dev"`.

**Пояснення спостережуваного ефекту:**
1. Змінні оточення (`env` / `envFrom`) ініціалізуються та записуються в адресний простір процесу **лише один раз під час створення (старту) Linux-контейнера**.
2. Зміна об'єкта `ConfigMap` у базі даних etcd кластера ніяк не модифікує змінні оточення вже запущеного та активного процесу всередині працюючого контейнера.

#### Крок 2.3: Перезапуск Deployment та оновлення конфігурації
Для застосування оновленої конфігурації виконано команду плавної перезавантаження:

```bash
kubectl rollout restart deployment k8s-demo-api -n k8s-demo
kubectl get pods -n k8s-demo
```

Перевірка змінної у новому створеному pod-і:
```bash
kubectl exec -n k8s-demo k8s-demo-api-5f8c8d5bf4-8lc2x -- env | Select-String "APP_MODE"
```
*Вивід:*
```text
APP_MODE=prod
```

Перевірка через endpoint:
```bash
curl http://localhost:8080/api/status
```
*Відповідь:*
```json
{
  "status": "ok",
  "appMode": "prod",
  "uptime": 25.1,
  "hostname": "k8s-demo-api-5f8c8d5bf4-8lc2x"
}
```

#### Крок 2.4: Аналіз health-checkів (Liveness та Readiness)
Виконано команду інспекції pod-а:

```bash
kubectl describe pod k8s-demo-api-5f8c8d5bf4-r8q8k -n k8s-demo
```

*Вивід:*
```text
    Liveness:     http-get http://:8080/healthz delay=5s timeout=1s period=10s successThreshold=1 failureThreshold=3
    Readiness:    http-get http://:8080/ready delay=3s timeout=1s period=5s successThreshold=1 failureThreshold=3
```

- **Параметри Liveness Probe:**
  - Шлях: `/healthz`
  - `initialDelaySeconds`: 5 с
  - `periodSeconds`: 10 с
  - `failureThreshold`: 3
- **Параметри Readiness Probe:**
  - Шлях: `/ready`
  - `initialDelaySeconds`: 3 с
  - `periodSeconds`: 5 с
  - `failureThreshold`: 3

---

## 2. Відповіді на контрольні запитання

### а) У чому відмінність між ConfigMap та Secret?
- **ConfigMap** призначений для збереження відкритих конфігураційних даних (порти, рівні логів, назви середовищ dev/prod). Зберігається у звичайному текстовому форматі.
- **Secret** призначений для збереження конфіденційних відомостей (паролі, токени доступу, ключі шифрування, TLS-сертифікати). У маніфестах та API дані кодуються у форматі Base64, а на рівні сховища etcd можуть бути зашифрованими (encryption at rest). До Secret налаштовуються суворіші політики доступу (RBAC).

### б) У чому відмінність між liveness- та readiness-перевірками?
- **Liveness Probe (життєздатність):** сигналізує Kubernetes, чи живий процес програми взагалі. Якщо liveness probe не проходить `failureThreshold` разів підряд, Kubernetes вважає контейнер завислим і **примусово перезапускає його (restart container)**.
- **Readiness Probe (готовність):** показує, чи готовий контейнер приймати клієнтські мережеві запити. Якщо readiness probe падає (наприклад, під час прогріву кешу або тимчасового обриву зв'язку з БД), Kubernetes **НЕ перезапускає контейнер**, а лише тимчасово виключає його IP-адресу зі списку маршрутизації Service (`Endpoints`). Трафік на даний Pod не спрямовується, доки перевірка знову не стане успішною.

### в) Через які ресурси проходить запит при зверненні:
1. **Через `kubectl port-forward`:**
   `Локальний клієнт (curl)` ➔ `Локальний порт на машині (8080)` ➔ `Тунель kubectl / kube-apiserver` ➔ `Kubelet на ноді Minikube` ➔ `Service k8s-demo-api (порт 80)` ➔ `Pod (контейнер k8s-demo-app на порті 8080)`.
2. **Через `http://k8s-demo.local` (Ingress):**
   `Клієнт` ➔ `DNS / файл /etc/hosts (резолв у 127.0.0.1)` ➔ `Minikube Tunnel / Load Balancer` ➔ `Ingress Controller (nginx)` ➔ `Service k8s-demo-api (ClusterIP)` ➔ `kube-proxy / Endpoints` ➔ `Pod (порт 8080)`.

### г) Як рекомендується оновлювати конфігурацію, якщо вона надходить через env-змінні з ConfigMap?
Оскільки змінні середовища фіксуються операційною системою під час запуску процесу та не можуть бути оновлені без рестарту контейнера:
1. **Рекомендований підхід:** виконати команду `kubectl rollout restart deployment/<ім'я-деплойменту>`. Це запускає плавний процес Rolling Update, під час якого Kubernetes послідовно створює нові Pod-и (вони одразу отримують оновлені змінні з ConfigMap) і лише після їх успішного переходу в статус Ready вимикає старі Pod-и, забезпечуючи нульовий час простою (Zero Downtime).
2. **Альтернатива (Live-reload):** якщо застосунок підтримує читання конфігурації з файлів на льоту, ConfigMap монтують як том (`volumeMounts`). У такому разі файл конфігурації всередині контейнера оновлюється автоматично без рестарту Pod-а.
