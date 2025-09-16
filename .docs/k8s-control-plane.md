# Kubernetes Control Plane — академическое введение и практическое руководство

## 1. Фундаментальное описание

**Control Plane** — это распределённая подсистема Kubernetes, реализующая *петлю управления*: желаемое состояние (декларативные манифесты) сравнивается с наблюдаемым, а контроллеры приводят кластер к консистентному состоянию. Control Plane определяет API кластера, хранит единый источник истины (etcd), планирует размещение подов и выполняет управляющие решения через контроллеры.

Ключевые свойства:
- **Декларативность**: объектная модель (Pod, Deployment, Service, …) хранится в `etcd` и излагает желаемое состояние.
- **Наблюдаемость**: состояние считывается через kube-apiserver из `etcd` и из периодических отчётов агентов (kubelet, контроллеров).
- **Обратная связь**: контроллеры приводят фактическое состояние к желаемому, реагируя на события/изменения.
- **Надёжность**: в продуктивных кластерах Control Plane развёртывается с кворумом (3–5 узлов `etcd`) и независимыми API‑endpoint’ами.

Составные компоненты Control Plane (типичный kubeadm‑кластер):
- **kube-apiserver** — единственная точка входа (REST) ко всем объектам кластера и операторам управления. Реализует аутентификацию, авторизацию (RBAC/ABAC), аудит, webhooks и admission chain.
- **etcd** — распределённое key–value хранилище, где хранится вся конфигурация и текущее состояние кластера. Требует регулярных бэкапов и чёткой политики кворума.
- **kube-scheduler** — сопоставляет новые Pod’ы подходящим нодам, учитывая ресурсы, taints/tolerations, топологию и policy.
- **kube-controller-manager** — набор контроллеров (ReplicaSet, Node, EndpointSlice, Job/CronJob и др.), поддерживающих инвариант «желательно = фактически».
- **cloud-controller-manager** (в облаке) — абстрагирует интеграции с провайдером (LB, маршруты, диски), перемещён из ядра в отдельный процесс.
- (Исторически) **kube-proxy** — data‑plane‑компонент на нодах; к Control Plane логически не относится, но тесно связан с сетевой моделью сервиса.

Архитектурные модели отказоустойчивости:
- **Single-CP (dev/test)**: один узел control‑plane; минимальная отказоустойчивость.
- **HA, stacked etcd**: на каждом control‑plane узле одновременно статические поды `kube-apiserver`, `kube-controller-manager`, `kube-scheduler` и локальный `etcd`; кворум по `etcd`.
- **HA, external etcd**: несколько control‑plane узлов + независимый кластер `etcd` (часто выделенные ВМ/узлы). Облегчает обновления и DR.

Модель безопасности Control Plane:
- **AuthN** (сертификаты, OIDC, токены), **AuthZ** (RBAC), **Admission** (встроенные и вебхуки), **Аудит** (audit policy).
- Рекомендуется защищать API сервер через LB с TLS, ограничивать доступ по сети и сегментации, вести аудит и ротацию сертификатов.

По умолчанию **control‑plane узлы** имеют taint `node-role.kubernetes.io/control-plane:NoSchedule`, чтобы пользовательские Pod’ы не планировались на них.

---

## 2. Что входит в Control Plane (конспект)

1. **kube-apiserver** — центральная точка общения (kubectl/клиенты/контроллеры → REST API).
2. **etcd** — KV‑хранилище состояния кластера.
3. **kube-scheduler** — выбор целевой ноды для Pod.
4. **kube-controller-manager** — контроллеры реплик, нод, джоб, эндпоинтов и др.
5. **cloud-controller-manager** (в облаке) — интеграция с API провайдера (LB, диски, маршрутизация).

**Роль control‑plane ноды**:
- Принимает/обрабатывает команды (kubectl apply/delete и т.п.).
- Следит за состоянием всех объектов.
- Не размещает пользовательские Pod’ы (из‑за taint).
- Должна быть максимально стабильной, обновляемой и защищённой.

---

## 3. Операционные практики

### 3.1. Наблюдение и диагностика
```bash
# Компоненты как поды (kubeadm: статические поды в kube-system)
kubectl -n kube-system get pods -o wide | egrep 'kube-(api|controller|scheduler)'

# События и состояние
kubectl -n kube-system describe pod -l component=kube-apiserver
kubectl -n kube-system logs -l component=kube-apiserver --tail=200

# Состояние etcd (если управляется kubeadm и запущен как static pod)
kubectl -n kube-system logs -l component=etcd --tail=200
```

Замечание: устаревшую команду `kubectl get componentstatuses` использовать не рекомендуется; ориентируйтесь на логи/метрики/пробы.

### 3.2. Бэкапы и DR (etcd)
- Регулярные snapshot’ы `etcd` (`etcdctl snapshot save`), хранение вне кластера.
- Тест восстановления (`etcdctl snapshot restore`), документация DR.
- Мониторинг латентности/размера БД/кворума.

### 3.3. Обновления и несовместимости версий
- Версия kubelet поддерживает **N-2** относительно kube‑apiserver.
- Для HA — последовательное обновление control‑plane узлов, затем worker’ов.
- Проверка webhook’ов/admission плагинов на совместимость.

### 3.4. Безопасность
- Минимизируйте сетевую доступность API‑сервера (файрволы, private LB).
- Включайте аудит и централизуйте логи.
- Используйте ограничительные Policy (PSa/PSP‑замены, Validating/MutatingWebhook’и) там, где это оправдано.

---

## 4. Типичные неполадки и подход к разбору

- **API‑сервер недоступен**: проверяем LB/DNS, сертификаты, логи `kube-apiserver` (bind‑порт, webhooks).
- **Проблемы `etcd`**: нет кворума, высокая задержка диска/сети, переполненный WAL/снапшоты → см. логи и состояние пиров.
- **Scheduler не размещает Pod’ы**: taints/affinity, нехватка ресурсов, поломка CNI/Node‑проверок.
- **Controller‑manager не конвергирует состояние**: ошибки доступа к API, конфликты версии, зависшие finalizer’ы.

---

## 5. Краткая памятка команд

```bash
# Просмотр control-plane подов
kubectl -n kube-system get pods -o wide | egrep 'kube-(api|controller|scheduler|etcd)'

# Логи ключевых компонентов
kubectl -n kube-system logs -l component=kube-apiserver --tail=200
kubectl -n kube-system logs -l component=kube-controller-manager --tail=200
kubectl -n kube-system logs -l component=kube-scheduler --tail=200
kubectl -n kube-system logs -l component=etcd --tail=200

# Сервисы/эндпоинты доступа к API (если через сервис/ingress/LB)
kubectl -n default get endpoints kubernetes -o wide
```
