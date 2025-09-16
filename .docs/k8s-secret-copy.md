# Копирование и переименование секретов с CA для разных volume в Kubernetes

## Назначение

В Kubernetes при монтировании секретов в Pod **нельзя использовать один и тот же секрет с одинаковым именем** для разных volume.  
Если попробовать сделать это, вы получите ошибку:

```
Pod "…" is invalid: [spec.volumes[3].name: Duplicate value: "..."]
```

Чтобы смонтировать один и тот же файл сертификата (`ca.crt`) в разные пути (например, `/etc/docker/certs.d/...` и `/etc/containers/certs.d/...`), необходимо:

1. Создать базовый секрет.
2. Скопировать его содержимое в два новых секрета с **разными именами**.
3. Монтировать эти разные секреты как отдельные volume.

---

## Примеры команд

### 1. Создание исходных секретов
```bash
kubectl -n ci-jobs create secret generic reg249-ca   --from-file=ca.crt=/home/axitech/KubernetesConfigs/gitlab-runner/SSL/reg249-ca/ca.crt

kubectl -n ci-jobs create secret generic reg94-ca   --from-file=ca.crt=/home/axitech/KubernetesConfigs/gitlab-runner/SSL/reg94-ca/ca.crt
```

### 2. Клонирование секрета с новым именем (для Docker)
```bash
kubectl -n ci-jobs get secret reg94-ca -o json | jq '.metadata.name="reg94-ca-docker" | del(.metadata.uid,.metadata.resourceVersion,.metadata.creationTimestamp,.metadata.namespace,.metadata.annotations,.metadata.managedFields)' | kubectl -n ci-jobs apply -f -
```

### 3. Клонирование секрета с другим именем (для Buildah/Podman)
```bash
kubectl -n ci-jobs get secret reg94-ca -o json | jq '.metadata.name="reg94-ca-containers" | del(.metadata.uid,.metadata.resourceVersion,.metadata.creationTimestamp,.metadata.namespace,.metadata.annotations,.metadata.managedFields)' | kubectl -n ci-jobs apply -f -
```

---

## Итог

- **Зачем:** Чтобы избежать конфликта имён volume при монтировании одного и того же секрета в разные директории.
- **Как:** Создаём один секрет → копируем его с разными именами → монтируем как разные volume.

Теперь можно безопасно использовать разные монтирования в одном Pod.
