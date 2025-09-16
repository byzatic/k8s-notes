# GitLab Runner (Kubernetes, CRI‑O) — README по исправлениям

Этот документ описывает реальные проблемы, которые возникли при развертывании **GitLab Runner с Kubernetes executor** в namespace `ci-system` (менеджер) и запуске job‑подов в `ci-jobs` на кластере с **CRI‑O**. Для каждой ошибки приведены:
- **Ошибка**
- **Объяснение ошибки**
- **Исправление**
- **Объяснение исправления**

В конце — итоговый фрагмент `values.yaml`, полезные команды проверки и минимальный тестовый job.

---

## Содержание

1. [TLS: `x509: certificate signed by unknown authority` при регистрации раннера](#1-tls-x509-certificate-signed-by-unknown-authority)
2. [Регистрация по legacy‑токену: предупреждение](#2-legacy-token-warning)
3. [RBAC: нет прав создавать `secrets` в `ci-jobs`](#3-rbac-secrets-forbidden)
4. [Отсутствует ServiceAccount для job‑подов](#4-serviceaccount-missing)
5. [Secret `gitlab-ca` не найден в `ci-jobs`](#5-secret-gitlab-ca-not-found)
6. [RBAC: `pods/attach` запрещён](#6-rbac-pods-attach-forbidden)
7. [CRI‑O: доступ к приватным реестрам (insecure + логин)](#7-cri-o-registries)
8. [Примеры и финальные конфиги](#8-examples)

---

## 1) TLS: `x509: certificate signed by unknown authority`

**Ошибка**

```
ERROR: Verifying runner... failed
... tls: failed to verify certificate: x509: certificate signed by unknown authority
```

**Объяснение ошибки**

Менеджер Runner’а (под в `ci-system`) стучится на `https://<gitlab.company.com ip>:8929/`, но **не доверяет сертификату** GitLab. Дополнительно, если в сертификате имя ≠ IP, нужна подсказка имени (SNI/hostname) при проверке.

**Исправление**

1. Создать Secret с публичным сертификатом GitLab:
```bash
kubectl -n ci-system create secret generic gitlab-ca \
  --from-file=<gitlab.company.com ip>.crt=/path/to/<gitlab.company.com ip>.crt
```

2. В Helm `values.yaml` указать:
```yaml
certsSecretName: gitlab-ca
```

3. (Если сертификат выдан на доменное имя, а используем IP) Добавить в `runners.config`:
```toml
tls_server_name = "gitlab.company.com"
```

**Объяснение исправления**

`certsSecretName` монтирует сертификат в `/etc/gitlab-runner/certs` внутри менеджера. Теперь раннер доверяет TLS цепочке GitLab. `tls_server_name` задаёт ожидаемое имя в сертификате при обращении по IP.

---

## 2) Legacy token warning

**Предупреждение**

```
WARNING: You have specified an authentication token in the legacy parameter --registration-token...
```

**Объяснение**

Runner 18.x предупреждает, что `--registration-token` запускает «legacy workflow», где часть параметров игнорируется.

**Исправление (необязательно сейчас)**

Перейти на **новый токен аутентификации** (создать раннер в UI GitLab → получить Authentication token) и в `values.yaml` использовать:
```yaml
runnerToken: "<AUTH_TOKEN>"
```
(И убрать `runnerRegistrationToken`.)

**Объяснение исправления**

Новый токен делает установку идемпотентной и убирает предупреждение. На работоспособность сейчас не влияет.

---

## 3) RBAC: `secrets is forbidden` в `ci-jobs` <a id="3-rbac-secrets-forbidden"></a>

**Ошибка**

```
secrets is forbidden: User "system:serviceaccount:ci-system:gitlab-runner" cannot create resource "secrets" in the namespace "ci-jobs"
```

**Объяснение ошибки**

Менеджер Runner’а (SA `ci-system:gitlab-runner`) в процессе джоба создаёт временные Secrets/Service/Pod и т.п. в `ci-jobs`. У него не было прав в этом namespace.

**Исправление**

Создать роль и привязку в `ci-jobs`:
```bash
kubectl -n ci-jobs create role gitlab-runner-manager \
  --verb=get,list,watch,create,update,patch,delete \
  --resource=pods,pods/log,pods/exec,secrets,configmaps,services,events

kubectl -n ci-jobs create rolebinding gitlab-runner-manager-binding \
  --role=gitlab-runner-manager \
  --serviceaccount=ci-system:gitlab-runner
```

**Объяснение исправления**

Роль даёт необходимые CRUD‑права на ключевые ресурсы, а биндинг назначает их **менеджеру** Runner’а из другого namespace.

---

## 4) ServiceAccount отсутствует в `ci-jobs` <a id="4-serviceaccount-missing"></a>

**Ошибка**

```
Error from server (NotFound): serviceaccounts "gitlab-runner" not found
```

**Объяснение ошибки**

В `runners.config` задано:
```toml
service_account = "gitlab-runner"
```
Runner пытается запускать job‑поды под этим SA в `ci-jobs`, но SA отсутствовал.

**Исправление**

Создать SA (и при необходимости права для job‑подов, если им надо в K8s API):
```bash
kubectl -n ci-jobs create serviceaccount gitlab-runner
# (опционально) только чтение API:
kubectl -n ci-jobs create rolebinding gitlab-runner-view \
  --clusterrole=view --serviceaccount=ci-jobs:gitlab-runner
```

**Объяснение исправления**

Теперь поды джобов успешно назначаются на существующий SA. Права `view` нужны только если сами джобы ходят в Kubernetes API.

---

## 5) Secret `gitlab-ca` не найден в `ci-jobs`

**Ошибка**

События пода:
```
FailedMount ... secret "gitlab-ca" not found
```

**Объяснение ошибки**

В `runners.config` смонтирован секрет:
```toml
[[runners.kubernetes.volumes.secret]]
  name = "gitlab-ca"
  mount_path = "/etc/gitlab-runner/certs"
```
Но секрет `gitlab-ca` был создан в `ci-system`, а job‑поды живут в `ci-jobs`. Секреты **не доступны** кросс‑неймспейсно.

**Исправление**

Создать такой же секрет в `ci-jobs`:
```bash
# из файла:
kubectl -n ci-jobs create secret generic gitlab-ca \
  --from-file=<gitlab.company.com ip>.crt=/path/to/<gitlab.company.com ip>.crt
# либо скопировать из ci-system с помощью jq
```

**Объяснение исправления**

Теперь job‑поды видят секрет в своём namespace и могут смонтировать сертификат.

---

## 6) RBAC: `pods/attach` запрещён

**Ошибка**

```
pods "runner-..." is forbidden: User "system:serviceaccount:ci-system:gitlab-runner" cannot create resource "pods/attach" in the namespace "ci-jobs"
```

**Объяснение ошибки**

Для стратегии исполнения Runner’у нужно **подключаться к контейнерам** (`attach`). Для subresource `pods/attach` (и `pods/exec`) нужен verb `create`.

**Исправление**

Обновить роль в `ci-jobs`, добавив subresources:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: gitlab-runner-manager
  namespace: ci-jobs
rules:
- apiGroups: [""]
  resources:
    - pods
    - pods/log
    - pods/exec
    - pods/attach
    - pods/portforward
    - secrets
    - configmaps
    - services
    - events
  verbs: ["get","list","watch","create","update","patch","delete"]
```

**Объяснение исправления**

Subresources требуют явного перечисления. Теперь Runner может устанавливать attach/exec‑соединения к контейнерам джоба.

---

## 7) CRI‑O: доступ к приватным реестрам (insecure + логин)

**Действия**

- На всех нодах CRI‑O разрешены нужные реестры (self‑signed/без TLS):
```toml
# /etc/containers/registries.conf
[[registry]]
prefix = "registry2.company.com:5000"
location = "registry2.company.com:5000"
insecure = true
blocked = false

[[registry]]
prefix = "<registry.company.com ip>:5000"
location = "<registry.company.com ip>:5000"
insecure = true
blocked = false

[[registry]]
prefix = "<gitlab.company.com ip>:5000"
location = "<gitlab.company.com ip>:5000"
insecure = true
blocked = false
# ... и остальные при необходимости
```

- Учетные данные на нодах (kubelet/CRI‑O) в `/var/lib/kubelet/config.json`:
```json
{
  "auths": {
    "<registry.company.com ip>:5000": { "auth": "BASE64(user:pass)" },
    "registry2.company.com:5000": { "auth": "BASE64(user:pass)" },
    "<gitlab.company.com ip>:5555": { "auth": "BASE64(user:pass)" }
    // ...
  }
}
```

- Перезапуск CRI‑O после изменений:
```bash
sudo systemctl restart crio
```

**Объяснение**

Pull образов выполняет **CRI‑O на нодах**, поэтому доверие к реестрам (TLS/insecure) и логины должны быть настроены **на каждой ноде**. Это позволяет не указывать `image_pull_secrets` в Runner’е (опционально).

---

## 8) Примеры

### 8.1 Итоговый фрагмент `values.yaml` (упрощённый)

```yaml
gitlabUrl: "https://<gitlab.company.com ip>:8929/"
# runnerToken: "<AUTH_TOKEN>"                      # рекомендовано на будущее
unregisterRunners: true
replicas: 2

certsSecretName: gitlab-ca

rbac:
  create: true
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
      environment = ["GIT_SUBMODULE_STRATEGY=recursive"]

      [runners.kubernetes]
        namespace = "ci-jobs"
        image = "<registry.company.com ip>:5000/system/docker:latest"
        service_account = "gitlab-runner"
        poll_timeout = 600
        cpu_request = "250m"
        memory_request = "256Mi"
        cpu_limit = "1000m"
        memory_limit = "1Gi"

        [[runners.kubernetes.volumes.secret]]
          name = "gitlab-ca"
          mount_path = "/etc/gitlab-runner/certs"

        [[runners.kubernetes.host_aliases]]
          ip = "<gitlab.company.com ip>"
          hostnames = ["gitlab.company.com"]
        [[runners.kubernetes.host_aliases]]
          ip = "<registry.company.com ip>"
          hostnames = ["registry2.company.com"]

metrics:
  enabled: true
```

### 8.2 RBAC манифесты (итог)

```yaml
# Role для менеджера Runner’а (ci-system SA управляет ресурсами в ci-jobs)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: gitlab-runner-manager
  namespace: ci-jobs
rules:
- apiGroups: [""]
  resources:
    - pods
    - pods/log
    - pods/exec
    - pods/attach
    - pods/portforward
    - secrets
    - configmaps
    - services
    - events
  verbs: ["get","list","watch","create","update","patch","delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: gitlab-runner-manager-binding
  namespace: ci-jobs
subjects:
- kind: ServiceAccount
  name: gitlab-runner
  namespace: ci-system
roleRef:
  kind: Role
  name: gitlab-runner-manager
  apiGroup: rbac.authorization.k8s.io
```

И ServiceAccount для job‑подов:
```bash
kubectl -n ci-jobs create serviceaccount gitlab-runner
kubectl -n ci-jobs create rolebinding gitlab-runner-view \
  --clusterrole=view --serviceaccount=ci-jobs:gitlab-runner
```

### 8.3 Проверки

- Статус раннера:
```bash
kubectl -n ci-system get pods -l app=gitlab-runner
kubectl -n ci-system logs -f deploy/gitlab-runner
```

- Прав доступа:
```bash
kubectl auth can-i --as=system:serviceaccount:ci-system:gitlab-runner -n ci-jobs create pods/attach
kubectl auth can-i --as=system:serviceaccount:ci-system:gitlab-runner -n ci-jobs create secrets
kubectl auth can-i --as=system:serviceaccount:ci-system:gitlab-runner -n ci-jobs get pods/log
```

- Pull образов на ноде:
```bash
sudo crictl pull <registry.company.com ip>:5000/system/docker:latest
```

### 8.4 Минимальный тестовый job

```yaml
# .gitlab-ci.yml
stages: [check]
check:
  stage: check
  tags: [k8s]    # или твои теги
  script:
    - echo "OK from k8s runner"
    - uname -a
```

---

## Примечания

- `helper_image :latest` лучше не задавать: чарт сам подберёт совместимый helper к версии Runner’а.
- `image_pull_secrets` **не обязателен**, так как авторизация к реестрам реализована на нодах (CRI‑O + kubelet config.json). Оставляй только если хочешь самодостаточность на уровне pod.
- Для ускорения билдов включи кэш S3/MinIO и/или сделай «обогрев» образов на нодах (DaemonSet с periodic pull).

Удачной работы!
