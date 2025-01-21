**Создание облачной инфраструктуры**

Для начала подготовим всю нашу инфрастркутуру в Yandex Cloud в Terraform-манифестах

1) Подготовим манифест для сервисного аккаунта для доступа к S3-бакету в котором будут удаленно хранится наши Terraform-стейты

```
# Создание сервисного аккаунта для Terraform
resource "yandex_iam_service_account" "service" {
  name      = var.account_name
  description = "service account to manage VMs"
  folder_id = var.folder_id
}

# Назначение роли editor сервисному аккаунту
resource "yandex_resourcemanager_folder_iam_member" "editor" {
  folder_id = var.folder_id
  role      = "editor"
  member    = "serviceAccount:${yandex_iam_service_account.service.id}"
  depends_on = [yandex_iam_service_account.service]
}

# Создание статического ключа доступа для сервисного аккаунта
resource "yandex_iam_service_account_static_access_key" "terraform_service_account_key" {
  service_account_id = yandex_iam_service_account.service.id
  description        = "static access key for object storage"
}

```

2) Далее подготовим манифест нашего бэкэнда для удаленного хранения стейтов в S3

```
# Создание объекта в существующей папке
resource "yandex_storage_object" "backend" {
  access_key = yandex_iam_service_account_static_access_key.terraform_service_account_key.access_key
  secret_key = yandex_iam_service_account_static_access_key.terraform_service_account_key.secret_key
  bucket = local.bucket_name
  key    = "terraform.tfstate"
  source = "./terraform.tfstate"
  depends_on = [yandex_storage_bucket.state_storage]
}
```


3) Создадим сеть и подсети для наших виртуальных машин в разных зонах доступности

```
#Создание пустой VPC
resource "yandex_vpc_network" "vpc0" {
  name = var.vpc_name
}

#Создадим в VPC subnet c названием subnet-a
resource "yandex_vpc_subnet" "subnet-a" {
  name           = var.subnet-a
  zone           = var.zone-a
  network_id     = yandex_vpc_network.vpc0.id
  v4_cidr_blocks = var.cidr-a
}

#Создание в VPC subnet с названием subnet-b
resource "yandex_vpc_subnet" "subnet-b" {
  name           = var.subnet-b
  zone           = var.zone-b
  network_id     = yandex_vpc_network.vpc0.id
  v4_cidr_blocks = var.cidr-b
}

#Создание в VPC subnet с названием subnet-d
resource "yandex_vpc_subnet" "subnet-d" {
  name           = var.subnet-d
  zone           = var.zone-d
  network_id     = yandex_vpc_network.vpc0.id
  v4_cidr_blocks = var.cidr-d
}
```

4) Создадим наш файл провайдера для доступа к Terraform

```
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
  required_version = ">=0.13"
}

provider "yandex" {
  token     = var.token
  cloud_id  = var.cloud_id
  folder_id = var.folder_id
  zone      = var.default_zone
}
```

5) Подготовим манифест для  создания нашего S3 бакета для хранения стейтов

```
# Создадим бакет с использованием ключа
resource "yandex_storage_bucket" "state_storage" {
  bucket     = local.bucket_name
  access_key = yandex_iam_service_account_static_access_key.terraform_service_account_key.access_key
  secret_key = yandex_iam_service_account_static_access_key.terraform_service_account_key.secret_key

  anonymous_access_flags {
    read = false
    list = false
  }
}

```



6) Заполним файл с перменными для получения всех значений в наших манифестах

```
#cloud vars
variable "token" {
  type        = string
  default     = "ЗДЕСЬ_МОГЛА_БЫТЬ_ВАША_РЕКЛАМА"
  description = "OAuth-token; https://cloud.yandex.ru/docs/iam/concepts/authorization/oauth-token"
}

variable "cloud_id" {
  type        = string
  default     = ""ЗДЕСЬ_МОГЛА_БЫТЬ_ВАША_РЕКЛАМА"
  description = "https://cloud.yandex.ru/docs/resource-manager/operations/cloud/get-id"
}

variable "folder_id" {
  type        = string
  default     = "ЗДЕСЬ_МОГЛА_БЫТЬ_ВАША_РЕКЛАМА"
  description = "https://cloud.yandex.ru/docs/resource-manager/operations/folder/get-id"

}

variable "default_zone" {
  type        = string
  default     = "ru-central1-a"
  description = "https://cloud.yandex.ru/docs/overview/concepts/geo-scope"
}

variable "account_name" {
  type        = string
  default     = "mezhibo-sa"
  description = "account_name"
}



variable "vpc_name" {
  type        = string
  default     = "vpc0"
  description = "VPC network"
}

variable "subnet-a" {
  type        = string
  default     = "subnet-a"
  description = "subnet name"
}

variable "subnet-b" {
  type        = string
  default     = "subnet-b"
  description = "subnet name"
}

variable "subnet-d" {
  type        = string
  default     = "subnet-d"
  description = "subnet name"
}

variable "zone-a" {
  type        = string
  default     = "ru-central1-a"
  description = "https://cloud.yandex.ru/docs/overview/concepts/geo-scope"
}

variable "zone-b" {
  type        = string
  default     = "ru-central1-b"
  description = "https://cloud.yandex.ru/docs/overview/concepts/geo-scope"
}

variable "zone-d" {
  type        = string
  default     = "ru-central1-d"
  description = "https://cloud.yandex.ru/docs/overview/concepts/geo-scope"
}

variable "cidr-a" {
  type        = list(string)
  default     = ["10.0.1.0/24"]
  description = "https://cloud.yandex.ru/docs/vpc/operations/subnet-create"
}

variable "cidr-b" {
  type        = list(string)
  default     = ["10.0.2.0/24"]
  description = "https://cloud.yandex.ru/docs/vpc/operations/subnet-create"
}

variable "cidr-d" {
  type        = list(string)
  default     = ["10.0.3.0/24"]
  description = "https://cloud.yandex.ru/docs/vpc/operations/subnet-create"
}


# Локальная переменная отвечающая за текущую дату в названии бакета
locals {
    current_timestamp = timestamp()
    formatted_date = formatdate("DD-MM-YYYY", local.current_timestamp)
    bucket_name = "state-storage-${local.formatted_date}"
}

variable "os_image_node" {
  type    = string
  default = "debian-12"
}
```


Манифесты готовы, теперь делаем Terraform apply и создаем наши ресурсы

![Image alt](https://github.com/mezhibo/Diplom/blob/1e248f8ced6090ba91fd1e59c430e38e2ca06422/IMG/1.jpg)


Видим что наша сеть и подсети создались (дефолтные создает сам яндекс в клауде, удалять не стал так все пересоздадутся, и работе не мешают)


Создался бакет для хранения


![Image alt](https://github.com/mezhibo/Diplom/blob/1e248f8ced6090ba91fd1e59c430e38e2ca06422/IMG/2.jpg)


Теперь через веб-интерфейс сходим в бакет, и увидим там файлы стейтов (состояний) Terraform


![Image alt](https://github.com/mezhibo/Diplom/blob/1e248f8ced6090ba91fd1e59c430e38e2ca06422/IMG/3.jpg)


Скачиваем его к себе и видим описание всех созданных нами ресурсов через Terraform


![Image alt](https://github.com/mezhibo/Diplom/blob/1e248f8ced6090ba91fd1e59c430e38e2ca06422/IMG/4.jpg)

Теперь также одной командой грохнем всю инфраструктуру и пойдем писать далее Terraform манифесты для виртуальных машин под наш k8s кластер.


![Image alt](https://github.com/mezhibo/Diplom/blob/1e248f8ced6090ba91fd1e59c430e38e2ca06422/IMG/5.jpg)


Первая часть диплома по подготовке инфраструктуры готова. Теперь можно приступать к созданию окружения на ВМ для развертывания на них k8s кластера.



**Создание Kubernetes кластера**

Выбрал создание k8s кластера через создание виртальных машин и Ansible с Kubespray, так как этот вариант максимально универсален и применим и для 

On-premise решения, которому компании доверяют больше и также можно развернуть on-cloud, что я собсвенно буду делать

Yandex Managed Service for Kubernetes от Yandex Cloud более узкнонаправленое и неуниверсальное средство развертывания, но у него ест и свои плюсы, в виде динамического создания и удаления нод в зависимости от нагрузки, чего нет в On-premise решениях и при работе с Yndex cloud это будет более дешевле содержать кластер из 1-3 нод которые то есть то нет от нагрузки, нежели платить сразу за 3 вм.

Я решил пойти по более универсальному пути, который плотно используется в работе на On-premise  и On-cloud


Создади 3 вм под наш кубер, 1 ноду мастера и 2 воркер ноды


Описываем tf мастер-ноду

```
# Ресурсы для создания master-node

# Создадим ресурс образа виртуальной машины для облачного сервиса Yandex Compute Cloud из существующего архива.
data "yandex_compute_image" "debian" {
  family = var.os_image_node
}


resource "yandex_compute_instance" "master-node" {
  name        = "${var.yandex_compute_instance_web[0].vm_name}-${count.index+1}"
  platform_id = var.yandex_compute_instance_web[0].platform_id

  count = var.yandex_compute_instance_web[0].count_vms

  resources {
    cores         = var.yandex_compute_instance_web[0].cores
    memory        = var.yandex_compute_instance_web[0].memory
    core_fraction = var.yandex_compute_instance_web[0].core_fraction
  }

  boot_disk {
    initialize_params {
      image_id = data.yandex_compute_image.debian.image_id
      type     = var.boot_disk_web[0].type
      size     = var.boot_disk_web[0].size
    }
  }

  metadata = {  
    serial-port-enable = 1
    ssh-keys = "debian:${file("~/.ssh/id_rsa.pub")}"
 }

  network_interface {
    subnet_id = yandex_vpc_subnet.subnet-a.id
    nat       = true
  }
  scheduling_policy {
    preemptible = true
  }
}
```

Теперь опишем воркер ноды

Так как воркер-нод у меня 2, самым правильным вариантом считаю создать ее через счетчик (count)

```
# Ресурсы для создания worker-node

resource "yandex_compute_instance" "worker-node" {
  name        = "${var.yandex_compute_instance_worker[0].vm_name}-${count.index+1}"
  platform_id = var.yandex_compute_instance_worker[0].platform_id

  count = var.yandex_compute_instance_worker[0].count_vms

  resources {
    cores         = var.yandex_compute_instance_worker[0].cores
    memory        = var.yandex_compute_instance_worker[0].memory
    core_fraction = var.yandex_compute_instance_worker[0].core_fraction
  }

  boot_disk {
    initialize_params {
      image_id = data.yandex_compute_image.debian.image_id
      type     = var.boot_disk_worker[0].type
      size     = var.boot_disk_worker[0].size
    }
  }

  metadata = {  
    serial-port-enable = 1
    ssh-keys = "debian:${file("~/.ssh/id_rsa.pub")}"
 }

  network_interface {
    subnet_id = yandex_vpc_subnet.subnet-a.id
    nat       = true
  }
  scheduling_policy {
    preemptible = true
  }
}
```

Теперь дополним наш файл с переменнынми нашими map-переменными для описания ресурсов наших виртуальных машин.

Добавим код ниже

```
# Переменные для master-node

variable "yandex_compute_instance_web" {
  type        = list(object({
    vm_name = string
    cores = number
    memory = number
    core_fraction = number
    count_vms = number
    platform_id = string
  }))

  default = [{
      vm_name = "master"
      cores         = 2
      memory        = 4
      core_fraction = 20
      count_vms = 1
      platform_id = "standard-v1"
    }]
}

variable "boot_disk_web" {
  type        = list(object({
    size = number
    type = string
    }))
    default = [ {
    size = 30
    type = "network-hdd"
  }]
}


# Переменные для worker-node

variable "yandex_compute_instance_worker" {
  type        = list(object({
    vm_name = string
    cores = number
    memory = number
    core_fraction = number
    count_vms = number
    platform_id = string
  }))

  default = [{
      vm_name = "worker"
      cores         = 4
      memory        = 6
      core_fraction = 20
      count_vms = 2
      platform_id = "standard-v1"
    }]
}

variable "boot_disk_worker" {
  type        = list(object({
    size = number
    type = string
    }))
    default = [ {
    size = 30
    type = "network-hdd"
  }]
}
```


Теперь будем подготавливать наш файл авто-инвентаризации хостов


Первым этапом подготовим terrafrom-output созданных нами ВМ

```
output "master-node" {
  value = flatten([
    [for i in yandex_compute_instance.master-node : {
      name = i.name
      ip_external   = i.network_interface[0].nat_ip_address
      ip_internal = i.network_interface[0].ip_address
    }],
  ])
}

output "worker-node" {
  value = flatten([
    [for i in yandex_compute_instance.worker-node : {
      name = i.name
      ip_external   = i.network_interface[0].nat_ip_address
      ip_internal = i.network_interface[0].ip_address
    }],
  ])
}
```

Далее опишем манифест создания нашего инвентори-файла на основе шаблона


```
resource "local_file" "hosts_yml_kubespray" {

  content  = templatefile("${path.module}/hosts.tftpl", {
    workers = yandex_compute_instance.worker-node
    masters = yandex_compute_instance.master-node
  })
  filename = "../ansible/inventory/hosts.yml"
}
```

Теперь подготовим сам шаблон в формате tftpl в который и будут приниматься значения terraform-output

```
all:
  hosts:%{ for idx, master-node in masters }
    master-${idx + 1}:
      ansible_host: ${master-node.network_interface[0].nat_ip_address}
      ip: ${master-node.network_interface[0].ip_address}
      access_ip: ${master-node.network_interface[0].ip_address}%{ endfor }%{ for idx, worker-node in workers }
      ansible_user: debian
    worker-${idx + 1}:
      ansible_host: ${worker-node.network_interface[0].nat_ip_address}
      ip: ${worker-node.network_interface[0].ip_address}
      access_ip: ${worker-node.network_interface[0].ip_address}%{ endfor }
      ansible_user: debian
  children:
    kube_control_plane:
      hosts:%{ for idx, master-node in masters }
        ${master-node.name}:%{ endfor }
    kube_node:
      hosts:%{ for idx, worker-node in workers }
        ${worker-node.name}:%{ endfor }
    etcd:
      hosts:%{ for idx, master-node in masters }
        ${master-node.name}:%{ endfor }
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
    calico_rr:
      hosts: {}
```

Все, на этом этапе работа с созданием облычных ресурсов в Yandex-cloud через terraform окончена

Далее начнем работать со средсвом управления конфигурацией Ansible

Выполянем terraform apply и создаем теперь полностью готовое окружение где будем разворчивать k8s и в него наше приложение и приложения мониторинга

![Image alt](https://github.com/mezhibo/Diplom/blob/1e248f8ced6090ba91fd1e59c430e38e2ca06422/IMG/6.jpg)


Все, видим что ресурсы созданы, оутпуты показаны, идем проверим наш файл авто-инвентори, как он заполнился

![Image alt](https://github.com/mezhibo/Diplom/blob/1e248f8ced6090ba91fd1e59c430e38e2ca06422/IMG/7.jpg)

Все, видим что все отлично, теперь подготовим Ansible - плейбук, который подготовит наш мастер-хост к настройке воркер-нод через kubespray


```
- name: Установка pip
  hosts: master-1
  become: true

  tasks:

    - name: Скачиваем файл get-pip.py
      ansible.builtin.get_url:
        url: https://bootstrap.pypa.io/get-pip.py
        dest: "./"

    - name: Удаляем EXTERNALLY-MANAGED
      ansible.builtin.file:
        path: /usr/lib/python3.11/EXTERNALLY-MANAGED
        state: absent

    - name: Устанавливаем pip
      ansible.builtin.shell: python3.11 get-pip.py



- name: Установка зависимостей из ansible-playbook kubespray
  hosts: master-1
  become: true

  tasks:

    - name: Выполнение apt update и установка git
      ansible.builtin.apt:
        update_cache: true
        pkg:
        - git

    - name: Клонируем kubespray из репозитория
      ansible.builtin.git:
        repo: https://github.com/kubernetes-sigs/kubespray.git
        dest: ./kubespray

    - name: Изменение прав на папку kubspray
      ansible.builtin.file:
        dest: ./kubespray
        recurse: yes
        owner: debian
        group: debian

    - name: Установка зависимостей из requirements.txt
      ansible.builtin.pip:
        requirements: /home/debian/kubespray/requirements.txt
        extra_args: -r /home/debian/kubespray/requirements.txt

    - name: Копирование содержимого папки inventory/sample в папку inventory/mycluster
      ansible.builtin.copy:
        src: /home/debian/kubespray/inventory/sample/
        dest: /home/debian/kubespray/inventory/mycluster/
        remote_src: true


- name: Подготовка master-node к установке kubespray из ansible-playbook
  hosts: master-1
  become: true

  tasks:

    - name: Копирование на master-node файла hosts.yml
      ansible.builtin.copy:
        src: ./inventory/hosts.yml
        dest: ./kubespray/inventory/mycluster/

    - name: Копирование на мастер приватного ключа
      ansible.builtin.copy:
        src: /root/.ssh/id_rsa
        dest: ./.ssh
        owner: debian
        mode: '0600'
```

Роль готова, теперь идем посыпать Ансиблом нашу будущую мастер-ноду


Выполняем запуск роли

```
ansible-playbook -i inventory/hosts.yml site.yml
```

![Image alt]([скрин8](https://github.com/mezhibo/Diplom/blob/1e248f8ced6090ba91fd1e59c430e38e2ca06422/IMG/8.jpg))



Видим что все таски по настройке нашей мастер-ноды прошли успешно, подключаемся к мастер-ноде и на ней запускаем ansible-role 
для настройки k8s-кластера через kubespray


Перестрахуемся и дадим права пользователю debian на репу kubespray, чтобы у нас не падали ансибл таски при настройке кластера (уже проверено))))


```
sudo chown -R debian:debian ~/kubespray
```

Переходим в репу и запускаем роль

```
ansible-playbook -i inventory/mycluster/hosts.yml cluster.yml -b -v
```

Видим что пошел процесс развертывания кластера k8s

![Image alt](https://github.com/mezhibo/Diplom/blob/1e248f8ced6090ba91fd1e59c430e38e2ca06422/IMG/9.jpg)

Процесс достаточно долгий, минут 20 точно.



