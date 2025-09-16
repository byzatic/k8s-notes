# Docker build в GitLab CI на Kubernetes‑runner: `Cannot connect to the Docker daemon` — причины и исправление

## Резюме
В job‑контейнере GitLab CI (Kubernetes executor) **нет локального Docker‑демона** и нет сокета `/var/run/docker.sock`, поэтому вызов `docker build` по умолчанию завершается ошибкой:
```
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```
Базовое решение — **запуск sidecar‑сервиса `docker:dind`** (Docker‑in‑Docker) и перенаправление клиента `docker` на TCP‑endpoint `DOCKER_HOST`.

---

## Почему это происходит
- На Kubernetes‑runner’e job запускается в изолированном Pod’е без встроенного Docker‑демона.
- Сокет `unix:///var/run/docker.sock` отсутствует, так как он принадлежит **ноде**, а не Pod’у.
- Подключать host‑socket в Pod (hostPath) небезопасно и обычно **запрещено политиками**.

---

## Быстрое исправление (DinD sidecar)

### .gitlab-ci.yml (рабочий пример)
```yaml
docker-build:
  stage: build
  image: docker:24-cli            # CLI докера внутри job-контейнера
  tags: [k8s]                     # тег Kubernetes-runner'а
  services:
    - name: docker:24-dind        # sidecar Docker daemon
      command:
        - --tls=false
        - --host=tcp://0.0.0.0:2375
        - --insecure-registry=<registry.company.com ip>:5000
        - --insecure-registry=<gitlab.company.com ip>:5555
  variables:
    DOCKER_HOST: "tcp://docker:2375"  # 'docker' — DNS-алиас сервиса в GitLab CI
    DOCKER_TLS_CERTDIR: ""            # отключаем TLS-автоконфиг докера
    DOCKER_DRIVER: "overlay2"
    DOCKER_BUILDKIT: "1"             # ускоряет сборку (BuildKit)
    DOCKERFILE_PATH: "./Dockerfile"
    BUILD_CONTEXT: "."
  before_script:
    - unset DOCKER_CONTEXT || true
    - docker context use default 2>/dev/null || true
    - docker version
    - docker info
  script:
    - echo -n "$CI_JOB_TOKEN" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
    - docker build -f "$DOCKERFILE_PATH" -t "$CI_REGISTRY_IMAGE:latest" "$BUILD_CONTEXT"
    - docker push "$CI_REGISTRY_IMAGE:latest"
```

### Требования к GitLab Runner’у (Kubernetes executor)
- Разрешить **privileged** для сервисов (DinD требуется привилегированный контейнер).
  - Helm‑chart `gitlab-runner`: `runners.privileged: true` или в `config.toml`:
    ```toml
    [runners.kubernetes]
      privileged = true
    ```
- Достаточно ephemeral storage на ноде для `/var/lib/docker` в sidecar.
- Сетевые политики должны пропускать трафик job ⇄ service внутри Pod’а/namespace.

### Проверка работоспособности
- В начале job `docker version` / `docker info` должны показывать подключение к `tcp://docker:2375`.
- `docker build` завершается успешно; `docker push` публикует образ в `$CI_REGISTRY_IMAGE`.
- При необходимости укажите `--build-arg` и `--platform` (arm64/amd64).

---

## Замечания по безопасности
- `--tls=false` и `DOCKER_TLS_CERTDIR=""` отключают TLS между CLI и daemon (коммуникация **внутри Pod’а**). Это приемлемо для CI‑sidecar, но не используйте такой endpoint извне.
- `--insecure-registry` раскрывает push/pull без проверки TLS к указанным адресам. Лучше настроить доверенный CA в registry и использовать HTTPS.
- CI‑секреты:
  - Для GitLab Container Registry используйте `$CI_JOB_TOKEN` + `$CI_REGISTRY_USER`.
  - Для внешних регистри — в переменную `DOCKER_AUTH_CONFIG` (JSON) или masked‑переменные.
- Не монтируйте host‑`/var/run/docker.sock` в Pod: это предоставляет чрезмерные привилегии.

---

## Производительность и кэш
- **BuildKit** (`DOCKER_BUILDKIT=1`) ускоряет сборку и поддерживает кэширование.
- Кэш между job’ами при DinD эфемерен. Варианты улучшения:
  - Использовать `--cache-from`/`--cache-to` с registry:
    ```bash
    docker build       --build-arg BUILDKIT_INLINE_CACHE=1       --build-arg CACHEBUST=$CI_COMMIT_SHA       -t $CI_REGISTRY_IMAGE:latest       -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA       -f "$DOCKERFILE_PATH" "$BUILD_CONTEXT"
    # затем пушить оба тега; следующий build сможет тянуть слои
    ```
  - GitLab Cache/Artifacts — хранить зависимости (например, директорию `~/.cache` менеджера пакетов).

---

## Типичные ошибки и их причины
| Симптом | Причина | Исправление |
|---|---|---|
| `Cannot connect to the Docker daemon at unix:///var/run/docker.sock` | Нет локального демона и сокета | Добавить сервис `docker:dind` и `DOCKER_HOST=tcp://docker:2375` |
| `client is newer than server` | Версии CLI/daemon не совпадают | Синхронизировать версии (`docker:24-cli` + `docker:24-dind`) |
| `error creating overlay mount` | Недостаточно места/подсистема overlay2 недоступна | Увеличить ephemeral storage ноды; проверить kernel/overlay2 |
| `unauthorized: HTTP Basic: Access denied` | Неверный логин в registry | Для GitLab — `echo -n "$CI_JOB_TOKEN" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"` |
| Медленная сборка без кэша | Эфемерность DinD | Включить BuildKit, использовать cache-from/cache-to с registry |

---

## Альтернативы DinD (когда privileged запрещён)
1. **Kaniko** — сборка образов из Dockerfile без демона Docker.
   ```yaml
   kaniko-build:
     stage: build
     image: gcr.io/kaniko-project/executor:latest
     variables:
       DOCKER_CONFIG: /kaniko/.docker
     script:
       - mkdir -p /kaniko/.docker
       - |
         cat > /kaniko/.docker/config.json <<'JSON'
         {"auths":{"$CI_REGISTRY":{"username":"$CI_REGISTRY_USER","password":"$CI_JOB_TOKEN"}}}
         JSON
       - /kaniko/executor            --context "$CI_PROJECT_DIR"            --dockerfile "$CI_PROJECT_DIR/Dockerfile"            --destination "$CI_REGISTRY_IMAGE:latest"            --snapshotMode=redo
   ```
2. **Buildah/Podman** (rootless) — требует соответствующие образы и разрешения user‑namespaces.
3. **Remote builder** — отдельный BuildKit/runner вне кластера; job шлёт команды на внешний endpoint по TLS.

**Рекомендация**: Если политика безопасности кластера строгая — используйте **Kaniko**. Если допустим privileged и нужна совместимость с Docker‑фичами — **DinD**.

---

## Чек‑лист админа Kubernetes‑runner’а
- `runners.kubernetes.privileged=true` для сервисов DinD.
- Квоты/лимиты на ephemeral storage и CPU/RAM для job и для sidecar.
- Сетевые политики разрешают связь job ↔ service `docker`.
- Registry доступен из кластера; при частном CA — добавьте доверие.
- Теги runner’а (`tags: [k8s]`) соответствуют job.

---

## Мини‑FAQ
**Можно ли подключить host‑docker.sock к job?**  
Технически возможно через hostPath + privileged, но крайне **не рекомендуется** из‑за рисков эскалации привилегий.

**Нужен ли TLS между job и DinD?**  
Внутри одного Pod/namespace — не обязательно. Для внешних соединений — только по TLS и с аутентификацией.

**Как ускорить сборку?**  
Включить BuildKit, использовать кэш через реестр, оптимизировать Dockerfile (слои, порядок RUN, multi‑stage).

---

## Итог
Ошибка возникает из‑за отсутствия докер‑демона в job‑контейнере. Добавьте sidecar `docker:dind` и направьте `docker` на `DOCKER_HOST=tcp://docker:2375`, либо используйте бездемонные билдеры (Kaniko). Учитывайте требования безопасности и конфигурацию runner’а.
