# Kubernetes: Namespace · Pod · Service · Deployment — Практическая шпаргалка

Этот файл — компактный справочник по созданию, настройке и обслуживанию базовых объектов Kubernetes и связей между ними. Команды безопасно запускать с правами и контекстом нужного кластера.

---

## 1) Namespace (Неймспейс)

### Назначение
Логическая область внутри кластера для изоляции ресурсов (Pods, Services, ConfigMaps, Secrets, Deployments), лимитов и квот.

### Базовые операции
```bash
# Создать namespace
kubectl create namespace test-env

# Просмотр всех неймспейсов
kubectl get ns

# Получать ресурсы в конкретном ns
kubectl get pods -n test-env
kubectl get all  -n test-env

# Удалить namespace (все ресурсы внутри будут удалены)
kubectl delete ns test-env
```

### Полезно для администрирования
```bash
# Ограничения и квоты
kubectl get resourcequota -n test-env
kubectl get limitrange -n test-env

# Подробности
kubectl describe ns test-env

# Вывести все объекты в ns (кроме CRD-ресурсов)
kubectl api-resources --namespaced=true -o name |   xargs -I{} kubectl get {} -n test-env --ignore-not-found
```

### Пример манифестов
```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: test-env
  labels:
    team: core
---
# limitrange.yaml — дефолтные requests/limits для контейнеров
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: test-env
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
---
# resourcequota.yaml — квоты на ресурсы в ns
apiVersion: v1
kind: ResourceQuota
metadata:
  name: standard-quota
  namespace: test-env
spec:
  hard:
    pods: "50"
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
```
```bash
kubectl apply -f namespace.yaml
kubectl apply -f limitrange.yaml
kubectl apply -f resourcequota.yaml
```

---

## 2) Pod (Под)

### Суть
Минимальная единица развертывания: один или несколько контейнеров, общий IPC/UTS/PID (опционально), сеть (общий IP), тома.

### Быстрый старт
```bash
# Создать под из образа nginx
kubectl run my-app --image=nginx

# Смотреть список, IP, ноду
kubectl get pods -o wide

# Детали по конкретному поду
kubectl describe pod my-app

# Логи (все контейнеры или конкретный)
kubectl logs my-app
kubectl logs my-app -c <container-name> -f --since=10m

# Подключиться внутрь (interactive shell)
kubectl exec -it my-app -- sh
kubectl exec -it my-app -c <container> -- bash
```

### Отладка
```bash
# Группировка по нодам (удобно для анализа)
kubectl get pods -A -o wide --sort-by=.spec.nodeName

# Список всех Pod’ов с указанием ноды
kubectl get pods -A -o wide

# События (последние, с сортировкой по времени)
kubectl get events --sort-by=.metadata.creationTimestamp

# Подождать, пока под станет Ready
kubectl wait --for=condition=Ready pod/my-app --timeout=120s

# Скопировать файл внутрь/наружу
kubectl cp ./local.file default/my-app:/tmp/
kubectl cp default/my-app:/etc/nginx/nginx.conf ./nginx.conf
```

### Пример Pod-манифеста (sidecar + volume)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
  labels:
    app: example
spec:
  containers:
    - name: app
      image: nginx:1.27
      ports:
        - containerPort: 80
      volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 5
    - name: log-sidecar
      image: busybox
      args: ["sh", "-c", "tail -n+1 -F /var/log/app.log"]
      volumeMounts:
        - name: data
          mountPath: /var/log
  volumes:
    - name: data
      emptyDir: {}
```

---

## 3) Deployment (Деплоймент)

### Суть
Управляет ReplicaSet/Pods, обеспечивает **масштабирование**, **rolling update**, **rollback**.

### Основные операции
```bash
# Создать деплоймент
kubectl create deployment my-app --image=nginx:1.27

# Масштабировать
kubectl scale deployment my-app --replicas=3

# Посмотреть раскатку/историю/откат
kubectl rollout status  deployment/my-app
kubectl rollout history deployment/my-app
kubectl rollout undo    deployment/my-app
kubectl rollout undo    deployment/my-app --to-revision=2

# Обновить образ на лету
kubectl set image deployment/my-app app=nginx:1.27.1

# Применить декларативный YAML
kubectl apply -f deployment.yaml
```

### Автомасштабирование (HPA)
```bash
# Требуется metrics-server
kubectl autoscale deployment my-app --min=2 --max=10 --cpu-percent=70

# Просмотр HPA
kubectl get hpa
```

### Пример deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 25%
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: nginx:1.27
          ports:
            - containerPort: 8080
          env:
            - name: ENV
              value: "prod"
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
          livenessProbe:
            httpGet: { path: /healthz, port: 8080 }
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet: { path: /ready, port: 8080 }
            initialDelaySeconds: 5
            periodSeconds: 5
```

---

## 4) Service (Сервис)

### Суть
Стабильная точка доступа (DNS/ClusterIP) и балансировка трафика к наборам Pod’ов (по `selector`).

### Типы Service
- **ClusterIP** (по умолчанию) — доступ *внутри* кластера.
- **NodePort** — доступ извне через `NODE_IP:NodePort`.
- **LoadBalancer** — внешний балансировщик (обычно облако).
- **Headless** (`clusterIP: None`) — без VIP, даёт прямой доступ к Pod’ам (например, StatefulSet).
- **ExternalName** — CNAME на внешнее DNS-имя, без прокси.

### Основные операции
```bash
# Экспонировать Deployment как Service
kubectl expose deployment my-app --port=80 --target-port=8080 --type=ClusterIP

# Просмотр сервисов и их адресов
kubectl get svc
kubectl get svc -o wide

# Детали/эндпоинты
kubectl describe svc my-app
kubectl get endpoints my-app
kubectl get endpointslices -l kubernetes.io/service-name=my-app

# Проверка доступности изнутри кластера (через временный под)
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash
# nslookup my-app.default.svc.cluster.local
# curl -I http://my-app.default.svc.cluster.local:80/

# Проброс порта локально
kubectl port-forward svc/my-app 8080:80
```

### Примеры YAML
```yaml
# service-clusterip.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
  ports:
    - name: http
      port: 80        # порт сервиса
      targetPort: 8080 # порт контейнера
  type: ClusterIP
---
# service-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-nodeport
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080     # (опционально) иначе назначит автоматически из 30000-32767
  type: NodePort
---
# service-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-lb
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
  type: LoadBalancer
  externalTrafficPolicy: Local  # сохранить исходный клиентский IP на бэкенде
---
# headless-service.yaml — без VIP, для Statefull приложений/сервис-дискавери
apiVersion: v1
kind: Service
metadata:
  name: my-app-headless
spec:
  clusterIP: None
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
---
# externalname.yaml — проброс на внешний DNS
apiVersion: v1
kind: Service
metadata:
  name: ext-api
spec:
  type: ExternalName
  externalName: api.example.com
```

### Тонкости и диагностика
```bash
# Проверить, что сервис «видит» поды (endpoints не пустой)
kubectl get endpoints my-app -o yaml

# Посмотреть адреса подов, на которые указывает сервис
kubectl get pods -l app=my-app -o jsonpath='{range .items[*]}{.metadata.name} {.status.podIP}{"
"}{end}'

# Проверить kube-proxy и режим (iptables/ipvs)
kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide
```

---

## 5) Связи: Namespace ⇄ Deployment ⇄ Pod ⇄ Service

- `Deployment.spec.selector.matchLabels` **должен совпадать** с `template.metadata.labels` — иначе обновления не будут работать.
- `Service.spec.selector` **должен совпадать** с метками Pod’ов, чтобы формировались Endpoints.
- Все объекты должны быть в **одном и том же namespace**, если не используете FQDN при обращении через DNS.

Проверка соответствий:
```bash
# Метки подов
kubectl get pods -l app=my-app -o jsonpath='{.items[*].metadata.labels}'
# Селектор деплоймента
kubectl get deploy my-app -o jsonpath='{.spec.selector.matchLabels}'
# Селектор сервиса
kubectl get svc my-app -o jsonpath='{.spec.selector}'
```

---

## 6) Конфигурация и секреты: ConfigMap / Secret

```bash
# Создать ConfigMap из файла и literal-пары
kubectl create configmap app-config --from-file=app.properties --from-literal=MODE=prod

# Создать Secret (base64 кодируется автоматически)
kubectl create secret generic app-secret --from-literal=DB_PASS='s3cr3t'

# Использование в Pod/Deployment
# (смонтировать как файлы или передать через env)
```

Фрагмент для env из ConfigMap/Secret:
```yaml
envFrom:
  - configMapRef:
      name: app-config
  - secretRef:
      name: app-secret
```

---

## 7) Частые повседневные команды (операции SRE/DevOps)

```bash
# Вывести все ресурсы ядра в ns
kubectl get all -n default

# Получить YAML любого объекта
kubectl get deploy my-app -o yaml

# Редактировать «на лету»
kubectl edit deploy my-app

# Точечный патч
kubectl patch deploy my-app -p '{"spec":{"replicas":5}}'

# Лейблы/аннотации
kubectl label   pod my-app env=prod --overwrite
kubectl annotate pod my-app commit=abc123 --overwrite

# Удаление
kubectl delete pod/my-app
kubectl delete -f deployment.yaml

# Профилирование «ресурсов» (при наличии metrics-server)
kubectl top nodes
kubectl top pods -n default

# Контекст/конфиг kubectl
kubectl config get-contexts
kubectl config use-context my-cluster
kubectl config set-context --current --namespace=test-env
```

---

## 8) Быстрые шаблоны команд (one-liners)

```bash
# IP всех подов по селектору
kubectl get pod -l app=my-app -o jsonpath='{range .items[*]}{.status.podIP}{"
"}{end}'

# ClusterIP и порты сервиса
kubectl get svc my-app -o jsonpath='{.spec.clusterIP}:{range .spec.ports[*]}{.port}{" "}{end}{"
"}'

# Проверка готовности всех подов деплоймента
kubectl get rs -l app=my-app -o jsonpath='{range .items[*]}{.status.readyReplicas}{" / "}{.status.replicas}{"
"}{end}'

# Быстрая раскатка с заменой образа
kubectl set image deploy/my-app app=nginx:1.27.1 && kubectl rollout status deploy/my-app

# Запустить временный debug-под
kubectl run -it --rm debug --image=busybox -- sh
```

---

## 9) Полезные дополнения

- `kubectl explain <resource> --recursive` — структура и описание полей.
- `kubectl api-resources` — список доступных ресурсов и их скоуп.
- Для локальной отладки сервисов без NodePort/LoadBalancer используйте `kubectl port-forward`.
- Для stateful-приложений: рассмотрите `StatefulSet + Headless Service`.
- Сетевые политики: `NetworkPolicy` для изоляции трафика между Pod’ами.

---

### Минимальный набор манифестов для работающего приложения
```yaml
# 1) Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: app-prod
---
# 2) Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: app-prod
spec:
  replicas: 3
  selector:
    matchLabels: { app: web }
  template:
    metadata:
      labels: { app: web }
    spec:
      containers:
        - name: web
          image: nginx:1.27
          ports: [{ containerPort: 8080 }]
---
# 3) Service (ClusterIP)
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: app-prod
spec:
  selector: { app: web }
  ports:
    - port: 80
      targetPort: 8080
```
```bash
kubectl apply -f app-bundle.yaml
kubectl -n app-prod get deploy,svc,pods -o wide
kubectl -n app-prod port-forward svc/web 8080:80
```

---

**Подсказка:** если «Service не работает», в 90% случаев проблема в _несоответствии меток/селекторов_, пустых `endpoints` или в том, что приложение внутри Pod не слушает `targetPort`/`address 0.0.0.0`.
