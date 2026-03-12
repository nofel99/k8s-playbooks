# Ansible — Kubernetes Cluster Setup

Автоматизированное развёртывание Kubernetes кластера на Ubuntu с помощью Ansible.

Кластер: 1 control plane (vm-1) + 2 worker nodes (vm-2, vm-3).  
Container runtime: containerd. CNI: Flannel.

---

## Структура проекта

```
.
├── ansible.cfg              # Конфигурация Ansible (inventory по умолчанию)
├── group_vars/
│   ├── all.yml              # Общие переменные (версии, CIDR)
│   └── k8s_masters.yml      # IP-адреса control plane
├── inventory.yml            # Хосты и группы
├── playbooks/
│   ├── 0-timezone.yml       # Настройка timezone
│   ├── 1-k8s-pre.yml        # Системные prerequisites
│   ├── 2-k8s-install.yml    # Установка kubeadm, kubelet, kubectl
│   ├── 3-containerd-install.yml  # Установка containerd
│   ├── 4-crictl-config.yml  # Конфигурация crictl
│   ├── 5-init-control.yml   # Инициализация control plane
│   ├── 6-cni.yml            # Установка Flannel CNI
│   ├── 7-join-workers.yml   # Подключение worker nodes
│   └── 8-external-ip-ssl.yml # Пересоздание сертификата с внешним IP
├── site.yml                 # Запуск всех плейбуков по порядку
└── README.md
```

---

## Требования

**На управляющей машине:**
- Ansible >= 2.12
- Python 3.12
- SSH-ключ `~/.ssh/yc-dev` с доступом к нодам

**На нодах:**
- Ubuntu 22.04 / 24.04
- Доступ по SSH от root
- Интернет для загрузки пакетов

**Ansible коллекции:**
```bash
ansible-galaxy collection install community.general
```

---

## Переменные

### group_vars/all.yml
| Переменная | Описание |
|---|---|
| `k8s_version` | Версия для APT репозитория (например `1.35`) |
| `k8s_package_version` | Версия пакетов apt (например `1.35.2-1.1`) |
| `k8s_kubeadm_version` | Версия для kubeadm init (например `1.35.2`) |
| `pod_network_cidr` | CIDR для pod сети (`10.244.0.0/16`) |
| `service_cidr` | CIDR для service сети (`10.96.0.0/12`) |
| `flannel_version` | Версия Flannel CNI (например `v0.26.2`) |

### group_vars/k8s_masters.yml
| Переменная | Описание |
|---|---|
| `external_ip` | Внешний IP control plane ноды |
| `internal_ip` | Внутренний IP control plane ноды |

По умолчанию kubeadm выписывает сертификат kube-apiserver только для внутренних адресов кластера. Если ты подключаешься к кластеру снаружи (например со своего ноутбука через kubectl), то TLS соединение падает с ошибкой — внешний IP не прописан в сертификате.
Плейбук пересоздаёт сертификат и добавляет в поле SAN (Subject Alternative Names) оба адреса:

external_ip — чтобы подключаться снаружи через интернет
internal_ip — чтобы работало внутри сети Yandex Cloud

---

## Быстрый старт

**1. Клонировать репозиторий:**
```bash
git clone git@github.com:nofel99/k8s-playbooks.git /opt/ansible
cd /opt/ansible
```

**2. Настроить переменные:**
```bash
# Версии Kubernetes
vim group_vars/all.yml

# IP-адреса control plane
vim group_vars/k8s_masters.yml
```

**3. Проверить доступность нод:**
```bash
ansible all -m ping
```

**4. Запустить полную установку кластера:**
```bash
ansible-playbook site.yml
```

**5. Запустить отдельный плейбук:**
```bash
ansible-playbook playbooks/5-init-control.yml
```

---

## Описание плейбуков

### 0-timezone.yml
Устанавливает timezone `Asia/Irkutsk` на всех нодах.  
Группа: `k8s_nodes`

### 1-k8s-pre.yml
Системные prerequisites для Kubernetes:
- отключение swap
- загрузка модулей ядра `overlay`, `br_netfilter`
- настройка sysctl параметров для сети

Группа: `k8s_nodes`

### 2-k8s-install.yml
Установка Kubernetes компонентов:
- добавление официального APT репозитория
- установка `kubelet`, `kubeadm`, `kubectl` зафиксированных версий
- hold пакетов от автообновления

Группа: `k8s_nodes`

### 3-containerd-install.yml
Установка и настройка containerd как container runtime:
- генерация дефолтного конфига
- включение `SystemdCgroup = true`

Группа: `k8s_nodes`

### 4-crictl-config.yml
Настройка `crictl` для работы с containerd через unix socket.  
Группа: `k8s_nodes`

### 5-init-control.yml
Инициализация control plane:
- `kubeadm init` с указанием CIDR и версии
- настройка `~/.kube/config` для root

Группа: `k8s_masters`

### 6-cni.yml
Установка Flannel CNI через `kubectl apply`.  
Версия фиксируется переменной `flannel_version`.  
Группа: `k8s_masters`

### 7-join-workers.yml
Подключение worker нод к кластеру:
- генерация join-команды на мастере
- выполнение join на каждой worker ноде (с проверкой — не джойнит повторно)

Группы: `k8s_masters`, `k8s_workers`

### 8-external-ip-ssl.yml
Пересоздание сертификата kube-apiserver с добавлением внешнего IP в SAN:
- удаление старых сертификатов
- генерация новых через `kubeadm init phase certs`
- перезапуск apiserver через crictl
- скачивание обновлённого kubeconfig на управляющую машину

Группа: `k8s_masters`
