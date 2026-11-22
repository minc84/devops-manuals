# Автоматическое развертывание Kubernetes-кластера (Terraform + Ansible)

Этот репозиторий содержит полный инфраструктурный код для автоматического развертывания production-ready Kubernetes-кластера (1 мастер + 2 воркера) на базе виртуальных машин Ubuntu 24.04 в KVM/Libvirt.

*   **Terraform** — создает виртуальные машины, сети, диски и генерирует inventory для Ansible.
*   **Ansible** — устанавливает и настраивает компоненты Kubernetes (containerd, kubeadm, Calico, Ingress-NGINX).

## 🏗 Архитектура

*   **Сеть VM:** `10.10.10.0/24` (NAT, шлюз `10.10.10.1`)
*   **Сеть подов (Calico):** `192.168.0.0/16`
*   **Мастер-нода:** `10.10.10.10` (Control Plane + etcd)
*   **Воркер-ноды:** `10.10.10.11`, `10.10.10.12`
*   **Ingress:** NodePort (без MetalLB, чтобы избежать проблем с ARP в NAT-сетях libvirt)

---

## 📂 Структура проекта

```text
.
├── main.tf                      # Инфраструктура Terraform (VM, сети, диски)
├── variables.tf                 # Переменные Terraform
├── terraform.tfvars             # Значения переменных (IP-адреса, ресурсы)
├── cloud_init.cfg               # Шаблон cloud-init для базовой настройки VM
├── network_config.cfg           # Шаблон netplan для статической сети
├── inventory.tpl                # Шаблон инвентаря Ansible (генерируется Terraform)
├── k8s-setup.yml                # Главный плейбук Ansible
└── roles/
    ├── common/tasks/main.yml    # Базовая настройка ОС, containerd, kubeadm
    ├── master/tasks/main.yml    # Инициализация Control Plane
    ├── worker/tasks/main.yml    # Подключение воркеров
    ├── calico/tasks/main.yml    # Установка CNI
    └── ingress/tasks/main.yml   # Установка Ingress-контроллера
```

---

# ЧАСТЬ 1. Terraform (Инфраструктура)

### 1. `variables.tf`
```hcl
variable "base_image_path" {
  type    = string
  default = "/home/minc84/linux_iso/noble-server-cloudimg-amd64.img"
}

variable "storage_pool" {
  type    = string
  default = "kube_pool_ans"
}

variable "ssh_public_key_path" {
  type        = string
  description = "Путь к публичному SSH ключу"
}

variable "ssh_private_key_path" {
  type        = string
  description = "Путь к приватному SSH ключу (для Ansible)"
  default     = "~/.ssh/id_rsa"
}

variable "k8s_version" {
  type        = string
  default     = "1.31"
}

variable "vms" {
  type = map(object({
    hostname = string
    memory   = number
    vcpus    = number
    disk_gb  = number
    ip       = string
    gateway  = string
    nic      = string
    role     = optional(string, "worker")
  }))
}
```

### 2. `terraform.tfvars`
```hcl
ssh_public_key_path  = "/home/minc84/.ssh/id_rsa.pub"
ssh_private_key_path = "/home/minc84/.ssh/id_rsa"

vms = {
  "k8s-master-ans" = {
    hostname = "k8s-master-ans"
    memory   = 4096
    vcpus    = 2
    disk_gb  = 20
    ip       = "10.10.10.10"
    gateway  = "10.10.10.1"
    nic      = "ens3"
    role     = "master"
  }
  "k8s-worker-01-ans" = {
    hostname = "k8s-worker-01-ans"
    memory   = 4096
    vcpus    = 2
    disk_gb  = 20
    ip       = "10.10.10.11"
    gateway  = "10.10.10.1"
    nic      = "ens3"
    role     = "worker"
  }
  "k8s-worker-02-ans" = {
    hostname = "k8s-worker-02-ans"
    memory   = 4096
    vcpus    = 2
    disk_gb  = 20
    ip       = "10.10.10.12"
    gateway  = "10.10.10.1"
    nic      = "ens3"
    role     = "worker"
  }
}
```

### 3. `main.tf`
```hcl
terraform {
  required_providers {
    libvirt = { source = "dmacvicar/libvirt", version = "~> 0.7.6" }
    local   = { source = "hashicorp/local",   version = "~> 2.5"   }
  }
}

provider "libvirt" { uri = "qemu:///system" }

resource "libvirt_pool" "kube_pool" {
  name = var.storage_pool
  type = "dir"
  path = "/var/lib/libvirt/images/${var.storage_pool}"
}

resource "libvirt_network" "vm_network" {
  name      = "k8s-net-ans"
  mode      = "nat"
  autostart = true
  domain    = "lab-ans"
  addresses = ["10.10.10.0/24"]
  dhcp { enabled = false }
  dns  { enabled = true  }
}

resource "libvirt_volume" "base_image" {
  name   = "ubuntu-24-04-base-ans.qcow2"
  pool   = libvirt_pool.kube_pool.name
  source = var.base_image_path
  format = "qcow2"
}

resource "libvirt_volume" "vm_disk" {
  for_each       = var.vms
  name           = "${each.key}-disk.qcow2"
  pool           = libvirt_pool.kube_pool.name
  base_volume_id = libvirt_volume.base_image.id
  format         = "qcow2"
  size           = each.value.disk_gb * 1073741824
}

resource "libvirt_cloudinit_disk" "cloud_init" {
  for_each = var.vms
  name     = "${each.key}-cloudinit.iso"
  pool     = libvirt_pool.kube_pool.name

  user_data = templatefile("${path.module}/cloud_init.cfg", {
    hostname       = each.value.hostname
    ssh_public_key = file(var.ssh_public_key_path)
  })

  network_config = templatefile("${path.module}/network_config.cfg", {
    ip      = each.value.ip
    gateway = each.value.gateway
    nic     = each.value.nic
  })
}

resource "libvirt_domain" "vm" {
  for_each  = var.vms
  name      = each.key
  memory    = each.value.memory
  vcpu      = each.value.vcpus
  type      = "kvm"
  cloudinit = libvirt_cloudinit_disk.cloud_init[each.key].id

  boot_device { dev = ["hd"] }
  disk { volume_id = libvirt_volume.vm_disk[each.key].id }

  network_interface {
    network_id     = libvirt_network.vm_network.id
    addresses      = [each.value.ip]
    wait_for_lease = false
  }

  cpu { mode = "host-passthrough" }
  console {
    type        = "pty"
    target_port = "0"
    target_type = "serial"
  }
  autostart = true
}

resource "local_file" "ansible_inventory" {
  content  = templatefile("${path.module}/inventory.tpl", {
    vms                  = var.vms
    ssh_private_key_path = var.ssh_private_key_path
    k8s_version          = var.k8s_version
  })
  filename = "${path.module}/inventory.ini"
}

resource "null_resource" "ansible_provisioner" {
  depends_on = [libvirt_domain.vm, local_file.ansible_inventory]
  triggers = {
    inventory_hash = local_file.ansible_inventory.content
    always_run     = timestamp()
  }
  provisioner "local-exec" {
    command = "sleep 60 && ansible-playbook -i ${path.module}/inventory.ini ${path.module}/k8s-setup.yml"
  }
}
```

### 4. `cloud_init.cfg`
```yaml
#cloud-config
hostname: ${hostname}
fqdn: ${hostname}.local
manage_etc_hosts: true

users:
  - name: ubuntu
    groups: [sudo]
    shell: /bin/bash
    sudo: ALL=(ALL) NOPASSWD:ALL
    lock_passwd: false
    ssh_authorized_keys:
      - ${ssh_public_key}

ssh_pwauth: false
package_update: true
package_upgrade: true

packages:
  - qemu-guest-agent
  - curl
  - htop
  - vim

runcmd:
  - systemctl enable qemu-guest-agent
  - systemctl start qemu-guest-agent

phone_home:
  url: http://10.10.10.1:10/
  post: [instance-id]
  tries: 1
```

### 5. `network_config.cfg`
```yaml
version: 2
ethernets:
  ${nic}:
    match:
      name: "${nic}"
    dhcp4: false
    addresses:
      - ${ip}/24
    routes:
      - to: default
        via: ${gateway}
    nameservers:
      addresses:
        - 1.1.1.1
        - 8.8.8.8
```

### 6. `inventory.tpl`
```ini
[master]
%{ for name, vm in vms ~}
%{ if vm.role == "master" ~}
${name} ansible_host=${vm.ip}
%{ endif ~}
%{ endfor ~}

[workers]
%{ for name, vm in vms ~}
%{ if vm.role == "worker" ~}
${name} ansible_host=${vm.ip}
%{ endif ~}
%{ endfor ~}

[k8s_cluster:children]
master
workers

[k8s_cluster:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=${ssh_private_key_path}
k8s_version=${k8s_version}
pod_network_cidr=192.168.0.0/16
```

---

# ЧАСТЬ 2. Ansible (Настройка K8s)

### 1. `k8s-setup.yml`
```yaml
---
- name: Подготовка всех нод (Common)
  hosts: k8s_cluster
  become: yes
  roles:
    - common

- name: Инициализация мастера
  hosts: master
  become: yes
  roles:
    - master

- name: Присоединение воркеров
  hosts: workers
  become: yes
  roles:
    - worker

- name: Установка Calico (CNI)
  hosts: master
  become: yes
  roles:
    - calico

- name: Установка Ingress NGINX
  hosts: master
  become: yes
  roles:
    - ingress
```

### 2. `roles/common/tasks/main.yml`
```yaml
- name: Отключить swap
  command: swapoff -a
  changed_when: false

- name: Закомментировать swap в fstab
  replace:
    path: /etc/fstab
    regexp: '^([^#].*\sswap\s.*)$'
    replace: '# \1'

- name: Загрузить модули ядра
  community.general.modprobe:
    name: "{{ item }}"
    state: present
  loop: [overlay, br_netfilter]

- name: Сохранить модули
  copy:
    dest: /etc/modules-load.d/k8s.conf
    content: "overlay\nbr_netfilter\n"

- name: Настроить sysctl
  ansible.posix.sysctl:
    name: "{{ item.name }}"
    value: "{{ item.value }}"
    state: present
    reload: yes
  loop:
    - { name: 'net.bridge.bridge-nf-call-iptables', value: '1' }
    - { name: 'net.bridge.bridge-nf-call-ip6tables', value: '1' }
    - { name: 'net.ipv4.ip_forward', value: '1' }

- name: Добавить /etc/hosts
  lineinfile:
    path: /etc/hosts
    line: "{{ hostvars[item].ansible_host }} {{ item }}"
  loop: "{{ groups['k8s_cluster'] }}"

- name: Установить пакеты
  apt:
    name:
      - containerd
      - apt-transport-https
      - ca-certificates
      - curl
      - gpg
      - conntrack
      - socat
    state: present
    update_cache: yes

- name: Создать директорию для конфига containerd (фикс для Ubuntu 24.04)
  file:
    path: /etc/containerd
    state: directory
    mode: '0755'

- name: Сгенерировать containerd config
  shell: containerd config default > /etc/containerd/config.toml
  args:
    creates: /etc/containerd/config.toml

- name: Включить SystemdCgroup
  replace:
    path: /etc/containerd/config.toml
    regexp: 'SystemdCgroup = false'
    replace: 'SystemdCgroup = true'

- name: Настроить pause image
  replace:
    path: /etc/containerd/config.toml
    regexp: 'sandbox_image = .*'
    replace: 'sandbox_image = "registry.k8s.io/pause:3.9"'

- name: Запустить containerd
  systemd:
    name: containerd
    state: restarted
    enabled: yes

- name: Добавить ключ репозитория k8s
  shell: |
    mkdir -p /etc/apt/keyrings
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v{{ k8s_version }}/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
  args:
    creates: /etc/apt/keyrings/kubernetes-apt-keyring.gpg

- name: Добавить репозиторий k8s
  copy:
    dest: /etc/apt/sources.list.d/kubernetes.list
    content: "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v{{ k8s_version }}/deb/ /\n"

- name: Установить kubelet, kubeadm, kubectl
  apt:
    name: [kubelet, kubeadm, kubectl]
    state: present
    update_cache: yes

- name: Зафиксировать версии (hold)
  dpkg_selections:
    name: "{{ item }}"
    selection: hold
  loop: [kubelet, kubeadm, kubectl]
```

### 3. `roles/master/tasks/main.yml`
```yaml
- name: Проверить наличие admin.conf
  stat:
    path: /etc/kubernetes/admin.conf
  register: admin_conf

- name: kubeadm init
  command: >
    kubeadm init
    --control-plane-endpoint="{{ ansible_host }}:6443"
    --pod-network-cidr={{ pod_network_cidr }}
    --upload-certs
  when: not admin_conf.stat.exists

- name: Создать директорию .kube
  file:
    path: /home/ubuntu/.kube
    state: directory
    owner: ubuntu
    group: ubuntu

- name: Скопировать admin.conf
  copy:
    src: /etc/kubernetes/admin.conf
    dest: /home/ubuntu/.kube/config
    remote_src: yes
    owner: ubuntu
    group: ubuntu
    mode: '0600'

- name: Сгенерировать join-команду
  command: kubeadm token create --print-join-command
  register: join_cmd
  changed_when: false

- name: Сохранить join-команду для воркеров
  set_fact:
    kube_join_cmd: "{{ join_cmd.stdout }}"
```

### 4. `roles/worker/tasks/main.yml`
```yaml
- name: Проверить наличие kubelet.conf (уже в кластере?)
  stat:
    path: /etc/kubernetes/kubelet.conf
  register: kubelet_conf

- name: kubeadm join
  command: "{{ hostvars[groups['master'][0]]['kube_join_cmd'] }}"
  when: not kubelet_conf.stat.exists
```

### 5. `roles/calico/tasks/main.yml`
```yaml
- name: Установить Calico
  command: kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/calico.yaml
  environment:
    KUBECONFIG: /home/ubuntu/.kube/config

- name: Ждать, пока поды Calico станут Running
  shell: |
    kubectl wait --for=condition=Ready pods -l k8s-app=calico-node -n kube-system --timeout=300s
  environment:
    KUBECONFIG: /home/ubuntu/.kube/config
```

### 6. `roles/ingress/tasks/main.yml`
```yaml
- name: Установить Ingress NGINX
  command: kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/baremetal/deploy.yaml
  environment:
    KUBECONFIG: /home/ubuntu/.kube/config

- name: Ждать, пока поды Ingress станут Running
  shell: |
    kubectl wait --namespace ingress-nginx --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=300s
  environment:
    KUBECONFIG: /home/ubuntu/.kube/config

- name: Переключить Ingress на NodePort
  shell: |
    kubectl patch svc ingress-nginx-controller -n ingress-nginx --type merge -p '{"spec": {"type": "NodePort"}}'
  environment:
    KUBECONFIG: /home/ubuntu/.kube/config
```

---

# 🏁 Шаг 3. Запуск

1. Убедись, что на хосте установлены `terraform` и `ansible`.
2. Перейди в директорию с проектом.
3. Выполни инициализацию Terraform:
   ```bash
   terraform init
   ```
4. Запусти создание инфраструктуры и настройку:
   ```bash
   terraform apply
   ```
5. Terraform создаст VM, сгенерирует `inventory.ini` и автоматически запустит плейбук Ansible.
6. Проверь статус кластера:
   ```bash
   ssh ubuntu@10.10.10.10
   kubectl get nodes
   ```
   Все 3 ноды должны перейти в статус `Ready`.