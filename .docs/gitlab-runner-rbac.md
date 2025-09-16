# RBAC: Права для GitLab Runner в Kubernetes

Этот документ описывает назначение и действие двух команд `kubectl`, создающих роль и привязку роли для сервисного аккаунта GitLab Runner.

---

## 1. Создание роли

```bash
kubectl -n ci-jobs create role gitlab-runner-manager   --verb=get,list,watch,create,update,patch,delete   --resource=pods,pods/log,pods/exec,secrets,configmaps,services,events
```

### Назначение
Создаёт объект **Role** с именем `gitlab-runner-manager` в пространстве имён `ci-jobs`.

### Разрешённые действия (verbs)
- `get` — получать объект.
- `list` — получать список объектов.
- `watch` — отслеживать изменения объектов.
- `create` — создавать объекты.
- `update` — обновлять объекты.
- `patch` — частично изменять объекты.
- `delete` — удалять объекты.

### Ресурсы
- `pods` — Pod’ы.
- `pods/log` — логи Pod’ов.
- `pods/exec` — выполнение команд в контейнере Pod’а.
- `secrets` — секреты (пароли, токены).
- `configmaps` — конфигурационные данные.
- `services` — сетевые сервисы.
- `events` — события в кластере.

**Итог:** Роль позволяет полное управление перечисленными ресурсами в namespace `ci-jobs`.

---

## 2. Привязка роли к сервисному аккаунту

```bash
kubectl -n ci-jobs create rolebinding gitlab-runner-manager-binding   --role=gitlab-runner-manager   --serviceaccount=ci-system:gitlab-runner
```

### Назначение
Создаёт объект **RoleBinding** с именем `gitlab-runner-manager-binding` в пространстве имён `ci-jobs`.

### Параметры
- `--role=gitlab-runner-manager` — указывает, какую роль привязать.
- `--serviceaccount=ci-system:gitlab-runner` — задаёт сервисный аккаунт:
  - Namespace: `ci-system`
  - Имя: `gitlab-runner`

**Итог:** сервисный аккаунт `gitlab-runner` (namespace `ci-system`) получает все права, заданные в роли `gitlab-runner-manager`, но только в namespace `ci-jobs`.

---

## 3. Связь объектов в RBAC

```plaintext
ServiceAccount (ci-system/gitlab-runner)
           │
           ▼
RoleBinding (ci-jobs/gitlab-runner-manager-binding)
           │
           ▼
Role (ci-jobs/gitlab-runner-manager) → Права на ресурсы
```

**Принцип:** ServiceAccount из одного namespace может быть привязан к Role в другом namespace, если RoleBinding находится в namespace роли.

---

## 4. Зачем это нужно GitLab Runner

GitLab Runner в режиме Kubernetes запускает CI job’ы внутри Pod’ов. Для корректной работы ему требуется:
- Создавать и удалять Pod’ы.
- Получать логи и выполнять команды в контейнерах.
- Доступ к ConfigMap и Secret для конфигурации и аутентификации.
- Создавать Service (при необходимости).
- Отслеживать события (events) для диагностики.

RBAC-настройки выше обеспечивают эти возможности **только в namespace `ci-jobs`**, что ограничивает зону действия и повышает безопасность.

---

## 5. Проверка и отладка

Проверить наличие роли:
```bash
kubectl -n ci-jobs get role gitlab-runner-manager -o yaml
```

Проверить привязку:
```bash
kubectl -n ci-jobs get rolebinding gitlab-runner-manager-binding -o yaml
```

Проверить права сервисного аккаунта (требуется плагин `kubectl-who-can`):
```bash
kubectl who-can create pods -n ci-jobs
```

---
