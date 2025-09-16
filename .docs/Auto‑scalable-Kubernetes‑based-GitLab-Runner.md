# Auto‑scalable Kubernetes‑based GitLab Runner

## Быстрый BugFix
- При возникновении проблем читай [gitlab runner quick fix](.docs%2Fgitlab-runner-fix.md) 
- Если там не нашел, смотри **Дополнительная документация** [index.md](index.md)

## Дополнительная документация
- Индекс документации проекта (краткие аннотации) -> [index.md](index.md)
- Больше документации k8s в kubernetes.io -> [kubernetes.io > docs](https://kubernetes.io/docs/home/)

## Кратко
Инфраструктурный шаблон для запуска GitLab Runner с **Kubernetes executor**. Каждая CI‑job выполняется в отдельном Pod’е, масштабирование происходит автоматически по мере появления задач. Поддерживается кэш/артефакты через S3‑совместимые хранилища (MinIO), сборка образов через `docker:dind` или бездемонные билдеры. Проект ориентирован на изоляцию окружений, воспроизводимость и эксплуатационную простоту.

## Ключевые возможности
- **Автомасштабирование CI‑нагрузки**: одна job — один Pod в `ci-jobs`.
- **Чистая изоляция**: build‑контейнеры и сервисы (dind и т.п.) запускаются только на время job.
- **Минимальные права**: отдельно вынесенный ServiceAccount и точечный RBAC в namespace `ci-jobs`.
- **Интеграция с реестрами**: поддержка приватных реестров с пользовательскими CA (секреты для `/etc/docker/certs.d` и `/etc/containers/certs.d`).
- **Кэш GitLab**: S3/MinIO с отдельным пользователем и приватным bucket’ом.
- **Наблюдаемость**: метрики самого Runner’а; совместимость с Prometheus.

## Архитектура
- **Namespaces**:  
  `ci-system` — сам GitLab Runner (Deployment, ServiceAccount).  
  `ci-jobs` — эфемерные Pod’ы job’ов с сервисными монтированиями секретов.
- **RBAC**: роль в `ci-jobs` с правами на `pods`, `pods/log`, `pods/exec`, `secrets`, `configmaps`, `services`, `events` и привязка к `ci-system:gitlab-runner`.
- **Секреты CA**:
    - `gitlab-ca` для доверия GitLab серверу (монтируется в `/etc/gitlab-runner/certs`).
    - Пары секретов для реестров: `reg249-ca-docker`/`reg249-ca-containers`, `reg94-ca-docker`/`reg94-ca-containers`.
- **Кэш**: MinIO (пользователь `gitlab-runner`, bucket `gitlab-runner-cache`, минимальная политика).

## Требования
- Kubernetes‑кластер (совместимые версии kubelet/Runner).
- Доступ к GitLab (HTTP(S)), приватным реестрам и MinIO/S3.
- Для сборки через `docker:dind` — разрешён `privileged` для job‑Pod’ов.
- SSL‑материалы и `values.yaml` (см. структуру ниже).

```
.
├── SSL
│   ├── gitlab-ca
│   │   └── 10.174.18.94.crt
│   ├── reg249-ca
│   │   └── ca.crt
│   └── reg94-ca
│       └── ca.crt
└── values.yaml
```

---

## Пример установки (Helm) с пошаговым описанием

### 0) Подготовка неймспейсов
```bash
kubectl create namespace ci-system
kubectl create namespace ci-jobs
```

### 1) ServiceAccount и права (RBAC)
> SA создаём в `ci-system`; права — в `ci-jobs`, где будут жить Pod’ы job’ов.

```bash
# ServiceAccount для Runner’а
kubectl -n ci-system create serviceaccount gitlab-runner

# Роль с минимальными правами для управления job‑ресурсами
kubectl -n ci-jobs create role gitlab-runner-manager   --verb=get,list,watch,create,update,patch,delete   --resource=pods,pods/log,pods/exec,secrets,configmaps,services,events

# Привязка роли к SA из ci-system
kubectl -n ci-jobs create rolebinding gitlab-runner-manager-binding   --role=gitlab-runner-manager   --serviceaccount=ci-system:gitlab-runner
```

### 2) Секреты с пользовательскими CA (во всех нужных неймспейсах)

> **Важно:** поды Runner’а читают `gitlab-ca` в `ci-system`, а job‑Pod’ы — секреты в `ci-jobs`. Секреты создаём в обоих NS, имена секретов **разные** для разных целей (Docker vs containers), чтобы их можно было монтировать независимо.

#### 2.1. Доверие к GitLab (секрет `gitlab-ca`)
```bash
# Runner (ci-system)
kubectl -n ci-system create secret generic gitlab-ca   --from-file=10.174.18.94.crt=./SSL/gitlab-ca/10.174.18.94.crt

# Job‑поды (ci-jobs)
kubectl -n ci-jobs create secret generic gitlab-ca   --from-file=10.174.18.94.crt=./SSL/gitlab-ca/10.174.18.94.crt
```

#### 2.2. Доверие к приватным реестрам (секреты для Docker/containers)
```bash
# 10.174.18.249:5000
kubectl -n ci-jobs create secret generic reg249-ca-docker   --from-file=ca.crt=./SSL/reg249-ca/ca.crt
kubectl -n ci-jobs create secret generic reg249-ca-containers   --from-file=ca.crt=./SSL/reg249-ca/ca.crt

# 10.174.18.94:5555
kubectl -n ci-jobs create secret generic reg94-ca-docker   --from-file=ca.crt=./SSL/reg94-ca/ca.crt
kubectl -n ci-jobs create secret generic reg94-ca-containers   --from-file=ca.crt=./SSL/reg94-ca/ca.crt
```

> Альтернатива через копирование существующего секрета (если уже есть `reg94-ca`/`reg249-ca`):  
> `kubectl -n ci-jobs get secret reg94-ca -o json | jq '.metadata.name="reg94-ca-docker" | del(.metadata.uid,.metadata.resourceVersion,.metadata.creationTimestamp,.metadata.namespace,.metadata.annotations,.metadata.managedFields)' | kubectl -n ci-jobs apply -f -`

### 3) Helm‑установка Runner’а
```bash
helm repo add gitlab https://charts.gitlab.io
helm repo update
```

Подготовьте `values.yaml` (пример из вашего окружения):

```yaml
gitlabUrl: "https://<gitlab.company.com ip>:8929/"
runnerRegistrationToken: "*******************"
unregisterRunners: true
replicas: 2

# чарт сам смонтирует секрет в /etc/gitlab-runner/certs
certsSecretName: gitlab-ca

rbac:
  create: true
  clusterWideAccess: true
serviceAccount:
  create: true
  name: gitlab-runner

runners:
  name: "k8s-autoscaling-runner"
  tags: "k8s,autoscaling"
  runUntagged: true
  requestConcurrency: 10
  config: |
    concurrent = 10
    check_interval = 3

    [[runners]]
      name = "k8s-autoscaling-runner"
      url = "https://<gitlab.company.com ip>:8929/"
      tls_server_name = "gitlab.company.com"
      executor = "kubernetes"
      environment = ["GIT_SSL_NO_VERIFY=true","GIT_SUBMODULE_STRATEGY=recursive", "DOCKER_AUTH_CONFIG={\"auths\": {\"<registry.company.com ip>:5000\": {\"auth\": \"***************************==\"}}}"]

      [runners.kubernetes]
        namespace = "ci-jobs"
        image = "<registry.company.com ip>:5000/system/docker:latest"
        service_account = "gitlab-runner"
        poll_timeout = 600
        # privileged=true на уровне раннера делает все джобы привилегированными. Лучше:
        #    либо завести отдельный runner с тегом, скажем, build-priv, и использовать его только для сборки образов;
        #    либо убрать глобальный privileged=true и включать привилегии точечно в джобе переменной KUBERNETES_PRIVILEGED: "true" (раннер её понимает). 
        privileged = true
        #helper_image = "gitlab/gitlab-runner-helper:x86_64-latest"
        # ресурсы джоб-подов
        cpu_request = "250m"
        memory_request = "256Mi"
        cpu_limit = "1000m"
        memory_limit = "4Gi"
        # imagePullSecrets для реестра (не нужен если на нодах настроена аутентификация)
        # image_pull_secrets = ["regcred"]
        # аналог extra_hosts
        [[runners.kubernetes.volumes.secret]]
          name = "gitlab-ca"
          mount_path = "/etc/gitlab-runner/certs"

        # <registry.company.com ip>:5000 (Docker CLI)
        [[runners.kubernetes.volumes.secret]]
          name = "reg249-ca-docker"
          mount_path = "/etc/docker/certs.d/<registry.company.com ip>:5000"
        # <registry.company.com ip>:5000 (Buildah/Podman)
        [[runners.kubernetes.volumes.secret]]
          name = "reg249-ca-containers"
          mount_path = "/etc/containers/certs.d/<registry.company.com ip>:5000"

        # <gitlab.company.com ip>:5555 (Docker CLI)
        [[runners.kubernetes.volumes.secret]]
          name = "reg94-ca-docker"
          mount_path = "/etc/docker/certs.d/<gitlab.company.com ip>:5555"
        # <gitlab.company.com ip>:5555 (Buildah/Podman)
        [[runners.kubernetes.volumes.secret]]
          name = "reg94-ca-containers"
          mount_path = "/etc/containers/certs.d/<gitlab.company.com ip>:5555"

        [[runners.kubernetes.host_aliases]]
          ip = "<gitlab.company.com ip>"
          hostnames = ["gitlab.company.com"]
        [[runners.kubernetes.host_aliases]]
          ip = "<registry.company.com ip>"
          hostnames = ["registry2.company.com"]

      [runners.cache]
        Type = "s3"
        Shared = true

        [runners.cache.s3]
          ServerAddress  = "<s3.company.com ip>:9000"      # S3 API MinIO
          BucketName     = "gitlab-runner-cache"
          AccessKey      = "gitlab-runner"
          SecretKey      = "******************"
          Insecure       = true                      # true, если HTTP или самоподписанный TLS без доверенного CA
          PathStyle      = true                      # обязательно для MinIO
          BucketLocation = "us-east-1"               # любое валидное значение

# метрики самого менеджера раннера
metrics:
  enabled: true
```

Установка/обновление:
```bash
helm install gitlab-runner gitlab/gitlab-runner -n ci-system -f values.yaml
# или
helm upgrade --install gitlab-runner gitlab/gitlab-runner -n ci-system -f values.yaml
```

### 4) Проверка работоспособности
```bash
kubectl -n ci-system get pods -l app=gitlab-runner
kubectl -n ci-system logs deploy/gitlab-runner --tail=200
# тестовый pipeline с двумя стадиями: build (push cache) и test (pull cache)
```

---

## Заметки и рекомендации
- **Привилегии дла сборки образов:** глобальное `privileged=true` у Runner’а делает все job‑Pod’ы привилегированными. Для повышения безопасности используйте отдельный Runner/теги для сборки образов или включайте привилегии точечно переменной `KUBERNETES_PRIVILEGED: "true"` в конкретной job.
- **Проверка кэша:** используйте простой пайплайн build/test (push/pull) для валидации кэша S3.
- **MinIO/S3:** выделяйте отдельного пользователя, приватный bucket и минимальную политику (List/Get/Put/Delete). Ключи храните в Secret/CI variables.
- **Секреты:** создавайте в каждом нужном namespace (Runner и job‑namespace). Для Docker и Podman/Buildah используйте **разные** секреты и разные mountPath’ы.
