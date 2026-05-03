# Практическое занятие №16

## Рузин Иван Александрович ЭФМО-01-25

### Деплой контейнеризированного приложения в Kubernetes через минимальные манифесты

## 1. Kubernetes стенд и доступ kubectl

Использованный стенд: `minikube`.

Проверка доступа:

```bash
kubectl get nodes
```

![get nodes](docs/get_nodes.png)

## 2. Как образы попали в кластер

Собраны локальные docker-образы с фиксированным тегом `0.1`:

```bash
docker build -t techip-auth:0.1 -f services/auth/Dockerfile .
docker build -t techip-tasks:0.1 -f services/tasks/Dockerfile .
```

![docker build](docs/docker_build.png)

Образы загружены в minikube без внешнего registry:

```bash
minikube image load techip-auth:0.1
minikube image load techip-tasks:0.1
minikube image ls | grep techip
```

![image load](docs/image_load.png)

## 3. Манифесты

Файлы находятся в `deploy/k8s/`:

- `namespace.yaml`
- `postgres.yaml`
- `auth.yaml`
- `tasks-configmap.yaml`
- `tasks-secret.yaml`
- `tasks-deployment.yaml`
- `tasks-service.yaml`
- `kustomization.yaml`

### 3.1 Namespace

Все ресурсы разворачиваются в namespace:

```text
tip2-pr16
```

### 3.2 PostgreSQL

PostgreSQL поднимается внутри Kubernetes, чтобы приложение запускалось так же
просто, как через Docker Compose.

`postgres.yaml` содержит:

- `Secret` с `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`;
- `PersistentVolumeClaim` для хранения данных;
- `Deployment` с образом `postgres:16-alpine`;
- `Service` `postgres:5432`;
- init-скрипт из `deploy/k8s/postgres-init.sql`.

### 3.3 Auth

`auth.yaml` содержит:

- `ConfigMap` `auth-config`;
- `Deployment` `auth`;
- образ `techip-auth:0.1`;
- порты `8081` и `50051`;
- TCP readiness/liveness probes;
- `Service` `auth`.

### 3.4 ConfigMap tasks

`tasks-config` содержит несекретные параметры:

- `TASKS_PORT=8082`
- `AUTH_GRPC_ADDR=auth:50051`
- `REDIS_ADDRS=`
- `RABBIT_URL=`
- `INSTANCE_ID=tasks-k8s`

Redis и RabbitMQ отключены, потому что для этой практики достаточно запуска
`tasks`, `auth` и PostgreSQL.

### 3.5 Secret tasks

`tasks-secret` содержит строку подключения к БД:

```text
TASKS_DB_DSN=postgres://tasks:tasks@postgres:5432/tasksdb?sslmode=disable
```

Это вынесено в `Secret`, потому что DSN содержит пароль.

### 3.6 Deployment tasks

`tasks` Deployment:

- `replicas: 1`
- `image: techip-tasks:0.1`
- `containerPort: 8082`
- env из `tasks-config` и `tasks-secret`
- `readinessProbe`: `GET /health`
- `livenessProbe`: `GET /health`

### 3.7 Service tasks

`tasks` Service:

- тип: `ClusterIP`
- порт: `8082`
- `targetPort: http`

## 4. Применение манифестов и проверка

Применение всех манифестов одной командой:

```bash
kubectl apply -k deploy/k8s
```

![apply](docs/apply.png)

Проверка ресурсов:

```bash
kubectl get pods -n tip2-pr16
kubectl get svc -n tip2-pr16
```

![pods](docs/pods.png)

![svc](docs/svc.png)

Подробная проверка Pod:

```bash
kubectl describe pod -n tip2-pr16 -l app=tasks
kubectl logs -n tip2-pr16 -l app=tasks
```

![describe](docs/describe.png)

## 5. Демонстрация доступа

Port-forward:

```bash
kubectl port-forward -n tip2-pr16 svc/tasks 8082:8082
```

![port-forward](docs/port_forward.png)

Проверка health endpoint:

```bash
curl -i http://127.0.0.1:8082/health
```

Ожидаемый ответ:

```json
{
  "instance_id": "tasks-k8s",
  "service": "tasks",
  "status": "ok"
}
```

![health](docs/health.png)

## 6. Масштабирование

Масштабирование Deployment:

```bash
kubectl scale deployment tasks -n tip2-pr16 --replicas=2
kubectl get pods -n tip2-pr16
```

![scale](docs/scale.png)

Вернуть одну реплику:

```bash
kubectl scale deployment tasks -n tip2-pr16 --replicas=1
```

## 7. Очистка стенда

Удаление ресурсов практики:

```bash
kubectl delete -k deploy/k8s
```

## 8. Контрольные вопросы

### Чем Pod отличается от Deployment?

`Pod` - минимальная единица запуска контейнеров в Kubernetes. `Deployment`
управляет Pod'ами: создает их, перезапускает, обновляет и масштабирует.

### Зачем нужен Service и почему нельзя ходить прямо в Pod?

Pod может пересоздаваться и получать новый IP. `Service` дает стабильное DNS-имя
и постоянную точку доступа к группе Pod'ов.

### Чем readiness probe отличается от liveness probe?

`readinessProbe` показывает, готов ли контейнер принимать трафик.
`livenessProbe` показывает, жив ли процесс; если проверка падает, Kubernetes
перезапускает контейнер.

### Зачем нужен ConfigMap и чем он отличается от Secret?

`ConfigMap` хранит обычную конфигурацию. `Secret` хранит чувствительные данные:
пароли, токены, строки подключения.

### Почему важно использовать теги образов, а не только latest?

Фиксированный тег делает деплой воспроизводимым. `latest` может указывать на
другой образ после новой сборки, из-за чего сложнее понять, какая версия реально
запущена.
