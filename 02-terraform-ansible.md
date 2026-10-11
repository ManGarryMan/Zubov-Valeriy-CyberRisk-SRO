# 2. Автоматизация поднятия виртуальных машин: Terraform и Ansible

## Зачем автоматизировать

Ручная настройка виртуалок медленная, плохо воспроизводится и приводит к «снежинкам» — серверам, настроенным чуть по-разному, которые невозможно точно пересоздать. Подход Infrastructure as Code (IaC) описывает инфраструктуру кодом, который хранится в Git, проверяется ревью и запускается одной командой.

С точки зрения кибербезопасности IaC даёт:
- воспроизводимость: одинаковая конфигурация каждый раз, меньше ошибок и «забытых» настроек;
- аудит: история изменений в Git показывает, кто и что менял;
- быстрое восстановление после инцидента: заражённый сервер пересоздаётся, а не лечится;
- возможность автоматически проверять конфигурации на уязвимости до развёртывания (tfsec, Checkov, ansible-lint).

Риски: секреты (пароли, ключи, токены) нельзя хранить в репозитории открытым текстом; state-файл Terraform может содержать чувствительные данные.

## Два инструмента, две задачи

|             |                   Terraform                  |                            Ansible                            |
|-------------|----------------------------------------------|---------------------------------------------------------------|
|    Задача   |   Создать инфраструктуру (ВМ, сети, диски)   |      Настроить то, что внутри (ПО, конфиги, пользователи)     |
|    Стиль    | Декларативный: описываешь итоговое состояние | В основном процедурный: список задач (но модули идемпотентны) |
|     Язык    |                      HCL                     |                        YAML (playbook)                        |
| Подключение |     API провайдера (облако, гипервизор)      |                     SSH (агент не нужен)                      |
|  Состояние  |               Хранит state-файл              |                   Без состояния (stateless)                   |

Обычно их используют вместе: Terraform создаёт ВМ, Ansible настраивает.

## Terraform

### Основные понятия
- Provider — плагин для работы с конкретной платформой (AWS, Azure, Proxmox, libvirt, VirtualBox и др.).
- Resource — объект инфраструктуры (виртуальная машина, диск, сеть).
- Variable / Output — входные параметры и выводимые значения.
- State — файл, в котором Terraform хранит соответствие между кодом и реальными ресурсами.
- Module — переиспользуемый набор ресурсов.

### Рабочий цикл
```bash
terraform init      # скачать провайдеры
terraform plan      # показать, что будет изменено (ничего не меняет)
terraform apply     # применить изменения
terraform destroy   # удалить всё созданное
```

### Пример (libvirt/KVM, локальная виртуалка)
```hcl
terraform {
  required_providers {
    libvirt = {
      source = "dmacvicar/libvirt"
    }
  }
}

provider "libvirt" {
  uri = "qemu:///system"
}

resource "libvirt_volume" "vm_disk" {
  name   = "demo-vm.qcow2"
  source = "https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img"
  format = "qcow2"
}

resource "libvirt_domain" "demo" {
  name   = "demo-vm"
  memory = 2048
  vcpu   = 2

  disk {
    volume_id = libvirt_volume.vm_disk.id
  }

  network_interface {
    network_name   = "default"
    wait_for_lease = true
  }
}

output "vm_ip" {
  value = libvirt_domain.demo.network_interface[0].addresses[0]
}
```

Этот пример приведён для понимания структуры. Точные версии провайдера и параметры нужно сверять с документацией, они меняются.

### Заметка о лицензии
С 2023 года Terraform распространяется под лицензией BUSL (не полностью open source). Существует открытый форк **OpenTofu** с совместимым синтаксисом.

## Ansible

### Основные понятия
- Inventory — список управляемых хостов и групп.
- Playbook — YAML-файл со сценарием: на каких хостах что выполнить.
- Task — одно действие, использующее модуль.
- Module — готовая единица работы (`apt`, `copy`, `user`, `service` и т.д.).
- Role — структурированный переиспользуемый набор задач.
- Идемпотентность — повторный запуск не меняет систему, если она уже в нужном состоянии.

### Пример inventory (`inventory.ini`)
```ini
[webservers]
192.168.122.10 ansible_user=ubuntu
```

### Пример playbook (`site.yml`): установка nginx и базовое укрепление SSH
```yaml
---
- name: Базовая настройка веб-сервера
  hosts: webservers
  become: true

  tasks:
    - name: Обновить кэш пакетов
      ansible.builtin.apt:
        update_cache: true

    - name: Установить nginx
      ansible.builtin.apt:
        name: nginx
        state: present

    - name: Запустить и включить nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

    - name: Запретить вход root по SSH
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PermitRootLogin'
        line: 'PermitRootLogin no'
      notify: Перезапустить sshd

  handlers:
    - name: Перезапустить sshd
      ansible.builtin.service:
        name: ssh
        state: restarted
```

### Запуск
```bash
ansible -i inventory.ini all -m ping            # проверка связи
ansible-playbook -i inventory.ini site.yml      # выполнить playbook
ansible-playbook -i inventory.ini site.yml --check   # пробный запуск без изменений
```

## Связка: как это работает вместе

1. `terraform apply` создаёт ВМ и выдаёт её IP (output).
2. IP попадает в inventory Ansible (вручную, скриптом или через динамический inventory).
3. `ansible-playbook` настраивает систему: пакеты, пользователи, сервисы, hardening.

## Другие инструменты рядом

- Vagrant — быстро поднимает локальные ВМ (VirtualBox и др.) по файлу `Vagrantfile`; удобно для лабораторных.
- Packer — собирает готовые образы ВМ.
- cloud-init — первичная настройка ВМ при первом запуске (пользователи, ключи, пакеты).
- Pulumi, CloudFormation, Bicep — альтернативы Terraform.
- Puppet, Chef**, SaltStack — альтернативы Ansible.

## Безопасность IaC (кратко)

- Не коммитить секреты. Использовать переменные окружения, Ansible Vault, HashiCorp Vault, облачные менеджеры секретов.
- Добавить `.gitignore` для `*.tfstate`, `*.tfvars` с секретами, `.terraform/`.
- Удалённое хранилище state с шифрованием и блокировками.
- Принцип наименьших привилегий для ключей/токенов, которые использует Terraform.
- Сканировать код: `tfsec`, `checkov`, `ansible-lint`.
- Фиксировать версии провайдеров и модулей.
