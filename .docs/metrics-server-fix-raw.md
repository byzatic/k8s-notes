можно ли в кубире посмотреть загрузку по всем нодам

--R----R----

Да, в Kubernetes можно посмотреть загрузку CPU, памяти и других метрик по всем нодам, но нужно учитывать нюанс — сам kubectl не хранит метрики, он только показывает данные от подключенных компонентов.

Есть три основных способа:

⸻

1. Через metrics-server (kubectl top)

Если у тебя установлен metrics-server, можно выполнить:

kubectl top nodes

Пример вывода:

NAME             CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node1.internal   450m         12%    1024Mi          60%
node2.internal   300m          8%    512Mi           30%

	•	CPU(cores) — использование CPU в милликорах (m = 1/1000 ядра).
	•	CPU% — процент от доступных ресурсов.
	•	MEMORY(bytes) — использованная память.
	•	MEMORY% — процент от доступной памяти.

📌 Если kubectl top говорит, что metrics API not available — нужно поставить metrics-server:

kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml


⸻

2. Через kubelet summary API (без metrics-server)

Если metrics-server нет, можно напрямую запросить kubelet:

kubectl proxy &
curl http://127.0.0.1:8001/api/v1/nodes/<node_name>/proxy/stats/summary

Там будет JSON с данными по CPU, памяти и дискам.

⸻

3. Через Prometheus / Grafana

В production лучше настроить мониторинг через Prometheus и Grafana — тогда можно:
	•	Смотреть загрузку всех нод во времени.
	•	Получать алерты при превышении порогов.
	•	Строить дашборды.

Если у тебя уже есть Prometheus в кластере:

node_cpu_seconds_total
node_memory_MemAvailable_bytes

и т.п. помогут вывести нагрузку.

⸻

Если хочешь, я могу прямо сейчас сделать тебе одну команду, которая сразу выведет загрузку CPU и памяти по всем нодам в таблице, без установки Grafana. Это удобно для быстрой диагностики.
Хочешь, сделаю?



--R----R----




root@node2:/home/axitech/KubernetesConfigs/gitlab-runner# kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
root@node2:/home/axitech/KubernetesConfigs/gitlab-runner# kubectl top nodes
error: Metrics API not available




--R----R----

Классика: metrics‑server встал, но не может опросить kubelet’ы (обычно из‑за самоподписанных сертификатов и/или адресов нод). Починим за пару шагов.

1) Быстрый диагноз

Посмотри, что он пишет и статус APIService:

# логи metrics-server
kubectl -n kube-system logs deploy/metrics-server

# статус APIService
kubectl get apiservices | grep metrics
kubectl describe apiservice v1beta1.metrics.k8s.io

Если увидишь что-то вроде x509: certificate signed by unknown authority или context deadline exceeded — это оно.

2) Починка через параметры к метрик‑серверу

Чаще всего хватает двух флагов:
	•	--kubelet-insecure-tls — не проверять TLS у kubelet (быстро и просто).
	•	--kubelet-preferred-address-types=InternalIP,Hostname,ExternalIP — опрашивать ноды по их InternalIP.

Применить одной командой:

kubectl -n kube-system set args deploy/metrics-server --containers=metrics-server -- \
  --secure-port=4443 \
  --kubelet-insecure-tls \
  --kubelet-preferred-address-types=InternalIP,Hostname,ExternalIP \
  --metric-resolution=30s

Подождать, пока под перезапустится:

kubectl -n kube-system rollout status deploy/metrics-server

3) Проверка

kubectl top nodes
kubectl top pods -A

Если всё ещё «Metrics API not available», пришли, пожалуйста, вывод:
	•	kubectl -n kube-system logs deploy/metrics-server
	•	kubectl describe apiservice v1beta1.metrics.k8s.io

(Иногда в изолированных средах ещё включают hostNetwork: true у metrics‑server и dnsPolicy: ClusterFirstWithHostNet, но в большинстве случаев двух флагов выше достаточно.)



--R----R----



E0812 06:49:49.735558       1 scraper.go:149] "Failed to scrape node" err="Get \"https://<k8s node 2 ip>:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for <k8s node 2 ip> because it doesn't contain any IP SANs" node="node2.internal"
E0812 06:49:49.738697       1 scraper.go:149] "Failed to scrape node" err="Get \"https://<k8s node 3 ip>:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for <k8s node 3 ip> because it doesn't contain any IP SANs" node="node3.internal"
E0812 06:49:49.741954       1 scraper.go:149] "Failed to scrape node" err="Get \"https://<k8s node 5 ip>:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for <k8s node 5 ip> because it doesn't contain any IP SANs" node="node5.internal"
I0812 06:49:50.126334       1 server.go:192] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
I0812 06:49:53.765309       1 server.go:192] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
I0812 06:50:03.773006       1 server.go:192] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
E0812 06:50:04.719693       1 scraper.go:149] "Failed to scrape node" err="Get \"https://<k8s node 1 ip>:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for <k8s node 1 ip> because it doesn't contain any IP SANs" node="node1.internal"
E0812 06:50:04.732440       1 scraper.go:149] "Failed to scrape node" err="Get \"https://<k8s node 3 ip>:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for <k8s node 3 ip> because it doesn't contain any IP SANs" node="node3.internal"
E0812 06:50:04.744630       1 scraper.go:149] "Failed to scrape node" err="Get \"https://<k8s node 6 ip>:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for <k8s node 6 ip> because it doesn't contain any IP SANs" node="node6.internal"
E0812 06:50:04.751121       1 scraper.go:149] "Failed to scrape node" err="Get \"https://<k8s node 5 ip>:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for <k8s node 5 ip> because it doesn't contain any IP SANs" node="node5.internal"
E0812 06:50:04.755214       1 scraper.go:149] "Failed to scrape node" err="Get \"https://<k8s node 4 ip>:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for <k8s node 4 ip> because it doesn't contain any IP SANs" node="node4.internal"
E0812 06:50:04.756305       1 scraper.go:149] "Failed to scrape node" err="Get \"https://<k8s node 2 ip>:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for <k8s node 2 ip> because it doesn't contain any IP SANs" node="node2.internal"
I0812 06:50:13.769972       1 server.go:192] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
E0812 06:50:19.717515       1 scraper.go:149] "Failed to scrape node" err="Get \"https://<k8s node 4 ip>:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for <k8s node 4 ip> because it doesn't contain any IP SANs" node="node4.internal"
E0812 06:50:19.734155       1 scraper.go:149] "Failed to scrape node" err="Get \"https://<k8s node 6 ip>:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for <k8s node 6 ip> because it doesn't contain any IP SANs" node="node6.internal"
E0812 06:50:19.738521       1 scraper.go:149] "Failed to scrape node" err="Get \"https://<k8s node 2 ip>:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for <k8s node 2 ip> because it doesn't contain any IP SANs" node="node2.internal"
E0812 06:50:19.741022       1 scraper.go:149] "Failed to scrape node" err="Get \"https://<k8s node 5 ip>:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for <k8s node 5 ip> because it doesn't contain any IP SANs" node="node5.internal"
E0812 06:50:19.745680       1 scraper.go:149] "Failed to scrape node" err="Get \"https://<k8s node 1 ip>:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for <k8s node 1 ip> because it doesn't contain any IP SANs" node="node1.internal"
E0812 06:50:19.759627       1 scraper.go:149] "Failed to scrape node" err="Get \"https://<k8s node 3 ip>:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for <k8s node 3 ip> because it doesn't contain any IP SANs" node="node3.internal"



--R----R----


Отличные логи — всё по делу: metrics‑server не доверяет kubelet serving certs, потому что в сертификатах kubelet нет IP‑SAN (ошибка буквально это и говорит). Есть два пути:

Быстрый рабочий фикс (1–2 минуты)

Скажем metrics‑server игнорировать верификацию TLS у kubelet и опрашивать их по InternalIP.

# добавить нужные флаги контейнеру metrics-server
kubectl -n kube-system set args deploy/metrics-server --containers=metrics-server -- \
  --secure-port=4443 \
  --kubelet-insecure-tls \
  --kubelet-preferred-address-types=InternalIP,Hostname,ExternalIP \
  --metric-resolution=30s

# дождаться перезапуска
kubectl -n kube-system rollout status deploy/metrics-server

# проверить
kubectl top nodes
kubectl top pods -A

Обычно этого достаточно, и kubectl top начнёт работать через ~10–30 секунд.

Если сеть/кластер специфичны и всё ещё не ходит — иногда помогает включить hostNetwork:

kubectl -n kube-system patch deploy metrics-server -p '{
  "spec":{"template":{"spec":{
    "hostNetwork": true,
    "dnsPolicy": "ClusterFirstWithHostNet"
  }}}}'

Но чаще это не требуется.

⸻

Правильный долгосрочный вариант (без --kubelet-insecure-tls)

Сгенерировать/починить kubelet serving certificates, чтобы у них были SAN с IP (InternalIP) и/или корректные DNS‑имена нод. Опции зависят от того, как у тебя поднят кластер:
	•	kubeadm: убедиться, что включена ротация serving‑сертов (RotateKubeletServerCertificate: true в kubelet-config), и что CSR‑сайнёр выдаёт сертификаты с нужными SAN. После — перезапуск kubelet/ожидание ротации.
	•	non‑kubeadm / кастом: указать kubelet флаги --tls-cert-file/--tls-private-key-file с сертификатом, где есть IP: <InternalIP> в SAN, либо обеспечить корректные DNS‑имена и опрашивать их.

Пока SAN не поправишь, безопаснее оставить --kubelet-insecure-tls только на metrics‑server (это локальная доверительная связь внутри кластера).

⸻

Почему именно такая ошибка

metrics‑server ходит на kubelet по https://<InternalIP>:10250/metrics/resource. Когда в сертификате kubelet нет IP SAN, стандартная проверка TLS не может сопоставить сертификат с адресом → cannot validate certificate for <IP> because it doesn't contain any IP SANs.

⸻

Если хочешь, могу:
	•	проверить текущие args/манифест metrics-server (через kubectl -n kube-system get deploy metrics-server -o yaml) и дать точный патч;
	•	подсказать, как именно поправить SAN’ы для kubelet под твой способ развёртывания (kubeadm/ручной).


--R----R----

root@node2:/home/axitech/KubernetesConfigs/gitlab-runner# kubectl -n kube-system set args deploy/metrics-server --containers=metrics-server -- \
  --secure-port=4443 \
  --kubelet-insecure-tls \
  --kubelet-preferred-address-types=InternalIP,Hostname,ExternalIP \
  --metric-resolution=30s
error: unknown flag: --containers
See 'kubectl set --help' for usage.

--R----R----
У тебя старая версия kubectl, в которой нет флага --containers у kubectl set args.
Сделаем это через kubectl patch или через редактирование деплоймента.

Вариант 1 — patch (быстро)

kubectl -n kube-system patch deployment metrics-server \
  --type='json' \
  -p='[
    {"op":"replace","path":"/spec/template/spec/containers/0/args",
     "value":[
       "--secure-port=4443",
       "--kubelet-insecure-tls",
       "--kubelet-preferred-address-types=InternalIP,Hostname,ExternalIP",
       "--metric-resolution=30s"
     ]
    }
  ]'

После этого:

kubectl -n kube-system rollout status deploy/metrics-server
kubectl top nodes


⸻

Вариант 2 — ручное редактирование (надёжно)

kubectl -n kube-system edit deploy metrics-server

В открывшемся YAML найди:

containers:
- name: metrics-server
  image: ...
  args:
    ...

и замени args на:

  args:
    - --secure-port=4443
    - --kubelet-insecure-tls
    - --kubelet-preferred-address-types=InternalIP,Hostname,ExternalIP
    - --metric-resolution=30s

Сохрани, выйди — kubectl сам перезапустит Pod’ы.

⸻

Хочешь, я сделаю тебе готовый YAML манифест metrics-server уже с этими флагами, чтобы просто kubectl apply -f и всё заработало? Так будет чище, чем ковырять деплой на лету.

--R----R----
root@node2:/home/axitech/KubernetesConfigs/gitlab-runner# kubectl -n kube-system rollout status deploy/metrics-server
Waiting for deployment "metrics-server" rollout to finish: 1 old replicas are pending termination...


--R----R----

Нормально, бывает: старый pod metrics‑server застрял на терминации — из‑за монтировок/проб/сети он не уходит, и rollout висит. Давай добьём аккуратно.

1) Посмотреть, что именно висит

kubectl -n kube-system get pods -l 'k8s-app=metrics-server,app!=metrics-server' -o wide || true
kubectl -n kube-system get pods -l 'app.kubernetes.io/name=metrics-server' -o wide || true
kubectl -n kube-system describe deploy/metrics-server
kubectl -n kube-system get rs -l app.kubernetes.io/name=metrics-server

2) Если видно старый pod в Terminating — снести его принудительно

# подставь имя старого пода из get pods
kubectl -n kube-system delete pod <old-metrics-pod> --force --grace-period=0

Потом снова:

kubectl -n kube-system rollout status deploy/metrics-server

3) Убедиться, что args применились и Service смотрит на правильный порт

kubectl -n kube-system get deploy metrics-server -o=jsonpath='{.spec.template.spec.containers[0].args}'; echo
kubectl -n kube-system get svc metrics-server -o yaml | grep -E 'port:|targetPort:'

Должно быть --secure-port=4443, и в сервисе targetPort: 4443.
Если targetPort другой — поправь:

kubectl -n kube-system patch svc metrics-server -p '{"spec":{"ports":[{"port":443,"targetPort":4443,"name":"https"}]}}'

4) Проверка логов и APIService

kubectl -n kube-system logs deploy/metrics-server --tail=100
kubectl get apiservices | grep metrics
kubectl describe apiservice v1beta1.metrics.k8s.io

Ошибок быть не должно (TLS мы отключили к kubelet через --kubelet-insecure-tls).

5) Финальная проверка

kubectl top nodes
kubectl top pods -A

Если опять повиснет «1 old replicas are pending termination…» — пришли kubectl -n kube-system get pods -l app.kubernetes.io/name=metrics-server -o wide и describe зависшего пода; подскажу точечную причину (иногда PDB/финалайзеры или сетевые webhooks держат).
--R----R----
root@node2:/home/axitech/KubernetesConfigs/gitlab-runner# kubectl -n kube-system get pods -l 'k8s-app=metrics-server,app!=metrics-server' -o wide || true
kubectl -n kube-system get pods -l 'app.kubernetes.io/name=metrics-server' -o wide || true
kubectl -n kube-system describe deploy/metrics-server
kubectl -n kube-system get rs -l app.kubernetes.io/name=metrics-server
NAME                              READY   STATUS             RESTARTS      AGE     IP             NODE             NOMINATED NODE   READINESS GATES
metrics-server-5dd7f7f59c-r57r8   0/1     Running            0             7m46s   10.244.3.58    node4.internal   <none>           <none>
metrics-server-7df8cd9c65-tdgmh   0/1     CrashLoopBackOff   4 (84s ago)   3m17s   10.244.8.221   node6.internal   <none>           <none>
No resources found in kube-system namespace.
Name:                   metrics-server
Namespace:              kube-system
CreationTimestamp:      Tue, 12 Aug 2025 06:48:32 +0000
Labels:                 k8s-app=metrics-server
Annotations:            deployment.kubernetes.io/revision: 2
Selector:               k8s-app=metrics-server
Replicas:               1 desired | 1 updated | 2 total | 0 available | 2 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  0 max unavailable, 25% max surge
Pod Template:
  Labels:           k8s-app=metrics-server
  Service Account:  metrics-server
  Containers:
   metrics-server:
    Image:           registry.k8s.io/metrics-server/metrics-server:v0.8.0
    Port:            10250/TCP
    Host Port:       0/TCP
    SeccompProfile:  RuntimeDefault
    Args:
      --secure-port=4443
      --kubelet-insecure-tls
      --kubelet-preferred-address-types=InternalIP,Hostname,ExternalIP
      --metric-resolution=30s
    Requests:
      cpu:        100m
      memory:     200Mi
    Liveness:     http-get https://:https/livez delay=0s timeout=1s period=10s #success=1 #failure=3
    Readiness:    http-get https://:https/readyz delay=20s timeout=1s period=10s #success=1 #failure=3
    Environment:  <none>
    Mounts:
      /tmp from tmp-dir (rw)
  Volumes:
   tmp-dir:
    Type:               EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:             
    SizeLimit:          <unset>
  Priority Class Name:  system-cluster-critical
  Node-Selectors:       kubernetes.io/os=linux
  Tolerations:          <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      False   MinimumReplicasUnavailable
  Progressing    True    ReplicaSetUpdated
OldReplicaSets:  metrics-server-5dd7f7f59c (1/1 replicas created)
NewReplicaSet:   metrics-server-7df8cd9c65 (1/1 replicas created)
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  7m47s  deployment-controller  Scaled up replica set metrics-server-5dd7f7f59c to 1
  Normal  ScalingReplicaSet  3m18s  deployment-controller  Scaled up replica set metrics-server-7df8cd9c65 to 1
No resources found in kube-system namespace.
root@node2:/home/axitech/KubernetesConfigs/gitlab-runner# 

--R----R----
Отличные логи — видно, что мы уже добавили нужные флаги, но у metrics-server сейчас несостыковка по порту контейнера.

В твоём Deployment у контейнера указано:
	•	Args: --secure-port=4443 (правильно),
	•	но Port: 10250/TCP и нет имени порта https.

Пробки (readiness/liveness) смотрят на порт с именем https, которого у контейнера нет → поды не проходят пробы → один в CrashLoopBackOff, другой 0/1.

Делаем два точных патча.

1) Поправить порт контейнера и дать ему имя https

kubectl -n kube-system patch deploy metrics-server --type='json' -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/ports",
   "value":[{"name":"https","containerPort":4443,"protocol":"TCP"}]
  }
]'

2) Убедиться, что Service бьёт в 4443

(иногда в манифесте уже так; если нет — патчим)

kubectl -n kube-system patch svc metrics-server -p '{
  "spec": {
    "ports": [{
      "name": "https",
      "port": 443,
      "targetPort": 4443,
      "protocol": "TCP"
    }]
  }
}'

3) Дождаться раскатки и проверить

kubectl -n kube-system rollout status deploy/metrics-server
kubectl -n kube-system get pods -l app.kubernetes.io/name=metrics-server -o wide
kubectl -n kube-system logs deploy/metrics-server --tail=100

Если всё ок, через 10–30 сек начнут приходить метрики:

kubectl top nodes
kubectl top pods -A

Если вдруг всё ещё ругнётся — кинь свежие logs деплоя metrics-server и describe apiservice v1beta1.metrics.k8s.io; но в 99% случаев эта пара патчей (порт контейнера + service targetPort) закрывает CrashLoop после смены --secure-port на 4443.

--R----R----
нет давай удалим и начнем сначала

--R----R----
Ок, сносим metrics‑server «под ноль» и ставим заново — сразу с правильными флагами и портами.

1) Удалить всё, что ставили из upstream‑манифеста

# Снести все объекты из components.yaml
kubectl delete -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml --ignore-not-found

# На всякий случай добить остатки (если что-то меняли вручную)
kubectl -n kube-system delete deploy/metrics-server svc/metrics-server sa/metrics-server --ignore-not-found
kubectl delete clusterrolebinding metrics-server:system:auth-delegator system:metrics-server --ignore-not-found
kubectl delete apiservice v1beta1.metrics.k8s.io --ignore-not-found

2) Поставить заново из upstream

kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

3) Сразу задать нужные флаги и порт контейнера

(в твоей среде kubelet’ы с самоподписанными сертами и без IP SAN → включаем --kubelet-insecure-tls, опрашиваем по InternalIP, и чётко задаём порт 4443)

# заменить args у контейнера на нужные
kubectl -n kube-system patch deploy metrics-server --type='json' -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/args",
   "value":[
     "--secure-port=4443",
     "--kubelet-insecure-tls",
     "--kubelet-preferred-address-types=InternalIP,Hostname,ExternalIP",
     "--metric-resolution=30s"
   ]
  },
  {"op":"replace","path":"/spec/template/spec/containers/0/ports",
   "value":[{"name":"https","containerPort":4443,"protocol":"TCP"}]
  }
]'

4) Убедиться, что Service смотрит на правильный порт

# таргет порт должен быть 4443
kubectl -n kube-system patch svc metrics-server -p '{
  "spec": {"ports":[{"name":"https","port":443,"targetPort":4443,"protocol":"TCP"}]}
}'

5) Дождаться раскатки и проверить

kubectl -n kube-system rollout status deploy/metrics-server
kubectl -n kube-system logs deploy/metrics-server --tail=100

kubectl get apiservices | grep metrics
kubectl describe apiservice v1beta1.metrics.k8s.io | sed -n '1,120p'

6) Использование

kubectl top nodes
kubectl top pods -A

Если вдруг снова упрётся, дай вывод:
	•	kubectl -n kube-system logs deploy/metrics-server --tail=100
	•	kubectl describe apiservice v1beta1.metrics.k8s.io

(Но с этой «чистой» установкой + патчами на args/порты в 99% случаев всё начинает работать через 10–30 секунд.)

--R----R----

root@node2:/home/axitech/KubernetesConfigs/gitlab-runner# kubectl -n kube-system rollout status deploy/metrics-server
Waiting for deployment "metrics-server" rollout to finish: 1 old replicas are pending termination...


--R----R----
Это значит, что старый Pod metrics‑server не может корректно завершиться. Давай быстро почистим и перезапустим.

Шаги «в лоб»
	1.	Посмотреть, что висит:

kubectl -n kube-system get rs -l k8s-app=metrics-server
kubectl -n kube-system get pods -l k8s-app=metrics-server -o wide

	2.	Убить старый Pod принудительно (подставь его имя из вывода выше):

kubectl -n kube-system delete pod <старый-pod> --force --grace-period=0

	3.	Если всё равно висит, сделаем «ресет» деплоймента:

kubectl -n kube-system scale deploy/metrics-server --replicas=0
kubectl -n kube-system wait --for=delete pod -l k8s-app=metrics-server --timeout=60s
kubectl -n kube-system scale deploy/metrics-server --replicas=1

	4.	Проверка раскатки и логов:

kubectl -n kube-system rollout status deploy/metrics-server
kubectl -n kube-system logs deploy/metrics-server --tail=100

	5.	Финальный тест:

kubectl top nodes
kubectl top pods -A

Если всё ещё не уходит

Иногда под держит финалайзер. Тогда:

POD=$(kubectl -n kube-system get pods -l k8s-app=metrics-server -o name | head -1)
kubectl -n kube-system get $POD -o json | sed 's/"finalizers": \[[^]]*\]/"finalizers": []/' | kubectl replace --raw /api/v1/namespaces/kube-system/$(echo $POD | cut -d/ -f1)s/$(echo $POD | cut -d/ -f2) -f -

и повторить пункт 3.

Если покажешь вывод get pods -o wide и get rs, подскажу конкретно какой именно pod/rs убрать.


--R----R----
root@node2:/home/axitech/KubernetesConfigs/gitlab-runner# kubectl -n kube-system patch deploy metrics-server --type='json' -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/ports",
   "value":[{"name":"https","containerPort":4443,"protocol":"TCP"}]
  }
]'
deployment.apps/metrics-server patched
root@node2:/home/axitech/KubernetesConfigs/gitlab-runner# kubectl -n kube-system patch svc metrics-server -p '{
  "spec": {
    "ports": [{
      "name": "https",
      "port": 443,
      "targetPort": 4443,
      "protocol": "TCP"
    }]
  }
}'
service/metrics-server patched
root@node2:/home/axitech/KubernetesConfigs/gitlab-runner# kubectl -n kube-system rollout status deploy/metrics-server
Waiting for deployment "metrics-server" rollout to finish: 1 old replicas are pending termination...
^Croot@node2:/home/axitech/KubernetesConfigs/gitlab-runner# kubectl -n kube-system rollout status deploy/metrics-server
Waiting for deployment "metrics-server" rollout to finish: 1 old replicas are pending termination...
^Croot@node2:/home/axitech/KubernetesConfigs/gitlab-runnerkubectl delete -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml --ignore-not-foundnd
serviceaccount "metrics-server" deleted
clusterrole.rbac.authorization.k8s.io "system:aggregated-metrics-reader" deleted
clusterrole.rbac.authorization.k8s.io "system:metrics-server" deleted
rolebinding.rbac.authorization.k8s.io "metrics-server-auth-reader" deleted
clusterrolebinding.rbac.authorization.k8s.io "metrics-server:system:auth-delegator" deleted
clusterrolebinding.rbac.authorization.k8s.io "system:metrics-server" deleted
service "metrics-server" deleted
deployment.apps "metrics-server" deleted
apiservice.apiregistration.k8s.io "v1beta1.metrics.k8s.io" deleted
root@node2:/home/axitech/KubernetesConfigs/gitlab-runner# kubectl -n kube-system delete deploy/metrics-server svc/metrics-server sa/metrics-server --ignore-not-found
kubectl delete clusterrolebinding metrics-server:system:auth-delegator system:metrics-server --ignore-not-found
kubectl delete apiservice v1beta1.metrics.k8s.io --ignore-not-found
root@node2:/home/axitech/KubernetesConfigs/gitlab-runner# kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
root@node2:/home/axitech/KubernetesConfigs/gitlab-runner# kubectl -n kube-system patch deploy metrics-server --type='json' -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/args",
   "value":[
     "--secure-port=4443",
     "--kubelet-insecure-tls",
     "--kubelet-preferred-address-types=InternalIP,Hostname,ExternalIP",
     "--metric-resolution=30s"
   ]
  },
  {"op":"replace","path":"/spec/template/spec/containers/0/ports",
   "value":[{"name":"https","containerPort":4443,"protocol":"TCP"}]
  }
]'
deployment.apps/metrics-server patched
root@node2:/home/axitech/KubernetesConfigs/gitlab-runner# kubectl -n kube-system patch svc metrics-server -p '{
  "spec": {"ports":[{"name":"https","port":443,"targetPort":4443,"protocol":"TCP"}]}
}'
service/metrics-server patched
root@node2:/home/axitech/KubernetesConfigs/gitlab-runner# kubectl -n kube-system rollout status deploy/metrics-server
Waiting for deployment "metrics-server" rollout to finish: 1 old replicas are pending termination...
^Croot@node2:/home/axitech/KubernetesConfigs/gitlab-runner# 
root@node2:/home/axitech/KubernetesConfigs/gitlab-runner# 
root@node2:/home/axitech/KubernetesConfigs/gitlab-runner# 
root@node2:/home/axitech/KubernetesConfigs/gitlab-runner# kubectl -n kube-system get rs -l k8s-app=metrics-server
NAME                        DESIRED   CURRENT   READY   AGE
metrics-server-5dd7f7f59c   1         1         0       91s
metrics-server-77fb6647cb   1         1         0       79s
root@node2:/home/axitech/KubernetesConfigs/gitlab-runner# kubectl -n kube-system get pods -l k8s-app=metrics-server -o wide
NAME                              READY   STATUS             RESTARTS      AGE   IP             NODE             NOMINATED NODE   READINESS GATES
metrics-server-5dd7f7f59c-qnhq8   0/1     Running            0             96s   10.244.8.223   node6.internal   <none>           <none>
metrics-server-77fb6647cb-t5jgw   0/1     CrashLoopBackOff   3 (39s ago)   85s   10.244.8.224   node6.internal   <none>           <none>
root@node2:/home/axitech/KubernetesConfigs/gitlab-runner# kubectl -n kube-system delete pod <старый-pod> --force --grace-period=0
-bash: старый-pod: No such file or directory
root@node2:/home/axitech/KubernetesConfigs/gitlab-runner# kubectl -n kube-system scale deploy/metrics-server --replicas=0
deployment.apps/metrics-server scaled
root@node2:/home/axitech/KubernetesConfigs/gitlab-runner# kubectl -n kube-system wait --for=delete pod -l k8s-app=metrics-server --timeout=60s
root@node2:/home/axitech/KubernetesConfigs/gitlab-runner# kubectl -n kube-system scale deploy/metrics-server --replicas=1
deployment.apps/metrics-server scaled
root@node2:/home/axitech/KubernetesConfigs/gitlab-runner# kubectl -n kube-system rollout status deploy/metrics-server
Waiting for deployment "metrics-server" rollout to finish: 0 of 1 updated replicas are available...
^Croot@node2:/home/axitech/KubernetesConfigs/gitlab-runnerkubectl -n kube-system logs deploy/metrics-server --tail=10000
      --kubelet-request-timeout duration          The length of time to wait before giving up on a single request to Kubelet. Non-zero values should contain a corresponding time unit (e.g. 1s, 2m, 3h). (default 10s)
      --kubelet-use-node-status-port              Use the port in the node status. Takes precedence over --kubelet-port flag.
  -l, --node-selector string                      Selector (label query) to filter on, not including uninitialized ones, supports '=', '==', and '!='.(e.g. -l key1=value1,key2=value2).

Apiserver secure serving flags:

      --bind-address ip                        The IP address on which to listen for the --secure-port port. The associated interface(s) must be reachable by the rest of the cluster, and by CLI/web clients. If blank or an unspecified address (0.0.0.0 or ::), all interfaces and IP address families will be used. (default 0.0.0.0)
      --cert-dir string                        The directory where the TLS certs are located. If --tls-cert-file and --tls-private-key-file are provided, this flag will be ignored. (default "apiserver.local.config/certificates")
      --disable-http2-serving                  If true, HTTP2 serving will be disabled [default=false]
      --http2-max-streams-per-connection int   The limit that the server gives to clients for the maximum number of streams in an HTTP/2 connection. Zero means to use golang's default.
      --permit-address-sharing                 If true, SO_REUSEADDR will be used when binding the port. This allows binding to wildcard IPs like 0.0.0.0 and specific IPs in parallel, and it avoids waiting for the kernel to release sockets in TIME_WAIT state. [default=false]
      --permit-port-sharing                    If true, SO_REUSEPORT will be used when binding the port, which allows more than one instance to bind on the same address and port. [default=false]
      --secure-port int                        The port on which to serve HTTPS with authentication and authorization. If 0, don't serve HTTPS at all. (default 443)
      --tls-cert-file string                   File containing the default x509 Certificate for HTTPS. (CA cert, if any, concatenated after server cert). If HTTPS serving is enabled, and --tls-cert-file and --tls-private-key-file are not provided, a self-signed certificate and key are generated for the public address and saved to the directory specified by --cert-dir.
      --tls-cipher-suites strings              Comma-separated list of cipher suites for the server. If omitted, the default Go cipher suites will be used. 
                                               Preferred values: TLS_AES_128_GCM_SHA256, TLS_AES_256_GCM_SHA384, TLS_CHACHA20_POLY1305_SHA256, TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA, TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256, TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA, TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384, TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305, TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256, TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA, TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256, TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA, TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384, TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305, TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256. 
                                               Insecure values: TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256, TLS_ECDHE_ECDSA_WITH_RC4_128_SHA, TLS_ECDHE_RSA_WITH_3DES_EDE_CBC_SHA, TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256, TLS_ECDHE_RSA_WITH_RC4_128_SHA, TLS_RSA_WITH_3DES_EDE_CBC_SHA, TLS_RSA_WITH_AES_128_CBC_SHA, TLS_RSA_WITH_AES_128_CBC_SHA256, TLS_RSA_WITH_AES_128_GCM_SHA256, TLS_RSA_WITH_AES_256_CBC_SHA, TLS_RSA_WITH_AES_256_GCM_SHA384, TLS_RSA_WITH_RC4_128_SHA.
      --tls-min-version string                 Minimum TLS version supported. Possible values: VersionTLS10, VersionTLS11, VersionTLS12, VersionTLS13
      --tls-private-key-file string            File containing the default x509 private key matching --tls-cert-file.
      --tls-sni-cert-key namedCertKey          A pair of x509 certificate and private key file paths, optionally suffixed with a list of domain patterns which are fully qualified domain names, possibly with prefixed wildcard segments. The domain patterns also allow IP addresses, but IPs should only be used if the apiserver has visibility to the IP address requested by a client. If no domain patterns are provided, the names of the certificate are extracted. Non-wildcard matches trump over wildcard matches, explicit domain patterns trump over extracted names. For multiple key/certificate pairs, use the --tls-sni-cert-key multiple times. Examples: "example.crt,example.key" or "foo.crt,foo.key:*.foo.com,foo.com". (default [])

Apiserver authentication flags:

      --authentication-kubeconfig string                  kubeconfig file pointing at the 'core' kubernetes server with enough rights to create tokenreviews.authentication.k8s.io.
      --authentication-skip-lookup                        If false, the authentication-kubeconfig will be used to lookup missing authentication configuration from the cluster.
      --authentication-token-webhook-cache-ttl duration   The duration to cache responses from the webhook token authenticator. (default 10s)
      --authentication-tolerate-lookup-failure            If true, failures to look up missing authentication configuration from the cluster are not considered fatal. Note that this can result in authentication that treats all requests as anonymous.
      --client-ca-file string                             If set, any request presenting a client certificate signed by one of the authorities in the client-ca-file is authenticated with an identity corresponding to the CommonName of the client certificate.
      --requestheader-allowed-names strings               List of client certificate common names to allow to provide usernames in headers specified by --requestheader-username-headers. If empty, any client certificate validated by the authorities in --requestheader-client-ca-file is allowed.
      --requestheader-client-ca-file string               Root certificate bundle to use to verify client certificates on incoming requests before trusting usernames in headers specified by --requestheader-username-headers. WARNING: generally do not depend on authorization being already done for incoming requests.
      --requestheader-extra-headers-prefix strings        List of request header prefixes to inspect. X-Remote-Extra- is suggested. (default [x-remote-extra-])
      --requestheader-group-headers strings               List of request headers to inspect for groups. X-Remote-Group is suggested. (default [x-remote-group])
      --requestheader-uid-headers strings                 List of request headers to inspect for UIDs. X-Remote-Uid is suggested. Requires the RemoteRequestHeaderUID feature to be enabled.
      --requestheader-username-headers strings            List of request headers to inspect for usernames. X-Remote-User is common. (default [x-remote-user])

Apiserver authorization flags:

      --authorization-always-allow-paths strings                A list of HTTP paths to skip during authorization, i.e. these are authorized without contacting the 'core' kubernetes server. (default [/healthz,/readyz,/livez])
      --authorization-kubeconfig string                         kubeconfig file pointing at the 'core' kubernetes server with enough rights to create subjectaccessreviews.authorization.k8s.io.
      --authorization-webhook-cache-authorized-ttl duration     The duration to cache 'authorized' responses from the webhook authorizer. (default 10s)
      --authorization-webhook-cache-unauthorized-ttl duration   The duration to cache 'unauthorized' responses from the webhook authorizer. (default 10s)

Apiserver audit log flags:

      --audit-log-batch-buffer-size int             The size of the buffer to store events before batching and writing. Only used in batch mode. (default 10000)
      --audit-log-batch-max-size int                The maximum size of a batch. Only used in batch mode. (default 1)
      --audit-log-batch-max-wait duration           The amount of time to wait before force writing the batch that hadn't reached the max size. Only used in batch mode.
      --audit-log-batch-throttle-burst int          Maximum number of requests sent at the same moment if ThrottleQPS was not utilized before. Only used in batch mode.
      --audit-log-batch-throttle-enable             Whether batching throttling is enabled. Only used in batch mode.
      --audit-log-batch-throttle-qps float32        Maximum average number of batches per second. Only used in batch mode.
      --audit-log-compress                          If set, the rotated log files will be compressed using gzip.
      --audit-log-format string                     Format of saved audits. "legacy" indicates 1-line text format for each event. "json" indicates structured json format. Known formats are legacy,json. (default "json")
      --audit-log-maxage int                        The maximum number of days to retain old audit log files based on the timestamp encoded in their filename.
      --audit-log-maxbackup int                     The maximum number of old audit log files to retain. Setting a value of 0 will mean there's no restriction on the number of files.
      --audit-log-maxsize int                       The maximum size in megabytes of the audit log file before it gets rotated.
      --audit-log-mode string                       Strategy for sending audit events. Blocking indicates sending events should block server responses. Batch causes the backend to buffer and write events asynchronously. Known modes are batch,blocking,blocking-strict. (default "blocking")
      --audit-log-path string                       If set, all requests coming to the apiserver will be logged to this file.  '-' means standard out.
      --audit-log-truncate-enabled                  Whether event and batch truncating is enabled.
      --audit-log-truncate-max-batch-size int       Maximum size of the batch sent to the underlying backend. Actual serialized size can be several hundreds of bytes greater. If a batch exceeds this limit, it is split into several batches of smaller size. (default 10485760)
      --audit-log-truncate-max-event-size int       Maximum size of the audit event sent to the underlying backend. If the size of an event is greater than this number, first request and response are removed, and if this doesn't reduce the size enough, event is discarded. (default 102400)
      --audit-log-version string                    API group and version used for serializing audit events written to log. (default "audit.k8s.io/v1")
      --audit-policy-file string                    Path to the file that defines the audit policy configuration.
      --audit-webhook-batch-buffer-size int         The size of the buffer to store events before batching and writing. Only used in batch mode. (default 10000)
      --audit-webhook-batch-max-size int            The maximum size of a batch. Only used in batch mode. (default 400)
      --audit-webhook-batch-max-wait duration       The amount of time to wait before force writing the batch that hadn't reached the max size. Only used in batch mode. (default 30s)
      --audit-webhook-batch-throttle-burst int      Maximum number of requests sent at the same moment if ThrottleQPS was not utilized before. Only used in batch mode. (default 15)
      --audit-webhook-batch-throttle-enable         Whether batching throttling is enabled. Only used in batch mode. (default true)
      --audit-webhook-batch-throttle-qps float32    Maximum average number of batches per second. Only used in batch mode. (default 10)
      --audit-webhook-config-file string            Path to a kubeconfig formatted file that defines the audit webhook configuration.
      --audit-webhook-initial-backoff duration      The amount of time to wait before retrying the first failed request. (default 10s)
      --audit-webhook-mode string                   Strategy for sending audit events. Blocking indicates sending events should block server responses. Batch causes the backend to buffer and write events asynchronously. Known modes are batch,blocking,blocking-strict. (default "batch")
      --audit-webhook-truncate-enabled              Whether event and batch truncating is enabled.
      --audit-webhook-truncate-max-batch-size int   Maximum size of the batch sent to the underlying backend. Actual serialized size can be several hundreds of bytes greater. If a batch exceeds this limit, it is split into several batches of smaller size. (default 10485760)
      --audit-webhook-truncate-max-event-size int   Maximum size of the audit event sent to the underlying backend. If the size of an event is greater than this number, first request and response are removed, and if this doesn't reduce the size enough, event is discarded. (default 102400)
      --audit-webhook-version string                API group and version used for serializing audit events written to webhook. (default "audit.k8s.io/v1")

Features flags:

      --contention-profiling           Enable block profiling, if profiling is enabled
      --debug-socket-path string       Use an unprotected (no authn/authz) unix-domain socket for profiling with the given path
      --enable-priority-and-fairness   If true, replace the max-in-flight handler with an enhanced one that queues and dispatches with priority and fairness (default true)
      --profiling                      Enable profiling via web interface host:port/debug/pprof/ (default true)

Logging flags:

      --log-flush-frequency duration         Maximum number of seconds between log flushes (default 5s)
      --log-json-info-buffer-size quantity   [Alpha] In JSON format with split output streams, the info messages can be buffered for a while to increase performance. The default value of zero bytes disables buffering. The size can be specified as number of bytes (512), multiples of 1000 (1K), multiples of 1024 (2Ki), or powers of those (3M, 4G, 5Mi, 6Gi). Enable the LoggingAlphaOptions feature gate to use this.
      --log-json-split-stream                [Alpha] In JSON format, write error messages to stderr and info messages to stdout. The default is to write a single stream to stdout. Enable the LoggingAlphaOptions feature gate to use this.
      --log-text-info-buffer-size quantity   [Alpha] In text format with split output streams, the info messages can be buffered for a while to increase performance. The default value of zero bytes disables buffering. The size can be specified as number of bytes (512), multiples of 1000 (1K), multiples of 1024 (2Ki), or powers of those (3M, 4G, 5Mi, 6Gi). Enable the LoggingAlphaOptions feature gate to use this.
      --log-text-split-stream                [Alpha] In text format, write error messages to stderr and info messages to stdout. The default is to write a single stream to stdout. Enable the LoggingAlphaOptions feature gate to use this.
      --logging-format string                Sets the log format. Permitted formats: "json" (gated by LoggingBetaOptions), "text". (default "text")
  -v, --v Level                              number for the log level verbosity
      --vmodule pattern=N,...                comma-separated list of pattern=N settings for file-filtered logging (only works for text log format)

panic: error creating self-signed certificates: mkdir apiserver.local.config: read-only file system

goroutine 1 [running]:
main.main()
	sigs.k8s.io/metrics-server/cmd/metrics-server/metrics-server.go:37 +0x88


--R----R----

Отличная диагностика: теперь падает сам metrics‑server при старте, потому что ему нужно сгенерировать самоподписанный сертификат, а по умолчанию он пишет его в apiserver.local.config/... на корне контейнера — а root‑файловая система у пода read‑only. Отсюда:

panic: error creating self-signed certificates: mkdir apiserver.local.config: read-only file system

Лекарь простой — сказать metrics‑server писать сертификаты в /tmp, который уже смонтирован как emptyDir.

Сделай так:

# добавить аргумент --cert-dir=/tmp
kubectl -n kube-system patch deploy metrics-server \
  --type='json' \
  -p='[
    {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--cert-dir=/tmp"}
  ]'

# убедиться, что порт и сервис ок (мы уже правили, но на всякий случай)
kubectl -n kube-system get deploy metrics-server -o=jsonpath='{.spec.template.spec.containers[0].args}'; echo
kubectl -n kube-system get deploy metrics-server -o=jsonpath='{.spec.template.spec.containers[0].ports}'; echo
kubectl -n kube-system get svc metrics-server -o yaml | grep -E 'port:|targetPort:'

# дождаться раскатки
kubectl -n kube-system rollout status deploy/metrics-server

# проверить логи
kubectl -n kube-system logs deploy/metrics-server --tail=100

Должны исчезнуть и panic, и TLS‑жалобы к kubelet (мы уже включили --kubelet-insecure-tls и правильные адреса).

Если всё ок — метрики поедут:

kubectl top nodes
kubectl top pods -A

Если что-то всё ещё не так — кинь последние 50–100 строк логов deploy/metrics-server, глянем дальше.


--R----R----

root@node2:/home/axitech/KubernetesConfigs/gitlab-runner# kubectl top nodes
NAME             CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
node1.internal   247m         12%    5649Mi          48%       
node2.internal   150m         3%     3082Mi          52%       
node3.internal   199m         9%     2059Mi          54%       
node4.internal   54m          1%     1307Mi          16%       
node5.internal   24m          1%     847Mi           22%       
node6.internal   126m         3%     1312Mi          17%   

--R----R----








