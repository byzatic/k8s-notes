# metrics-server — установка и устранение неполадок (сводка по диалогу)

Этот README консолидирует шаги установки `metrics-server` и набор типовых проблем с их решениями, которые встретились в диалоге‑дампе `metrics-server-fix-raw.md` (Q/A, разделённые `--R----R----`).

## Что хотим получить
- Рабочий `kubectl top nodes` и `kubectl top pods -A`.
- Стабильный `Deployment` `metrics-server` в `kube-system` без падений и с успешными readiness/liveness‑пробами.
- Корректную регистрацию API‑сервиса `v1beta1.metrics.k8s.io`.

---

## Быстрый рецепт (чистая переустановка с исправлениями)

> Выполняйте с правами, эквивалентными `cluster-admin`.

1) **Удалить всё ранее установленное (если есть):**
```bash
# Снести все объекты из upstream-манифеста
kubectl delete -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml --ignore-not-found

# На всякий случай добить возможные остатки
kubectl -n kube-system delete deploy/metrics-server svc/metrics-server sa/metrics-server --ignore-not-found
kubectl delete clusterrolebinding metrics-server:system:auth-delegator system:metrics-server --ignore-not-found
kubectl delete apiservice v1beta1.metrics.k8s.io --ignore-not-found
```

2) **Поставить заново из upstream:**
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

3) **Сразу задать нужные флаги контейнера и привести порт к 4443:**
> В изолированных средах kubelet часто с самоподписанными сертификатами и без IP SAN — поэтому включаем игнор TLS‑верификации и опрос по InternalIP. Порт внутри контейнера переводим на 4443, чтобы избежать конфликтов и сделать явный HTTPS‑порт.

```bash
# Заменить args у контейнера metrics-server
kubectl -n kube-system patch deploy metrics-server --type='json' -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/args",
   "value":[
     "--secure-port=4443",
     "--kubelet-insecure-tls",
     "--kubelet-preferred-address-types=InternalIP,Hostname,ExternalIP",
     "--metric-resolution=30s"
   ]}
]'
```

4) **Синхронизировать порт контейнера и порт/targetPort в сервисе:**
```bash
# Контейнер: порт 4443 и имя порта https (для проб)
kubectl -n kube-system patch deploy metrics-server --type='json' -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/ports",
   "value":[{"name":"https","containerPort":4443,"protocol":"TCP"}]}
]'

# Сервис: целевой порт 4443
kubectl -n kube-system patch svc metrics-server -p '{
  "spec": {"ports":[{"name":"https","port":443,"targetPort":4443,"protocol":"TCP"}]}
}'
```

5) **(Опционально, при read‑only rootfs) Указать каталог для самоподписанных сертификатов:**
```bash
kubectl -n kube-system patch deploy metrics-server --type='json' -p='[
  {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--cert-dir=/tmp"}
]'
```

6) **Подождать раскатку и проверить логи:**
```bash
kubectl -n kube-system rollout status deploy/metrics-server
kubectl -n kube-system logs deploy/metrics-server --tail=100
```

7) **Проверить APIService и метрики:**
```bash
kubectl get apiservices | grep metrics
kubectl describe apiservice v1beta1.metrics.k8s.io

kubectl top nodes
kubectl top pods -A
```

---

## Типовые проблемы и решения (встречались в диалоге)

### 1) Ошибки TLS при опросе kubelet
**Симптомы в логах:**
- `x509: cannot validate certificate for <IP> because it doesn't contain any IP SANs`
- `certificate signed by unknown authority`
- `context deadline exceeded` (как следствие TLS‑проблем)

**Причина:** сертификаты kubelet без IP SAN или самоподписанные/не доверенные для `metrics-server`.

**Решение (быстрый рабочий):**
```bash
# уже включено в рецепт выше
--kubelet-insecure-tls
--kubelet-preferred-address-types=InternalIP,Hostname,ExternalIP
```
> Долгосрочно — перевыпустить корректные сертификаты kubelet с IP SAN или настроить доверие.

---

### 2) CrashLoopBackOff после смены `--secure-port`
**Симптомы:**
- Один под в `CrashLoopBackOff`, другой `0/1`.
- В `describe deploy` видно `--secure-port=4443`, но в контейнере порт остался `10250` или не имеет имени `https`.

**Причина:** readiness/liveness‑пробы смотрят на порт с именем `https`. После смены `--secure-port` они проверяют не тот порт, под «не здоров» и перезапускается.

**Решение:**
```bash
# контейнерный порт и имя
kubectl -n kube-system patch deploy metrics-server --type='json' -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/ports",
   "value":[{"name":"https","containerPort":4443,"protocol":"TCP"}]}
]'
# сервис направить на 4443
kubectl -n kube-system patch svc metrics-server -p '{
  "spec": {"ports":[{"name":"https","port":443,"targetPort":4443,"protocol":"TCP"}]}
}'
```

---

### 3) `panic: error creating self-signed certificates: mkdir apiserver.local.config: read-only file system`
**Причина:** у пода read‑only rootfs, а `metrics-server` пытается писать сертификаты в каталог по умолчанию.

**Решение:**
```bash
kubectl -n kube-system patch deploy metrics-server --type='json' -p='[
  {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--cert-dir=/tmp"}
]'
```
> `tmp` уже смонтирован как `emptyDir`, запись разрешена.

---

### 4) `error: unknown flag: --containers` при `kubectl set args`
**Причина:** используемая версия `kubectl` не поддерживает `--containers` для `kubectl set args`.

**Решение:** использовать JSON‑patch (`kubectl patch --type='json'`) как в рецепте выше или `kubectl edit deploy/metrics-server`.

---

### 5) `Metrics API not available`
**Диагностика:**
```bash
kubectl -n kube-system logs deploy/metrics-server --tail=100
kubectl get apiservices | grep metrics
kubectl describe apiservice v1beta1.metrics.k8s.io
```
**Частые причины и решения:**
- TLS/адреса kubelet → см. п.1.
- Неверные порты/имена портов/пробы → см. п.2.
- Ожидание раскатки/инициализации — подождать 10–30 секунд после фиксов.

---

## Проверка (пример успешного вывода)
```text
NAME             CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node1.internal   247m         12%    5649Mi          48%
node2.internal   150m         3%     3082Mi          52%
node3.internal   199m         9%     2059Mi          54%
node4.internal   54m          1%     1307Mi          16%
node5.internal   24m          1%     847Mi           22%
node6.internal   126m         3%     1312Mi          17%
```

---

## Полезные команды наблюдения/диагностики
```bash
# состояние развертывания и раскатки
kubectl -n kube-system get deploy/metrics-server -o wide
kubectl -n kube-system rollout status deploy/metrics-server

# логи и события
kubectl -n kube-system logs deploy/metrics-server --tail=100
kubectl -n kube-system describe deploy/metrics-server
kubectl -n kube-system get pods -l app.kubernetes.io/name=metrics-server -o wide

# состояние APIService
kubectl get apiservices | grep metrics
kubectl describe apiservice v1beta1.metrics.k8s.io
```

---

## Замечания по безопасности
Флаг `--kubelet-insecure-tls` ослабляет проверку TLS при опросе kubelet и уместен в закрытых кластерах/сегментах сети. Для продакшна рекомендуется привести сертификаты kubelet в порядок (IP SAN, цепочка доверия) и затем убрать этот флаг.

---

## Результат
После применения шагов выше `metrics-server` стабильно проходит пробы, регистрирует `v1beta1.metrics.k8s.io`, а `kubectl top` возвращает метрики по нодам и подам.
