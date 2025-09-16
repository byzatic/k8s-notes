# MinIO пользователь и политика для GitLab Runner Cache (через `mc`)

Этот документ описывает, как создать отдельного пользователя в MinIO, выделить ему приватный бакет для кэша GitLab Runner и прикрепить минимальную политику доступа. **Важно:** в тексте и командах используется клиент **`mc`** (MinIO Client). Если у вас установлен бинарь `mcli`, создайте алиас `alias mc=mcli` или используйте `mcli` вместо `mc`.

---

## 1) Предпосылки

- Доступ к MinIO (адрес и root‑учётные данные администратора).
- Утилита `mc` установлена локально.
- Планируемые сущности:
  - **Пользователь:** `gitlab-runner`
  - **Бакет:** `gitlab-runner-cache`
  - **Политика:** `gitlab-cache-policy`
  - **MinIO alias:** `myminio`

> Заменяйте адрес/ключи на свои значения. В примерах показан HTTP‑эндпоинт `http://<s3.company.com ip>:9000`. Если у вас TLS, используйте `https://`.

---

## 2) Определить alias к MinIO

```bash
mc alias set myminio http://<MINIO_HOST>:9000 <MINIO_ADMIN_ACCESS_KEY> <MINIO_ADMIN_SECRET_KEY>
# пример:
# mc alias set myminio http://<s3.company.com ip>:9000 admin <ADMIN_SECRET>
```

Проверка:
```bash
mc admin info myminio
```

---

## 3) Создать пользователя для GitLab Runner

```bash
mc admin user add myminio gitlab-runner '<RUNNER_SECRET_KEY>'
```

> `gitlab-runner` — это **Access Key** пользователя для GitLab Runner; строка `'<RUNNER_SECRET_KEY>'` — его **Secret Key**. Сохраните их: они понадобятся раннеру для доступа к S3‑кэшу.

Проверка:
```bash
mc admin user info myminio gitlab-runner
mc admin user list myminio | grep gitlab-runner
```

---

## 4) Создать бакет и закрыть публичный доступ

```bash
mc mb myminio/gitlab-runner-cache
mc anonymous set none myminio/gitlab-runner-cache
```

Проверка:
```bash
mc ls myminio
mc ls myminio/gitlab-runner-cache
```

---

## 5) Создать минимальную политику и прикрепить её к пользователю

### 5.1. Подготовить JSON политики
```bash
cat >/tmp/gitlab-cache-policy.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": ["s3:ListBucket"], "Resource": ["arn:aws:s3:::gitlab-runner-cache"] },
    { "Effect": "Allow", "Action": ["s3:GetObject","s3:PutObject","s3:DeleteObject"], "Resource": ["arn:aws:s3:::gitlab-runner-cache/*"] }
  ]
}
JSON
```

### 5.2. Создать и прикрепить политику
```bash
mc admin policy create myminio gitlab-cache-policy /tmp/gitlab-cache-policy.json
mc admin policy attach myminio gitlab-cache-policy --user gitlab-runner
```

Проверка:
```bash
mc admin policy info myminio gitlab-cache-policy
```

---

## 6) Настройка GitLab Runner на использование MinIO как S3‑кэша

Вы можете настроить через **config.toml** или через **переменные окружения**/Helm‑значения.

### Вариант A: `config.toml` раннера
```toml
[runners]
  # … прочие настройки раннера …

[runners.cache]
  Type = "s3"
  Shared = true

[runners.cache.s3]
  ServerAddress = "<s3.company.com ip>:9000"      # без схемы; MinIO порт
  BucketName   = "gitlab-runner-cache"
  AccessKey    = "gitlab-runner"
  SecretKey    = "<RUNNER_SECRET_KEY>"
  Insecure     = true                        # true для HTTP; false для HTTPS
  # BucketLocation = "us-east-1"            # можно указать при необходимости
```

### Вариант B: переменные окружения / Helm (Kubernetes executor)
```yaml
# Фрагмент Deployment/Values (пример)
env:
  - name: RUNNER_CACHE_TYPE
    value: "s3"
  - name: RUNNER_CACHE_SHARED
    value: "true"
  - name: RUNNER_CACHE_S3_SERVER_ADDRESS
    value: "<s3.company.com ip>:9000"
  - name: RUNNER_CACHE_S3_BUCKET_NAME
    value: "gitlab-runner-cache"
  - name: RUNNER_CACHE_S3_ACCESS_KEY
    valueFrom:
      secretKeyRef:
        name: gitlab-runner-cache-secret
        key: access_key
  - name: RUNNER_CACHE_S3_SECRET_KEY
    valueFrom:
      secretKeyRef:
        name: gitlab-runner-cache-secret
        key: secret_key
  - name: RUNNER_CACHE_S3_INSECURE
    value: "true" # для HTTP; при HTTPS укажите "false"
```

> В Helm‑чарте GitLab Runner существуют соответствующие параметры `runners.cache.*` — используйте их, если вы управляете раннером через Helm.

---

## 7) Быстрая проверка кэша из GitLab CI

Простейший пайплайн для валидации (создаёт файл в кэше на первом job и читает на втором):

```yaml
stages: [build, test]

build:
  stage: build
  tags: [k8s]
  script:
    - mkdir -p .cache
    - date > .cache/poke.txt
    - ls -la .cache
  cache:
    key: "$CI_PROJECT_NAME"
    paths: [".cache/"]
    policy: push

test:
  stage: test
  tags: [k8s]
  script:
    - ls -la .cache || true
    - cat .cache/poke.txt
  cache:
    key: "$CI_PROJECT_NAME"
    paths: [".cache/"]
    policy: pull
```

Если `test` читает `poke.txt`, кэш настроен корректно.

---

## 8) Частые ошибки и советы

| Симптом | Причина | Что делать |
|---|---|---|
| `AccessDenied`/`Forbidden` | Нет прав у пользователя | Убедитесь, что политика прикреплена к `gitlab-runner` и разрешает List/Get/Put/Delete для бакета. |
| `x509: certificate signed by unknown authority` | HTTP↔HTTPS несогласованность/самоподписанный CA | Для HTTP укажите `Insecure=true`. Для HTTPS добавьте доверенный CA в раннер или используйте официальный сертификат. |
| Кэш не восстанавливается | Разные `cache.key`/пути или первый прогон | Используйте одинаковый `key` и `paths`. Первый прогон только публикует кэш. |
| `mc: command not found` | Клиент не установлен/назван по‑другому | Установите MinIO Client или сделайте `alias mc=mcli`. |

---

## 9) Замечания по безопасности

- **Храните ключи** `gitlab-runner` в секретах CI/Kubernetes (masked/secret). Не коммитьте ключи в репозиторий.
- Делайте бакет приватным (`mc anonymous set none`), прикрепляйте **минимально необходимую** политику.
- Ограничьте сетевой доступ к MinIO только из нужных подсетей/namespace’ов.

---

## Итог

После выполнения шагов:
1) создан пользователь `gitlab-runner` с S3‑доступом только к бакету `gitlab-runner-cache`,  
2) бакет приватен,  
3) раннер настроен на использование этого бакета для кэширования артефактов между job’ами.
