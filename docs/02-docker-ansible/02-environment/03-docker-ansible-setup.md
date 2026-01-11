### Docker и Ansible: установка и базовая настройка

#### Docker на Ubuntu 24.04 LTS

**Что это:**

- **Docker** — платформа контейнеризации приложений.
- Упаковывает приложение и зависимости в контейнер.
- Контейнер работает одинаково везде.

---

**1. Удалить старые версии Docker (если есть):**

```bash
sudo apt remove docker.io docker-compose docker-compose-v2 docker-doc podman-docker
```

---

**2. Обновить пакеты:**

```bash
sudo apt update && sudo apt upgrade -y
```

---

**3. Установить зависимости:**

```bash
sudo apt install -y ca-certificates curl
```

---

**4. Добавить GPG ключ Docker:**

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

---

**5. Добавить репозиторий Docker:**

```bash
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

---

**6. Установить Docker Engine:**

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

**Результат:** Docker daemon запустится автоматически.

---

**7. Проверить что Docker работает:**

```bash
sudo systemctl status docker
```

**Результат:** Должен вывести статус `active (running)`.

---

**8. Создать группу Docker (если ещё не создана):**

```bash
sudo groupadd docker
```

**Примечание:** На Ubuntu группа обычно создаётся автоматически при установке.

---

**9. Добавить пользователя в группу Docker:**

```bash
sudo usermod -aG docker $USER
newgrp docker
```

**Результат:** Теперь можешь использовать Docker без `sudo`.

---

**10. Проверить установку Docker:**

```bash
docker run hello-world
```

**Результат:** Скачает образ и выведет "Hello from Docker!".

---

**11. Проверить Docker Compose:**

```bash
docker compose version
```

**Результат:** Выведет версию Docker Compose (встроена в Docker 24.0+).

---

**12. Настроить автозагрузку Docker при старте системы:**

```bash
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

---

**13. Исправить права доступа (если были ошибки при sudo):**

```bash
sudo chown "$USER":"$USER" /home/"$USER"/.docker -R
sudo chmod g+rwx "$HOME/.docker" -R
```

---

**14. Запустить демо-контейнер:**

```bash
docker run -d -p 8080:80 nginx:latest
curl http://localhost:8080
docker stop $(docker ps -q)
```

**Результат:** Запустит nginx и покажет HTML страницу.

---

## Ansible на Ubuntu 24.04 LTS

**Что это:**

- **Ansible** — инструмент автоматизации инфраструктуры.
- Пишешь playbook (инструкции в YAML).
- Ansible выполняет на удаленных серверах по SSH.

---

**1. Установить Ansible:**

```bash
sudo apt install -y ansible
```

**Или через pip (более свежая версия):**
```bash
sudo apt install -y python3-pip
pip3 install --user ansible
```

---

**2. Проверить установку:**

```bash
ansible --version
```

**Результат:** Выведет версию Ansible.

---

**3. Создать структуру проекта Ansible:**

```bash
mkdir -p ~/ansible/{inventory,playbooks,roles}
```

---

**4. Создать файл инвентаря (список серверов):**

```bash
nano ~/ansible/inventory/hosts
```

**Содержание:**
```ini
[local]
localhost ansible_connection=local

[servers]
server1 ansible_host=192.168.1.100 ansible_user=ubuntu ansible_port=22
server2 ansible_host=192.168.1.101 ansible_user=ubuntu ansible_port=22

[docker_hosts]
server1
server2

[docker_hosts:vars]
ansible_python_interpreter=/usr/bin/python3
```

---

**5. Проверить доступ к хостам:**

```bash
ansible all -i ~/ansible/inventory/hosts -m ping
```

**Результат:** `pong` для каждого доступного хоста.

---

**6. Создать файл конфигурации Ansible:**

```bash
nano ~/ansible/ansible.cfg
```

**Содержание:**
```ini
[defaults]
inventory = inventory/hosts
host_key_checking = False
remote_user = ubuntu
private_key_file = ~/.ssh/id_ed25519
```

---

**7. Создать простой playbook для обновления системы:**

```bash
nano ~/ansible/playbooks/update.yml
```

**Содержание:**
```yaml
---
- name: Update system and install packages
  hosts: servers
  gather_facts: yes
  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
      become: yes
    
    - name: Upgrade packages
      apt:
        upgrade: dist
        autoremove: yes
        autoclean: yes
      become: yes
    
    - name: Install useful packages
      apt:
        name:
          - curl
          - wget
          - git
          - vim
        state: present
      become: yes
```

---

**8. Выполнить playbook:**

```bash
ansible-playbook playbooks/update.yml
```

**Результат:** Ansible подключится и выполнит задачи на всех серверах.

---

**9. Playbook для установки Docker на удалённых серверах:**

```bash
nano ~/ansible/playbooks/install-docker.yml
```

**Содержание:**
```yaml
---
- name: Install Docker Engine
  hosts: docker_hosts
  gather_facts: yes
  become: yes
  
  tasks:
    - name: Remove old Docker packages
      apt:
        name:
          - docker.io
          - docker-compose
          - podman-docker
        state: absent
    
    - name: Update apt cache
      apt:
        update_cache: yes
    
    - name: Install dependencies
      apt:
        name:
          - ca-certificates
          - curl
          - gnupg
        state: present
    
    - name: Create keyrings directory
      file:
        path: /etc/apt/keyrings
        state: directory
        mode: '0755'
    
    - name: Add Docker GPG key
      shell: |
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
        gpg --dearmor -o /etc/apt/keyrings/docker.asc
      args:
        creates: /etc/apt/keyrings/docker.asc
    
    - name: Add Docker repository
      shell: |
        echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
        https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
        tee /etc/apt/sources.list.d/docker.list > /dev/null
      args:
        creates: /etc/apt/sources.list.d/docker.list
    
    - name: Update apt cache
      apt:
        update_cache: yes
    
    - name: Install Docker Engine
      apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-buildx-plugin
          - docker-compose-plugin
        state: present
    
    - name: Enable Docker service
      systemd:
        name: docker
        enabled: yes
        state: started
    
    - name: Enable containerd service
      systemd:
        name: containerd
        enabled: yes
        state: started
    
    - name: Add user to docker group
      user:
        name: "{{ ansible_user }}"
        groups: docker
        append: yes
```

---

**10. Выполнить playbook для установки Docker:**

```bash
ansible-playbook playbooks/install-docker.yml
```

---

**11. Ad-hoc команда для проверки Docker на всех серверах:**

```bash
ansible docker_hosts -m shell -a "docker --version"
```

**Результат:** Выведет версию Docker на каждом сервере.

---

**12. Ad-hoc команда для запуска контейнера:**

```bash
ansible docker_hosts -m shell -a "docker run -d -p 8080:80 nginx:latest"
```

---

## Vagrant (опционально)

**Что это:**

- **Vagrant** — управление виртуальными машинами через код.
- Быстрое создание одинаковых VM для разработки.
- Интеграция с Ansible для автоматизации.

---

**1. Установить Vagrant:**

```bash
sudo apt install -y vagrant
```

---

**2. Создать проект Vagrant:**

```bash
mkdir ~/vagrant-project
cd ~/vagrant-project
vagrant init ubuntu/noble64
```

---

**3. Настроить Vagrantfile с Docker и Ansible:**

```bash
nano Vagrantfile
```

**Содержание:**
```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/noble64"
  
  config.vm.network "private_network", ip: "192.168.33.10"
  
  config.vm.provider "virtualbox" do |vb|
    vb.memory = 2048
    vb.cpus = 2
  end
  
  config.vm.provision "shell", inline: <<-SHELL
    apt-get update
    apt-get install -y curl git python3-pip
  SHELL
  
  # Интеграция с Ansible
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "../ansible/playbooks/install-docker.yml"
    ansible.inventory_path = "../ansible/inventory/hosts"
  end
end
```

---

**4. Запустить VM:**

```bash
vagrant up
```

---

**5. Подключиться к VM:**

```bash
vagrant ssh
```

---

**6. Информация о VM:**

```bash
vagrant status
vagrant global-status
```

---

**7. Остановить и удалить VM:**

```bash
vagrant halt        # Остановить
vagrant destroy     # Удалить
```

---
