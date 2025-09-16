# Kubernetes Nodes — академическое введение и практическое руководство

## 1. Фундаментальное описание

**Node** — вычислительный узел кластера, предоставляющий ресурсы CPU/памяти/сети для рабочих нагрузок в Pod’ах. На каждой ноде запускается **kubelet** (агент управления), **container runtime** (через CRI, например CRI‑O или containerd) и обычно **kube-proxy** (сетевой data‑plane для сервисов).

Ключевые элементы узла:
- **kubelet**: регистрирует ноду в API, запускает/останавливает Pod’ы через CRI, обновляет статусы, публикует условия ноды (*Node Conditions*).
- **CRI runtime**: изоляция контейнеров, образы, сети/тома (совместно с CNI/CSI).
- **kube-proxy**: программирует правила транспорта сервисов (iptables/IPVS) или заменяется CNIs/LB.
- **cAdvisor** (встроен в kubelet): телеметрия ресурсов контейнеров, используется многими метриками.

Модель состояния ноды:
- **Capacity/Allocatable** — «всего» и «доступно для Pod’ов с учётом системных резерваций».
- **Conditions** — `Ready`, `MemoryPressure`, `DiskPressure`, `PIDPressure`, `NetworkUnavailable` и т.д.
- **Labels/Taints** — семантика размещения (ролей, зон, аппаратных профилей и ограничений планировщика).

Статус `Ready=True` означает, что kubelet корректно общается с API‑сервером и нода готова принимать Pod’ы. `NotReady` или `Unknown` указывает на проблемы с kubelet, сетью, сертификатами или перегрузкой.

---

## 2. Обзор и роли нод (команды)

Список нод:
```bash
kubectl get nodes
```

Пример из кластера:
```text
NAME             STATUS     ROLES           AGE    VERSION
node1.internal   NotReady   control-plane   241d   v1.31.4
node2.internal   NotReady   control-plane   241d   v1.31.4
node3.internal   Ready      control-plane   241d   v1.31.4
node4.internal   Ready      <none>          241d   v1.31.4
node5.internal   Ready      <none>          241d   v1.31.4
node6.internal   Ready      <none>          241d   v1.31.4
```

Расширенный вывод (IP/контейнерный рантайм/ярлыки ролей):
```bash
kubectl get nodes -o wide
kubectl get nodes --show-labels
kubectl get nodes -L node-role.kubernetes.io/control-plane -o wide
kubectl get nodes -o   custom-columns=NAME:.metadata.name,ROLE:.metadata.labels['node-role\.kubernetes\.io/control-plane']
```

Пример:
```text
NAME             STATUS     ROLES           AGE    VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION   CONTAINER-RUNTIME   CONTROL-PLANE
node1.internal   NotReady   control-plane   241d   v1.31.4   <k8s node 1 ip>   <none>        Debian GNU/Linux 12 (bookworm)   6.1.0-28-amd64   cri-o://1.31.3      
node2.internal   NotReady   control-plane   241d   v1.31.4   <k8s node 2 ip>   <none>        Debian GNU/Linux 12 (bookworm)   6.1.0-31-amd64   cri-o://1.31.5      
node3.internal   Ready      control-plane   241d   v1.31.4   <k8s node 3 ip>   <none>        Debian GNU/Linux 12 (bookworm)   6.1.0-28-amd64   cri-o://1.31.3      
node4.internal   Ready      <none>          241d   v1.31.4   <k8s node 4 ip>   <none>        Debian GNU/Linux 12 (bookworm)   6.1.0-28-amd64   cri-o://1.31.3      
node5.internal   Ready      <none>          241d   v1.31.4   <k8s node 5 ip>   <none>        Debian GNU/Linux 12 (bookworm)   6.1.0-28-amd64   cri-o://1.31.3      
node6.internal   Ready      <none>          241d   v1.31.4   <k8s node 6 ip>   <none>        Debian GNU/Linux 12 (bookworm)   6.1.0-28-amd64   cri-o://1.31.3
```

Подробная информация по ноде:
```bash
kubectl describe node <имя-ноды>
```
Полезно для просмотра `Labels`, `Capacity/Allocatable`, `Conditions`, версии kubelet, событий и т.д.

Статус готовности (jsonpath):
```bash
kubectl get nodes   -o jsonpath='{range .items[*]}{.metadata.name}{"	"}{.status.conditions[?(@.type=="Ready")].status}{"	"}{.status.conditions[?(@.type=="Ready")].reason}{"	"}{.status.conditions[?(@.type=="Ready")].message}{"
"}{end}'
```
Пример вывода:
```text
node1.internal    Unknown  NodeStatusUnknown  Kubelet stopped posting node status.
node2.internal    Unknown  NodeStatusUnknown  Kubelet stopped posting node status.
node3.internal    True     KubeletReady       kubelet is posting ready status
node4.internal    True     KubeletReady       kubelet is posting ready status
node5.internal    True     KubeletReady       kubelet is posting ready status
node6.internal    True     KubeletReady       kubelet is posting ready status
```

---

## 3. Метрики и наблюдаемость

`kubectl top` требует установленный **metrics-server**.
```bash
watch -n 5 'kubectl top pods -A --sort-by=cpu | head -20'
watch -n 5 'kubectl top nodes | head -20'
```

Суммарная загрузка кластера (пример из заметок):
```bash
kubectl top nodes | awk 'NR>1 {cpu+=$2; mem+=$4} END {print "CPU(m):", cpu, "Memory(Mi):", mem}'
# Пример: CPU(m): 1218  Memory(Mi): 14761
```
Единицы измерения:
- `m` в `CPU(m)` — **millicore** (1/1000 vCPU). 1000m ≈ 1 vCPU. (В выводе Kubernetes m в CPU(m) означает милликор (millicore) — тысячная доля одного CPU ядра.)
- Память в `Mi` — **mebibytes** (2^20 байт).

Подсчёт контейнеров по нодам:
```bash
kubectl get pods -A -o json | jq -r '.items[] | "\(.spec.nodeName) \(.spec.containers[].name)"' | awk '{print $1}' | sort | uniq -c
```

---

## 4. Диагностика состояний `NotReady` / `Unknown`

Частые причины:
- kubelet не работает/падает: `systemctl status kubelet`, `journalctl -u kubelet -f`.
- Разрыв связи kubelet ↔ apiserver: DNS/LB/файрвол, просроченные сертификаты kubelet.
- Неверный CRI‑сокет: убедитесь, что kubelet настроен на CRI‑O (`/var/run/crio/crio.sock`) или нужный runtime.
- CNI неисправна: поды не получают IP/маршруты; проверьте манифесты плагина и логи на ноде.
- Давление ресурсов: `MemoryPressure`, `DiskPressure`, `PIDPressure` в `kubectl describe node`.
- Часы и сертификаты: проверяйте NTP и срок действия сертификатов kubelet.

Быстрый чек‑лист:
```bash
# kubelet
systemctl status kubelet
journalctl -u kubelet --since "1 hour ago"

# runtime (пример для CRI-O)
systemctl status crio
crictl info

# сеть и CNI
ip a; ip r
kubectl -n kube-system get pods -o wide | grep -i cni
```

Операции с нодой при обслуживании:
```bash
# временно не пускать новые поды
kubectl cordon <node>

# аккуратно эвакуировать поды (DaemonSet/StaticPods останутся)
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data

# вернуть в кластер
kubectl uncordon <node>
```

Тонкая настройка размещения:
- **taints/tolerations** — исключают ноды по умолчанию и разрешают только совместимые поды.
- **labels + nodeSelector/nodeAffinity** — позитивный выбор нод по характеристикам (зона/тип CPU/SSD и т.д.).

---

## 5. Памятка команд администратора

```bash
# Инвентарь
kubectl get nodes -o wide --show-labels

# Состояние ноды подробно
kubectl describe node <node>

# Размещение подов на конкретной ноде
kubectl get pods -A -o wide --field-selector spec.nodeName=<node>

# События по ноде
kubectl get events --field-selector involvedObject.kind=Node,involvedObject.name=<node> -A --sort-by=.lastTimestamp
```

---

## 6. Рекомендации по эксплуатации

- Резервируйте ресурсы для системных процессов (kubelet/system-reserved/kube-reserved).
- Следите за версиями: kubelet обычно допускает **N-2** относительно control plane.
- Обновляйте runtime и ядро OS планово; используйте `drain`/`cordon` перед перезагрузкой.
- Используйте метрики/алерты по Conditions и по задержке kubelet → apiserver.
