# RBAC в Kubernetes — практическое руководство

Этот README объясняет модель RBAC (Role-Based Access Control) в Kubernetes, основные сущности, лучшие практики, проверку прав и типовые примеры. Включено применение RBAC для GitLab Runner, а также полезные команды для диагностики и аудита.

---

## Что такое RBAC

**RBAC** (*Role-Based Access Control*, роль-ориентированное управление доступом) — модель контроля доступа, в которой права субъектов (пользователей, групп, сервисных аккаунтов) выдаются через **роли** и их **привязки**. Вместо назначения прав напрямую субъекту, вы создаёте роль с набором разрешённых действий над ресурсами, а затем привязываете роль к субъекту.

---

## Базовые сущности RBAC

| Объект                 | Область действия                  | Назначение |
|------------------------|-----------------------------------|------------|
| **Role**               | Namespace-скоуп                   | Набор правил (rules) — какие **verbs** разрешены над какими **resources** в пределах одного namespace. |
| **ClusterRole**        | Кластерный скоуп                  | Аналог Role, но действует для всего кластера; может применяться как к кластерным, так и к namespaced-ресурсам. |
| **RoleBinding**        | Namespace-скоуп                   | Привязывает Role к субъектам в указанном namespace. |
| **ClusterRoleBinding** | Кластерный скоуп                  | Привязывает ClusterRole к субъектам на уровне всего кластера (или сразу к нескольким ns). |

**Субъекты (subjects):** `User`, `Group`, `ServiceAccount`.

**Правила (rules):**
- **verbs**: `get`, `list`, `watch` (чтение), `create`, `update`, `patch` (изменение), `delete` (удаление), иногда `deletecollection`.
- **resources**: `pods`, `deployments`, `secrets`, `configmaps`, `services`, `events`, и т.д.
- **apiGroups**: `""` (core), `apps`, `batch`, `rbac.authorization.k8s.io`, и др.
- **resourceNames** *(опционально)*: ограничение до конкретных ресурсов по имени.
- **nonResourceURLs** *(ClusterRole)*: доступ к путям API вне объектной модели (например, `/metrics`).

---

## Минимальный пример (чтение Pod’ов в ns)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: my-sa
    namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Что даёт:** сервисный аккаунт `dev/my-sa` может читать Pod’ы только в `dev`.

---

## Пример ClusterRole + ClusterRoleBinding (кластерные права)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: view-nodes
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: view-nodes-binding
subjects:
  - kind: User
    name: alice@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view-nodes
  apiGroup: rbac.authorization.k8s.io
```

**Что даёт:** пользователь `alice@example.com` может читать `nodes` во всём кластере.

---

## Пример с ограничением на конкретные объекты (resourceNames)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: prod
  name: edit-only-configmap-x
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["app-config"]
    verbs: ["get", "update", "patch"]
```

**Что даёт:** разрешено редактировать **только** `ConfigMap/prod/app-config`.

---

## RBAC для GitLab Runner (пример из практики)

Эти две команды создают в Kubernetes роль и привязку роли (RBAC) для сервисного аккаунта GitLab Runner’а, чтобы он мог управлять ресурсами в namespace `ci-jobs`.

```bash
kubectl -n ci-jobs create role gitlab-runner-manager   --verb=get,list,watch,create,update,patch,delete   --resource=pods,pods/log,pods/exec,secrets,configmaps,services,events

kubectl -n ci-jobs create rolebinding gitlab-runner-manager-binding   --role=gitlab-runner-manager   --serviceaccount=ci-system:gitlab-runner
```

**Разбор:**
- Роль `gitlab-runner-manager` (ns: `ci-jobs`) — полный набор прав для работы с `pods`, `pods/log`, `pods/exec`, `secrets`, `configmaps`, `services`, `events`.
- Привязка `gitlab-runner-manager-binding` выдаёт эту роль сервисному аккаунту `ci-system/gitlab-runner`, что позволяет Runner’у создавать/удалять Pod’ы для job’ов, читать логи, выполнять команды, получать конфиги/секреты и т.д. в **namespace `ci-jobs`**.

> Примечание: сервисный аккаунт из одного namespace (`ci-system`) может получать права в другом (`ci-jobs`) через RoleBinding, созданный в целевом namespace.

```
kubectl -n ci-jobs create role gitlab-runner-manager \
  --verb=get,list,watch,create,update,patch,delete \
  --resource=pods,pods/log,pods/exec,secrets,configmaps,services,events
```

**Что происходит:**
  - -n ci-jobs — роль создаётся внутри namespace ci-jobs (роль namespace-скоупная).
  - create role gitlab-runner-manager — создаётся объект типа Role с именем gitlab-runner-manager.
  - --verb=... — перечислены действия (verbs), которые разрешены:
  - get — получить объект.
  - list — получить список объектов.
  - watch — «подписаться» на события изменений объектов.
  - create — создавать новые объекты.
  - update — изменять существующие.
  - patch — частично изменять объект.
  - delete — удалять объект.
  - --resource=... — указаны ресурсы, к которым применяются разрешения:
  - pods — сами Pod’ы.
  - pods/log — логи Pod’ов.
  - pods/exec — выполнение команд внутри контейнера Pod’а (kubectl exec).
  - secrets — секреты (например, токены, пароли).
  - configmaps — конфигурационные данные.
  - services — объекты Service.
  - events — события в кластере.

**Итог:** роль позволяет полностью управлять перечисленными ресурсами в ci-jobs.

```
kubectl -n ci-jobs create rolebinding gitlab-runner-manager-binding \
  --role=gitlab-runner-manager \
  --serviceaccount=ci-system:gitlab-runner
```

**Что происходит:**
  - create rolebinding gitlab-runner-manager-binding — создаётся объект RoleBinding с именем gitlab-runner-manager-binding.
  - --role=gitlab-runner-manager — указываем, какую роль привязываем.
  - --serviceaccount=ci-system:gitlab-runner — указываем сервисный аккаунт, которому назначается эта роль:
  - namespace аккаунта — ci-system.
  - имя аккаунта — gitlab-runner.

**Итог:** сервисный аккаунт gitlab-runner (из namespace ci-system) получает в namespace ci-jobs все права, которые определены в роли gitlab-runner-manager.

**Зачем это в контексте GitLab Runner**

В Kubernetes GitLab Runner запускает job’ы внутри Pod’ов. Чтобы он мог:
  - создавать Pod’ы для CI job’ов,
  - загружать их логи,
  - выполнять команды (kubectl exec),
  - читать конфиги и секреты,
  - создавать сервисы, слушать события, ему нужны соответствующие права через RBAC.

Без этих команд Runner не смог бы корректно работать в ci-jobs.

---

## Полезные kubectl-команды для проверки и аудита

```bash
# Посмотреть YAML роли/привязки
kubectl -n <ns> get role <name> -o yaml
kubectl -n <ns> get rolebinding <name> -o yaml
kubectl get clusterrole <name> -o yaml
kubectl get clusterrolebinding <name> -o yaml

# Кто может выполнить действие? (плагин kubectl-who-can)
kubectl who-can create pods -n ci-jobs

# Может ли мой текущий пользователь?
kubectl auth can-i get pods -n dev
kubectl auth can-i create deployments --as=alice@example.com -n dev

# Обзор всех ролей/привязок в ns
kubectl -n <ns> get role,rolebinding
kubectl get clusterrole,clusterrolebinding

# Проверить api-resources (какие ресурсы и apiGroups существуют)
kubectl api-resources
```

---

## Best practices (минимизация рисков)

1. **Принцип минимальных привилегий**: выдавайте только те verbs и ресурсы, которые требуются.
2. **Разделяйте окружения по Namespace** и выдавайте права адресно.
3. **Ограничивайте доступ к Secret’ам** — чаще всего это самая чувствительная часть.
4. **Используйте групповые роли**: привязывайте роли к группам, а пользователей добавляйте/убирайте из групп.
5. **Версионируйте RBAC в Git**: применяйте `kubectl apply -f` через CI/CD.
6. **Аудитируйте**: регулярно проверяйте `Role/ClusterRole` на избыточные разрешения.
7. **Разделяйте роли по ответственности**: чтение/запись/администрирование — в разные роли.

---

## Распространённые шаблоны

### ReadOnly для команды разработки в ns
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dev-readonly
  namespace: dev
rules:
  - apiGroups: ["", "apps", "batch"]
    resources: ["pods", "pods/log", "deployments", "replicasets", "jobs", "cronjobs", "services", "events"]
    verbs: ["get", "list", "watch"]
```

### CI-пайплайн: создавать/удалять Pod’ы и читать их логи
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ci-pod-manager
  namespace: ci-jobs
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "pods/exec", "events"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    verbs: ["get"]
```

### Доступ к nonResourceURLs (пример для метрик API-сервера)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: metrics-reader
rules:
  - nonResourceURLs: ["/metrics"]
    verbs: ["get"]
```

---

## Тонкости и частые ошибки

- **Несоответствие namespace’ов**: `RoleBinding` должен находиться в том ns, где действует `Role`. При этом `subjects[].namespace` — это ns сервисного аккаунта, он может отличаться.
- **Забытые apiGroups**: для core API используйте `apiGroups: [""]`. Для `deployments` — `apps`, для `jobs/cronjobs` — `batch`.
- **Избыточные права**: избегайте `*` в verbs/resources без необходимости.
- **Отсутствие rights на pods/exec**: для `kubectl exec` нужен ресурс `pods/exec`.
- **nonResourceURLs только в ClusterRole**: в Role их не бывает.

---

## Заключение

RBAC — ключевой механизм безопасного и управляемого доступа к ресурсам Kubernetes. Грамотная декомпозиция прав через Role/ClusterRole и их привязки обеспечивает изоляцию, удобство администрирования и соответствие принципу минимальных привилегий. Используйте приведённые шаблоны как основу и адаптируйте под требования вашей организации.
