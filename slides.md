---
marp: true
theme: default
paginate: true
style: |
  section {
    font-family: 'Segoe UI', Arial, sans-serif;
    font-size: 22px;
    background: #ffffff;
    color: #1a1a2e;
  }
  h1 {
    color: #0066cc;
    font-size: 42px;
    border-bottom: 3px solid #0066cc;
    padding-bottom: 10px;
  }
  h2 {
    color: #0066cc;
    font-size: 32px;
  }
  h3 {
    color: #004499;
    font-size: 26px;
  }
  code {
    background: #f0f4f8;
    color: #1a1a2e;
    border-radius: 4px;
    padding: 2px 6px;
  }
  pre {
    background: #f0f4f8;
    border-left: 4px solid #0066cc;
    border-radius: 6px;
    padding: 16px;
    font-size: 16px;
  }
  ul li {
    margin-bottom: 8px;
  }
  section.title {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    text-align: center;
    background: linear-gradient(135deg, #0066cc 0%, #004499 100%);
    color: white;
  }
  section.title h1 {
    color: white;
    border-bottom: 3px solid rgba(255,255,255,0.5);
    font-size: 52px;
  }
  section.title h2 {
    color: rgba(255,255,255,0.85);
    font-size: 28px;
    font-weight: normal;
  }
  section.section-title {
    display: flex;
    flex-direction: column;
    justify-content: center;
    background: #e8f0fe;
    border-left: 8px solid #0066cc;
  }
  section.section-title h1 {
    font-size: 44px;
    border-bottom: none;
  }
  table {
    width: 100%;
    border-collapse: collapse;
    font-size: 18px;
  }
  th {
    background: #0066cc;
    color: white;
    padding: 8px 12px;
  }
  td {
    padding: 8px 12px;
    border-bottom: 1px solid #dde3ea;
  }
  tr:nth-child(even) td {
    background: #f0f4f8;
  }
---

<!-- _class: title -->

# Kubernetes
## Архитектура, эксплуатация и курс на AI
### 2026

---

<!-- _class: section-title -->

# 1. Введение

---

## Что такое Kubernetes

- **Оркестратор контейнеров** — берёт декларативное описание желаемого состояния и поддерживает его на кластере машин
- Размещает, перезапускает, масштабирует, связывает сетью
- Вырос из внутренней системы Google — **Borg**
- Open source с 2014 года, под управлением **CNCF**
- Де-факто стандарт оркестрации в индустрии

---

## Три фундаментальных концепции

**Декларативность**
Описываешь желаемое состояние — не шаги к нему

**Reconciliation loop**
Control plane непрерывно сверяет текущее состояние с желаемым и исправляет расхождения

**API-first**
Всё в k8s — объект в API. Поды, сервисы, конфиги, права — всё через единый интерфейс

---

## Контекст и масштаб

| Параметр | Значение |
|---|---|
| Актуальная версия | v1.36 (апрель 2026) |
| Поддерживаемые версии | 1.34, 1.35, 1.36 |
| Частота релизов | 3 раза в год |
| Максимум нод | 5 000 |
| Максимум подов | 300 000 на кластер |

- Наш контекст: **on-prem**, банковский сектор
- Полный контроль, соответствие требованиям безопасности и регуляторики

```bash
# Проверить версию кластера
kubectl version --short
# Client Version: v1.36.0
# Server Version: v1.36.0
```

---

## Helm — пакетный менеджер Kubernetes

- **Chart** — пакет шаблонов Kubernetes-манифестов
- **Values** — параметры под окружение, переопределяют chart без изменения шаблонов
- **Release** — установленный экземпляр chart в кластере
- **Repository** — хранилище чартов

```
chart/
├── Chart.yaml       ← метаданные: имя, версия, зависимости
├── values.yaml      ← значения по умолчанию
└── templates/       ← Kubernetes YAML с шаблонами Go
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

---

## Helm — основные команды

```bash
# Установить / обновить release
helm upgrade --install my-app ./chart \
  -f values.prod.yaml \
  -n production

# Все release в кластере
helm list -A

# История ревизий и откат
helm history my-app -n production
helm rollback my-app 2 -n production
```

> `upgrade --install` — идемпотентная операция: создаст если нет, обновит если есть

---

## Helm в production: паттерны

- **Один chart — много окружений**: `values.dev.yaml`, `values.prod.yaml`
- **Chart в Git** — весь стек под версионным контролем
- **Автоматизация деплоя** — через Ansible или CI/CD
- **Локальное хранилище** — без зависимости от внешних репозиториев

```bash
# Что реально будет применено — превью перед деплоем
helm template my-app ./chart -f values.prod.yaml
```

---

<!-- _class: section-title -->

# 2. Архитектура кластера

---

## Две плоскости кластера

```
┌─────────────────────────────────────────┐
│              CONTROL PLANE              │
│  kube-apiserver  │  kube-scheduler      │
│  etcd            │  controller-manager  │
└─────────────────────────────────────────┘
           │ единственная точка входа
┌──────────┼──────────┬───────────────────┐
│  NODE 1  │  NODE 2  │      NODE 3       │
│ kubelet  │ kubelet  │     kubelet       │
│kube-proxy│kube-proxy│   kube-proxy      │
│containerd│containerd│   containerd      │
└──────────┴──────────┴───────────────────┘
```

---

## Control Plane

**kube-apiserver**
Единственная точка входа. Все компоненты общаются только через него. Валидирует, аутентифицирует, авторизует, пишет в etcd

**etcd**
Единственное место хранения состояния кластера. Нет etcd — нет кластера

**kube-scheduler**
Следит за подами без ноды, выбирает куда их поставить

**kube-controller-manager**
Набор контроллеров в одном процессе. Каждый реализует reconciliation loop

---

## Data Plane (Worker Nodes)

**kubelet**
Агент на каждой ноде. Получает PodSpec, обеспечивает запуск контейнеров. Общается с runtime через CRI

**kube-proxy**
Сетевые правила на ноде. Реализует абстракцию Service — трафик на ClusterIP → реальный под

**container runtime**
Запускает контейнеры. Стандарт — **containerd** через CRI
> Docker как runtime умер с v1.24

```bash
# Все ноды кластера — версия, CRI, IP
kubectl get nodes -o wide
# NAME       STATUS   VERSION   CONTAINER-RUNTIME
# master-1   Ready    v1.36.0   containerd://2.0.1
# worker-1   Ready    v1.36.0   containerd://2.0.1

# Компоненты control plane живые
kubectl get pods -n kube-system
```

---

## Flow: `kubectl apply` → Running pod

1. `kubectl` → **apiserver**: валидация, аутентификация, admission webhooks, запись в etcd
2. **scheduler** видит под без ноды → выбирает ноду → записывает назначение в etcd
3. **kubelet** на выбранной ноде видит новый PodSpec → дёргает containerd через CRI
4. **containerd** скачивает образ, создаёт и запускает контейнер
5. **kubelet** репортит статус обратно в apiserver

> Весь flow **асинхронный** — `kubectl apply` не ждёт запуска пода

---

<!-- _class: section-title -->

# 3. Жизненный цикл пода

---

## Путь пода: Pending → Running

```
kubectl apply
     │
     ▼
  apiserver ──── etcd
     │
     ▼
Admission Webhooks
  Mutating → Validating
     │
     ▼
  Scheduler
  выбирает ноду
     │
     ▼
  kubelet → containerd → контейнер запущен
     │
     ▼
  Pod: Running
```

---

## Фазы пода

| Фаза | Описание |
|---|---|
| **Pending** | Принят кластером, контейнеры ещё не запущены |
| **Running** | Назначен на ноду, хотя бы один контейнер запущен |
| **Succeeded** | Все контейнеры завершились с кодом 0 |
| **Failed** | Все завершились, хотя бы один с ненулевым кодом |
| **Unknown** | apiserver не может получить статус — проблема с нодой |

---

## Init vs Sidecar containers

**Init containers**
- Запускаются **строго последовательно** до основных
- Каждый следующий — только если предыдущий завершился с кодом 0
- Типично: ждать БД, подготовить конфиги

**Sidecar containers → Stable (v1.33)**
- `restartPolicy: Always` в init containers
- Живут **всё время жизни пода**
- Поддерживают probes
- Паттерн: log agents, service mesh proxy, secrets injector

---

## Restart Policy и CrashLoopBackOff

**Restart Policy**
- `Always` — перезапускать всегда (default для Deployment)
- `OnFailure` — только при ненулевом exit code
- `Never` — не перезапускать

**CrashLoopBackOff** — экспоненциальная задержка: `10с → 20с → 40с → max 5 мин`

```bash
# Что произошло до прошлого перезапуска
kubectl logs <pod> --previous
# Подробно: статус, число рестартов, Events
kubectl describe pod <pod>
# Exit code последнего падения
kubectl get pod <pod> -o jsonpath=\
  '{.status.containerStatuses[0].lastState.terminated.exitCode}'
# 137 = OOMKilled  |  1 = ошибка приложения  |  0 = успех
```

---

## Probes

**livenessProbe** — жив ли. Если нет — kubelet перезапускает
**readinessProbe** — готов ли. Если нет — выходит из endpoints
**startupProbe** — блокирует liveness пока приложение не застартовало

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15  # ждём старта
  periodSeconds: 10
  failureThreshold: 3       # 3 провала → перезапуск

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 2       # 2 провала → выходит из endpoints
```

> ⚠️ `failureThreshold` слишком низкий + liveness = перезапуски под нагрузкой

---

<!-- _class: section-title -->

# 4. Сеть в Kubernetes

---

## Flat Network Model

Четыре правила которые не зависят от реализации:

- Каждый под получает **свой уникальный IP**
- Поды общаются напрямую **без NAT** — на любой ноде
- Нода может достучаться до любого пода **без NAT**
- IP который под видит у себя = IP который видят другие

Реализацию берёт на себя **CNI-плагин**

---

## CNI — Container Network Interface

Стандарт плагинов для сетевой связности контейнеров

Когда kubelet создаёт под — вызывает CNI-плагин:
- Создаёт сетевой интерфейс в network namespace пода
- Назначает IP
- Настраивает маршруты

> Выбор CNI — одно из первых решений при развёртывании кластера

---

## Calico — почему мы используем его

- Маршрутизация через **BGP** — каждая нода анонсирует подсеть своих подов
- **Без оверлея** по умолчанию — трафик напрямую, без инкапсуляции
- Fallback: VXLAN / IP-in-IP если BGP недоступен
- Реализует **Network Policy**
- **Felix** — агент на каждой ноде, программирует iptables/eBPF правила

---

## Traffic Flows

```
[ Pod A ]──veth──[ bridge/eBPF ]──veth──[ Pod B ]  ← одна нода, нет NAT

[ Pod A ]──BGP route──[ Node 2 eth0 ]──veth──[ Pod B ]  ← разные ноды

[ Pod ]──iptables/nftables──[ ClusterIP ]──реальный Pod
           kube-proxy программирует правила

[ External ]──NodePort/LB──DNAT──[ Pod ]
```

> ClusterIP — виртуальный IP, не существует на ни одном интерфейсе ноды

---

## Network Policy

По умолчанию — всё открыто. Network Policy ограничивает трафик на L3/L4

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-only-frontend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
```

> ⚠️ Работает только если CNI поддерживает. Calico ✅ Flannel ❌

---

## Ingress → Gateway API

**Проблемы Ingress:**
- Vendor-specific annotations у каждого контроллера
- Нет разделения ролей — платформа и приложение в одном объекте
- Только HTTP/HTTPS

**Ingress-NGINX — EOL март 2026** 🔴

**Gateway API — официальная замена:**

| Объект | Владелец | Ответственность |
|---|---|---|
| `GatewayClass` | Инфраструктура | Определяет контроллер |
| `Gateway` | Платформа | Точка входа, TLS, listeners |
| `HTTPRoute` / `TLSRoute` | Команда приложения | Правила маршрутизации |

---

<!-- _class: section-title -->

# 5. Scheduler и Affinity

---

## Как работает планировщик

```
Все ноды кластера
       │
       ▼
  ┌─────────┐
  │  FILTER │  Отсекает неподходящие ноды
  └─────────┘  (CPU/memory, affinity, taints, NotReady)
       │
       ▼
  ┌─────────┐
  │  SCORE  │  Оценивает оставшиеся ноды
  └─────────┘  (баланс ресурсов, топология)
       │
       ▼
  ┌─────────┐
  │  BIND   │  Записывает решение в etcd
  └─────────┘
```

```bash
# Под завис в Pending — смотреть Events
kubectl describe pod <pod> -n <ns>
# Events:
#   Warning  FailedScheduling  0/5 nodes available:
#   3 node(s) had untolerated taint {dedicated: gpu},
#   2 Insufficient memory
```

---

## nodeSelector и nodeAffinity

**nodeSelector** — простой, но негибкий:
```yaml
spec:
  nodeSelector:
    disktype: ssd
```

**nodeAffinity** — мощная замена:
```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/arch
          operator: In
          values: [amd64]
```

- `required...` — жёсткое требование
- `preferred...` — предпочтение с весом

---

## podAffinity и podAntiAffinity

**podAntiAffinity** — разнести реплики по нодам:
```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: backend
      topologyKey: kubernetes.io/hostname
```

| topologyKey | Эффект |
|---|---|
| `kubernetes.io/hostname` | Не более 1 реплики на ноду |
| `topology.kubernetes.io/zone` | Не более 1 реплики в зону |

---

## Taints и Tolerations

**Taint на ноде** — нода отталкивает поды:
```bash
kubectl taint nodes node1 dedicated=gpu:NoSchedule
```

| Эффект | Поведение |
|---|---|
| `NoSchedule` | Новые поды без toleration не попадут |
| `PreferNoSchedule` | Scheduler постарается избежать |
| `NoExecute` | Существующие поды будут выселены |

**Toleration в поде:**
```yaml
tolerations:
- key: dedicated
  operator: Equal
  value: gpu
  effect: NoSchedule
```

```bash
# Labelы нод — основа для nodeAffinity
kubectl get nodes --show-labels
# Taints на конкретной ноде
kubectl describe node <node> | grep -A5 Taints
```

---

<!-- _class: section-title -->

# 6. Хранилище

---

## Три ключевых объекта

**PersistentVolume (PV)** — кусок хранилища, кластерный ресурс
**PersistentVolumeClaim (PVC)** — запрос от пода, только через него под получает том
**StorageClass** — шаблон: provisioner + параметры + reclaim policy

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: fast
  resources:
    requests:
      storage: 10Gi
```

```bash
kubectl get pvc -n <ns>
# NAME       STATUS   VOLUME   CAPACITY   ACCESS MODES
# data-pvc   Bound    pv-001   10Gi       RWO
```

---

## Access Modes

| Режим | Описание |
|---|---|
| `ReadWriteOnce (RWO)` | Один под на одной ноде. Самый распространённый |
| `ReadOnlyMany (ROX)` | Много подов читают |
| `ReadWriteMany (RWX)` | Много подов читают и пишут. Нужен NFS/CephFS |
| `ReadWriteOncePod (RWOP)` | Только один под во всём кластере |

---

## Dynamic Provisioning

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/no-provisioner
parameters:
  type: ssd
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

> `WaitForFirstConsumer` — PV создаётся только после scheduling пода
> Гарантирует что том создастся в правильной зоне

**Reclaim Policy:**
- `Delete` — данные удаляются. Осторожно в production
- `Retain` — данные сохраняются, PV освобождается вручную

```bash
# Все PV кластера — статус, reclaim policy, размер
kubectl get pv
# Доступные StorageClass
kubectl get storageclass
```

---

## Подводные камни StatefulSet

- PVC **не удаляются** при удалении StatefulSet — защита от потери данных
- Масштабирование **строго последовательное** — pod-1 ждёт pod-0 Ready
- При удалении пода его PVC биндится к новому поду с тем же именем
- Для большинства stateful приложений достаточно **RWO** — каждый под свой том

```bash
# PVC остаются после удаления StatefulSet
kubectl delete statefulset postgres
kubectl get pvc
# NAME              STATUS   VOLUME    CAPACITY
# data-postgres-0   Bound    pv-001    20Gi     ← не удалился!
# data-postgres-1   Bound    pv-002    20Gi
```

---

<!-- _class: section-title -->

# 7. RBAC и Безопасность

---

## Ключевые объекты RBAC

| Объект | Описание |
|---|---|
| `ServiceAccount` | Идентификатор для процессов внутри кластера |
| `Role` | Права в рамках одного namespace |
| `ClusterRole` | Права на уровне всего кластера |
| `RoleBinding` | Связывает Role с субъектом в namespace |
| `ClusterRoleBinding` | Связывает ClusterRole с субъектом глобально |

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: pod-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: pod-reader-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: monitoring-sa
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## Принцип минимальных прав

Типичные ошибки:

❌ **`default` ServiceAccount для всего** — накапливает лишние права

❌ **`cluster-admin` для приложений** — открывает полный доступ к кластеру

❌ **`automountServiceAccountToken: true`** для подов которые не ходят в apiserver — токен в файловой системе доступен атакующему

```yaml
spec:
  automountServiceAccountToken: false
```

---

## Secrets — типичные ошибки

❌ **base64 ≠ шифрование** — это просто кодирование

```bash
# Любой может прочитать Secret одной командой
kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d
# mysecretpassword123
```

❌ **Encryption at Rest выключен по умолчанию** — данные в etcd в открытом виде

❌ **Секреты через env** — видны в `kubectl describe pod`, могут утечь в логи

✅ **Монтировать как volume** — доступно только через файловую систему

```bash
# Что может делать конкретный ServiceAccount
kubectl auth can-i list pods \
  --as=system:serviceaccount:default:default
# Все RoleBinding в namespace
kubectl get rolebindings -n <ns> -o wide
# Все ClusterRoleBinding — кто имеет широкие права
kubectl get clusterrolebindings -o wide | grep -v system:
```

---

## Mutating Admission Policies → Stable (v1.36)

**До v1.36:** отдельный webhook-сервер → TLS → операционная нагрузка

**Теперь:** декларативно через CEL прямо в YAML

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingAdmissionPolicy
metadata:
  name: set-default-limits
spec:
  matchConstraints:
    resourceRules:
    - apiGroups: ["apps"]
      apiVersions: ["v1"]
      operations: ["CREATE"]
      resources: ["deployments"]
  mutations:
  - patchType: ApplyConfiguration
    expression: "Object{...}"
```

> Оценка in-process внутри apiserver — быстрее, надёжнее, нет внешних зависимостей

---

<!-- _class: section-title -->

# 8. etcd Deep Dive

---

## Почему etcd — особенный

- **Единственное место** хранения состояния кластера
- Все объекты, конфиги, секреты, статусы — в etcd
- Все остальные компоненты k8s **stateless** — их можно пересоздать
- etcd потерял данные → **кластер потерян**

---

## Raft-консенсус

- Один узел — **leader**, остальные — followers
- Все записи идут **только через leader**
- Запись успешна когда **кворум** подтвердил

**Кворум = (N/2) + 1**

| Нод | Кворум | Можно потерять |
|---|---|---|
| 1 | 1 | 0 |
| 3 | 2 | **1** |
| 5 | 3 | **2** |
| 7 | 4 | **3** |

---

## Почему нечётное число нод

При чётном числе — отказоустойчивость не растёт, стоимость растёт:

- 4 ноды: кворум 3, потерять 1 — **то же что 3 ноды**
- 6 нод: кворум 4, потерять 2 — **то же что 5 нод**

**Стандарт: 3 или 5 нод etcd**
7 — только для очень крупных кластеров
> Больше 7 — не рекомендуется, latency записи растёт

---

## Бэкап и рестор

**Снапшот:**
```bash
ETCDCTL_API=3 etcdctl snapshot save snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

**Проверить снапшот:**
```bash
etcdctl snapshot status snapshot.db --write-out=table
```

**Рестор:**
```bash
etcdctl snapshot restore snapshot.db --data-dir=/var/lib/etcd-restore
```

> 🔑 Золотое правило: бэкап etcd **перед любым обновлением кластера**

---

## Диагностика

```bash
# Здоровье, список членов, лидер
etcdctl endpoint health --write-out=table
etcdctl member list --write-out=table
etcdctl endpoint status --write-out=table

# Переполнение базы: дефрагментация + проверка размера
etcdctl defrag --endpoints=https://127.0.0.1:2379 \
  --cacert=... --cert=... --key=...
etcdctl endpoint status --write-out=table
# DB SIZE уменьшился
```

**Типичные проблемы:**
- **Split-brain** — нет кворума, read-only. Причина: сетевой партиционинг
- **Медленный диск** — fsync на каждую запись. Только SSD, выделенный диск
- **Переполнение** — лимит 2GB. `etcdctl defrag` + auto-compaction
- **Частая смена лидера** — нестабильная сеть, перегруженные ноды

---

<!-- _class: section-title -->

# 9. Kubernetes 2025–2026
## Курс на AI

---

## Системный разворот платформы

Последние релизы — не просто набор фич.
Это **системный разворот** под новый класс workloads:

- AI / ML inference
- GPU-вычисления
- HPC

Kubernetes перестаёт быть просто "платформой для микросервисов" —
это **инфраструктурное ядро для AI production**

---

## DRA → GA (v1.34)

**Dynamic Resource Allocation** — новый механизм для специализированного железа

**Проблема device plugins:** GPU выдаётся целиком, нельзя разделить, нет видимости состояния

**DRA решает:**
```yaml
spec:
  devices:
    requests:
    - name: gpu
      firstAvailable:
      - deviceClassName: big-gpu
      - deviceClassName: mid-gpu
      - deviceClassName: small-gpu
```

v1.36: поддержка **partitionable devices** — один GPU делится между workloads

---

## Gateway API Inference Extension (v1.36)

Обычный round-robin не работает для LLM inference:

- Запросы к разным моделям — разная стоимость
- Бэкенд может быть занят длинным inference
- Прогретый KV-cache критичен для latency

**Inference Extension маршрутизирует с учётом:**
- Какая модель запрошена
- Состояние очереди на бэкенде
- Прогретость кэша
- Метрики: tokens/sec, queue depth

---

## In-Place Pod Resize → GA (v1.35)

Вертикальное масштабирование **без рестарта пода**

```bash
kubectl patch pod inference-pod --patch \
  '{"spec":{"containers":[{"name":"model",
    "resources":{"limits":{"cpu":"8","memory":"32Gi"}}}]}}'
```

Критично для inference:
- Модель загружена в память
- Рестарт дорогой
- Динамически добавляем ресурсы под нагрузкой

---

## Остальные важные изменения

| Фича | Версия | Статус |
|---|---|---|
| Sidecar containers | v1.33 | ✅ Stable |
| In-Place Pod Resize | v1.35 | ✅ GA |
| DRA | v1.34 | ✅ GA |
| Mutating Admission Policies | v1.36 | ✅ Stable |
| User Namespaces | v1.36 | ✅ Stable |
| Gateway API v1.5 | февраль 2026 | ✅ Stable |
| cgroup v1 | v1.35 | 🔴 Удалён |
| containerd 1.x | v1.35 | 🔴 EOL |
| Ingress-NGINX | март 2026 | 🔴 EOL |

---

## Вывод

AI меняет требования к инфраструктуре:
- **GPU** как управляемый ресурс → DRA
- **Умный routing** с учётом состояния модели → Gateway API Inference Extension
- **Динамическое масштабирование** без даунтайма → In-Place Pod Resize

Инструменты которые мы уже знаем —
становятся платформой для следующего поколения workloads

---

<!-- _class: title -->

# Спасибо
## Вопросы?
