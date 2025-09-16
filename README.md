# Индекс документации (краткие аннотации)

Для каждого файла приведены короткое описание и несколько ключевых команд/фрагментов, если они встречаются в тексте.

## [Auto‑scalable-Kubernetes‑based-GitLab-Runner.md](.docs%2FAuto%E2%80%91scalable-Kubernetes%E2%80%91based-GitLab-Runner.md)
**Заголовок:** GitLab Runner на Kubernetes: автоскейл и изоляция job (1 job = 1 Pod)
**Краткое описание:** Развёртывание GitLab Runner (Helm) с Kubernetes executor: автоскейл менеджеров и pod-ов под задачи, приватные реестры (CA/секреты), кэш/артефакты в MinIO/S3, сборки через docker:dind или бездемонные билдеры (kaniko/buildkit), метрики Prometheus, корректные RBAC/ServiceAccount, nodeSelector/tolerations/affinity, (опц.) HPA.
**Ключевые команды/фрагменты:**
- helm repo add gitlab https://charts.gitlab.io && helm repo update
- helm upgrade --install gitlab-runner gitlab/gitlab-runner -n ci --create-namespace -f values.yaml
- values.yaml: gitlabUrl, runnerRegistrationToken|runnerToken, runners.privileged: true
- values.yaml: runners.kubernetes: serviceAccountName, imagePullSecrets, namespace, pollTimeout
- values.yaml: cache.s3: serverAddress, bucketName, accessKey, secretKey, insecure: true  # MinIO
- values.yaml: runners.config (TOML): concurrent, check_interval, [[runners.kubernetes]]
- docker:dind: volumes для /var/lib/docker, --mtu при необходимости; альтернатива — kaniko/buildkit
- metrics.enabled: true  # включить /metrics у раннера для Prometheus
- RBAC/SA: rbac.create: true, serviceAccount.create: true | serviceAccount.name: ...
- (опц.) HPA: hpa.enabled: true, hpa.minReplicas, hpa.maxReplicas, hpa.metrics

## [docker-build-on-k8s-runner.md](.docs%2Fdocker-build-on-k8s-runner.md)
**Заголовок:** Docker build в GitLab CI на Kubernetes‑runner: `Cannot connect to the Docker daemon` — причины и исправление
**Краткое описание:** Решение сборки образов в GitLab CI на Kubernetes‑runner через sidecar `docker:dind` и переменную `DOCKER_HOST`. Исправления/настройки GitLab Runner (Kubernetes executor, privileged, storage, сеть).
**Ключевые команды/фрагменты:**
- `…см. YAML в документе…`
- `docker build       --build-arg BUILDKIT_INLINE_CACHE=1       --build-arg CACHEBUST=$CI_COMMIT_SHA       -t $CI_REGISTRY_IMAGE:latest       -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA       -f "$DOCKERFILE_PATH" "$BUILD_CONTEXT"`
- `…см. YAML в документе…`

## [gitlab-ci-cache-check.md](.docs%2Fgitlab-ci-cache-check.md)
**Заголовок:** GitLab CI: проверка работы кэша (Kubernetes runner)
**Краткое описание:** Проверка GitLab CI Cache: публикация кэша в build и восстановление в test, ключи и пути кэша. Исправления/настройки GitLab Runner (Kubernetes executor, privileged, storage, сеть).
**Ключевые команды/фрагменты:**
- `…см. YAML в документе…`

## [gitlab-runner-fix.md](.docs%2Fgitlab-runner-fix.md)
**Заголовок:** GitLab Runner (Kubernetes, CRI‑O) — README по исправлениям
**Краткое описание:** RBAC‑настройки ролей и привязок для сервисных аккаунтов и GitLab Runner. Копирование/переименование Secret через `kubectl get -o json | jq | kubectl apply -f -`.
**Ключевые команды/фрагменты:**
- `kubectl -n ci-system create secret generic gitlab-ca \`
- `…см. YAML в документе…`
- `…см. YAML в документе…`
- `kubectl -n ci-jobs create role gitlab-runner-manager \`

## [gitlab-runner-rbac.md](.docs%2Fgitlab-runner-rbac.md)
**Заголовок:** RBAC: Права для GitLab Runner в Kubernetes
**Краткое описание:** RBAC‑настройки ролей и привязок для сервисных аккаунтов и GitLab Runner. Исправления/настройки GitLab Runner (Kubernetes executor, privileged, storage, сеть).
**Ключевые команды/фрагменты:**
- `kubectl -n ci-jobs create role gitlab-runner-manager   --verb=get,list,watch,create,update,patch,delete   --resource=pods,pods/log,pods/exec,secrets,configmaps,services,events`
- `kubectl -n ci-jobs create rolebinding gitlab-runner-manager-binding   --role=gitlab-runner-manager   --serviceaccount=ci-system:gitlab-runner`
- `kubectl -n ci-jobs get role gitlab-runner-manager -o yaml`
- `kubectl -n ci-jobs get rolebinding gitlab-runner-manager-binding -o yaml`

## [k8s-basic-concepts.md](.docs%2Fk8s-basic-concepts.md)
**Заголовок:** Базовые концепции Kubernetes — Академическое изложение
**Краткое описание:** RBAC‑настройки ролей и привязок для сервисных аккаунтов и GitLab Runner. Фундаментальные концепции и диагностика Control Plane (apiserver, scheduler, controller‑manager, etcd).

## [k8s-cheatsheet-namespace-pod-deployment-service.md](.docs%2Fk8s-cheatsheet-namespace-pod-deployment-service.md)
**Заголовок:** Kubernetes: Namespace · Pod · Service · Deployment — Практическая шпаргалка
**Краткое описание:** Установка и отладка `metrics-server` для команд `kubectl top` и регистрации `metrics.k8s.io`. Шпаргалка по Namespace/Pod/Deployment/Service с командами `kubectl` и YAML‑примером.
**Ключевые команды/фрагменты:**
- `kubectl create namespace test-env`
- `kubectl get ns`
- `kubectl get pods -n test-env`
- `kubectl get all  -n test-env`

## [k8s-control-plane.md](.docs%2Fk8s-control-plane.md)
**Заголовок:** Kubernetes Control Plane — академическое введение и практическое руководство
**Краткое описание:** RBAC‑настройки ролей и привязок для сервисных аккаунтов и GitLab Runner. Фундаментальные концепции и диагностика Control Plane (apiserver, scheduler, controller‑manager, etcd).
**Ключевые команды/фрагменты:**
- `kubectl -n kube-system get pods -o wide | egrep 'kube-(api|controller|scheduler)'`
- `kubectl -n kube-system describe pod -l component=kube-apiserver`
- `kubectl -n kube-system logs -l component=kube-apiserver --tail=200`
- `kubectl -n kube-system logs -l component=etcd --tail=200`

## [k8s-metrics-server.md](.docs%2Fk8s-metrics-server.md)
**Заголовок:** metrics-server — установка и устранение неполадок (сводка по диалогу)
**Краткое описание:** RBAC‑настройки ролей и привязок для сервисных аккаунтов и GitLab Runner. Установка и отладка `metrics-server` для команд `kubectl top` и регистрации `metrics.k8s.io`.
**Ключевые команды/фрагменты:**
- `kubectl delete -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml --ignore-not-found`
- `kubectl -n kube-system delete deploy/metrics-server svc/metrics-server sa/metrics-server --ignore-not-found`
- `kubectl delete clusterrolebinding metrics-server:system:auth-delegator system:metrics-server --ignore-not-found`
- `kubectl delete apiservice v1beta1.metrics.k8s.io --ignore-not-found`

## [k8s-nodes.md](.docs%2Fk8s-nodes.md)
**Заголовок:** Kubernetes Nodes — академическое введение и практическое руководство
**Краткое описание:** Фундаментальные концепции и диагностика Control Plane (apiserver, scheduler, controller‑manager, etcd). Установка и отладка `metrics-server` для команд `kubectl top` и регистрации `metrics.k8s.io`.
**Ключевые команды/фрагменты:**
- `kubectl get nodes`
- `kubectl get nodes -o wide`
- `kubectl get nodes --show-labels`
- `kubectl get nodes -L node-role.kubernetes.io/control-plane -o wide`

## [k8s-rbac.md](.docs%2Fk8s-rbac.md)k8s-rbac.md
**Заголовок:** RBAC в Kubernetes — практическое руководство
**Краткое описание:** RBAC‑настройки ролей и привязок для сервисных аккаунтов и GitLab Runner. Шпаргалка по Namespace/Pod/Deployment/Service с командами `kubectl` и YAML‑примером.
**Ключевые команды/фрагменты:**
- `…см. YAML в документе…`
- `…см. YAML в документе…`
- `…см. YAML в документе…`
- `kubectl -n ci-jobs create role gitlab-runner-manager   --verb=get,list,watch,create,update,patch,delete   --resource=pods,pods/log,pods/exec,secrets,configmaps,services,events`

## [minio-gitlab-runner-cache.md](.docs%2Fminio-gitlab-runner-cache.md)
**Заголовок:** GitLab Runner Cache в MinIO/S3 (Kubernetes executor) — настройки и проверка  
**Краткое описание:** Настройка кэша GitLab Runner в MinIO/S3: параметры `values.yaml` (`cache.s3.serverAddress/bucketName/accessKey/secretKey/insecure/pathStyleAccess`), предварительное создание бакета, секреты доступа, особенности self‑signed TLS/CA и проверка через `mc`/`awscli`. Рекомендации по ключам и путям кэша в `.gitlab-ci.yml`.
**Ключевые команды/фрагменты:**
- `values.yaml: cache.type: s3, cache.path: gitlab-runner`
- `values.yaml: cache.s3: serverAddress: minio.minio.svc.cluster.local:9000, bucketName: runner-cache, accessKey/secretKey, insecure: true, pathStyleAccess: true`
- `helm upgrade --install gitlab-runner gitlab/gitlab-runner -n ci --create-namespace -f values.yaml`
- `kubectl -n ci create secret generic minio-s3-credentials --from-literal=accesskey=$MINIO_ACCESS_KEY --from-literal=secretkey=$MINIO_SECRET_KEY   # (если используете секрет вместо явных значений)`
- `mc alias set local http://$MINIO_HOST $MINIO_ACCESS_KEY $MINIO_SECRET_KEY --api S3v4 && mc mb -p local/runner-cache || true`
- `.gitlab-ci.yml → cache: key: "$CI_PROJECT_NAME-$CI_COMMIT_REF_SLUG"; paths: [.m2/repository, .gradle, node_modules, target]`
- `…см. YAML в документе…`

## [k8s-secret-copy.md](.docs%2Fk8s-secret-copy.md)
**Заголовок:** Kubernetes: копирование/репликация Secret между неймспейсами и кластерами  
**Краткое описание:** Практики переноса Secret: через `kubectl + jq/yq` с очисткой метаданных, создание заново (`docker-registry`), перенос по label‑selector, и безопасные альтернативы (`SealedSecrets`, `SOPS`). Особенности типов (`kubernetes.io/dockerconfigjson`, `Opaque`) и кросс‑кластерного копирования.
**Ключевые команды/фрагменты:**
- `kubectl -n src get secret my-secret -o json | jq 'del(.metadata.uid,.metadata.resourceVersion,.metadata.creationTimestamp,.metadata.managedFields) | .metadata.namespace="dst"' | kubectl -n dst apply -f -`
- `kubectl --context=src -n src get secret my-secret -o yaml | yq 'del(.metadata.*) | .metadata.namespace="dst"' > my-secret.yaml && kubectl --context=dst -n dst apply -f my-secret.yaml`
- `kubectl -n dst create secret docker-registry regcred --docker-server=$REG --docker-username=$USER --docker-password=$PASS --docker-email=$MAIL`
- `for s in $(kubectl -n src get secret -l app=myapp -o name); do kubectl -n src get $s -o json | jq 'del(.metadata.*) | .metadata.namespace="dst"' | kubectl -n dst apply -f -; done`
- `sealed-secrets: kubeseal --format=yaml < secret.yaml > sealedsecret.yaml  # перенос без раскрытия ключей`
- `…см. YAML/примеры в документе…`
