## Цель работы
Научиться писать Molecule-тесты для ролей

## Ход работы
1. Написать Ansible Playbook поднимающий OpenVPN сервер
2. Через Molecule инициировать структуру для роли и вынести задачи поднятия сервера, с конфигурацией через переменные
3. При завершении работы плейбук должен разместить в playbook-dir артефактный ovpn-файл для подключения
4. Роль должна проходить стандартные тестами molecule test
5. Плейбук подключает роль через ansible-galaxy

## Выполнение лабораторной работы
1. Написать Ansible Playbook поднимающий OpenVPN сервер
Подниаем машины при помощи команды `vagrant up` и в директории `/roles` создаем новую роль openvpn:
```
ansible-galaxy init openvpn
```

В директории roles/openvpn/tasks добавим в main.yml задачи по установке и настройке openvpn сервера:
```
- name: Update apt cache
  apt:
    update_cache: yes
  become: yes
  changed_when: false

- name: Install OpenVPN and Easy-RSA
  apt:
    name:
      - openvpn
      - easy-rsa
    state: present
  become: yes

- name: Ensure Easy-RSA directory exists
  block:
    - name: Check if Easy-RSA directory exists
      stat:
        path: "{{ easy_rsa_path }}"
      register: easy_rsa_exists

    - name: Create Easy-RSA directory if it doesn't exist
      command: make-cadir "{{ easy_rsa_path }}"
      become: yes
      when: not easy_rsa_exists.stat.exists

- name: Ensure PKI directory exists
  block:
    - name: Check if PKI directory exists
      stat:
        path: "{{ easy_rsa_pki_path }}"
      register: pki_exists

    - name: Create PKI directory if it doesn't exist
      command: ./easyrsa init-pki
      args:
        chdir: "{{ easy_rsa_path }}"
      become: yes
      when: not pki_exists.stat.exists

- name: Create and manage CA and server certificates
  block:
    - name: Check if CA certificate already exists
      stat:
        path: "{{ easy_rsa_pki_path }}/ca.crt"
      register: ca_cert

    - name: Build CA without password and common name
      shell: echo "MyOpenVPN_CA" | ./easyrsa build-ca nopass
      args:
        chdir: "{{ easy_rsa_path }}"
      become: yes
      when: not ca_cert.stat.exists

    - name: Check if server key already exists
      stat:
        path: "{{ easy_rsa_pki_path }}/private/server.key"
      register: server_key

    - name: Generate server request without password
      shell: echo "MyOpenVPN_Server" | ./easyrsa gen-req server nopass
      args:
        chdir: "{{ easy_rsa_path }}"
      become: yes
      when: not server_key.stat.exists

    - name: Check if server certificate already exists
      stat:
        path: "{{ easy_rsa_pki_path }}/issued/server.crt"
      register: server_cert

    - name: Request sign for server certificate
      shell: echo "yes" | ./easyrsa sign-req server server
      args:
        chdir: "{{ easy_rsa_path }}"
      become: yes
      when: not server_cert.stat.exists

    - name: Check if Diffie-Hellman parameters already exist
      stat:
        path: "{{ easy_rsa_pki_path }}/dh.pem"
      register: dh_params

    - name: Generate Diffie-Hellman parameters
      command: ./easyrsa gen-dh
      args:
        chdir: "{{ easy_rsa_path }}"
      become: yes
      when: not dh_params.stat.exists

    - name: Check if TLS key already exists
      stat:
        path: "{{ easy_rsa_pki_path }}/ta.key"
      register: tls_key

    - name: Generate TLS key
      command: openvpn --genkey --secret pki/ta.key
      args:
        chdir: "{{ easy_rsa_path }}"
      become: yes
      when: not tls_key.stat.exists

- name: Ensure OpenVPN directory
  file:
    path: /etc/openvpn
    state: directory
    mode: '0755'
  become: yes

- name: Copy required files to OpenVPN directory
  copy:
    src: "{{ item.src }}"
    dest: "{{ item.dest }}"
    remote_src: yes
    force: no
  with_items:
    - { src: "{{ easy_rsa_pki_path }}/ca.crt", dest: "{{ openvpn_config_path }}/ca.crt" }
    - { src: "{{ easy_rsa_pki_path }}/issued/server.crt", dest: "{{ openvpn_config_path }}/server.crt" }
    - { src: "{{ easy_rsa_pki_path }}/private/server.key", dest: "{{ openvpn_config_path }}/server.key" }
    - { src: "{{ easy_rsa_pki_path }}/dh.pem", dest: "{{ openvpn_config_path }}/dh.pem" }
    - { src: "{{ easy_rsa_pki_path }}/ta.key", dest: "{{ openvpn_config_path }}/ta.key" }
  become: yes

- name: Apply OpenVPN server configuration
  template:
    src: server.conf.j2
    dest: /etc/openvpn/server.conf
  become: yes

#- name: Start OpenVPN server
#  systemd:
#    name: openvpn@server
#    state: started
#    enabled: yes

- name: Check if OpenVPN is running
  shell: pgrep -f "openvpn --config /etc/openvpn/server.conf"
  register: openvpn_process
  ignore_errors: true
  changed_when: false

- name: Start OpenVPN server manually
  command: openvpn --config /etc/openvpn/server.conf
  async: 30
  poll: 0
  become: yes
  when: openvpn_process.rc != 0

- name: Create client configuration file
  template:
    src: client.ovpn.j2
    dest: "client.ovpn"
```

Добавим шаблон для сервера, на котором мы разворачиваем роль openvpn в файле `/openvpn/templetes/server.conf.j2`:
```
port {{ openvpn_port }}
proto {{ openvpn_proto }}
dev {{ openvpn_dev }}
server {{ openvpn_server_subnet }}
ifconfig-pool-persist ipp.txt
push "route {{ openvpn_server_subnet }}"
user {{ openvpn_user }}
group {{ openvpn_group }}
{% if openvpn_persist_key %}
ca {{ openvpn_ca }}
cert {{ openvpn_cert }}
key {{ openvpn_key }}
dh {{ openvpn_dh }}
log-append {{ openvpn_log }}
persist-key
{% endif %}
{% if openvpn_persist_tun %}
persist-tun
{% endif %}
keepalive {{ openvpn_keepalive }}
status {{ openvpn_status_log }} 1
```

Также добавим шаблон для клиент-сервера в файле `/openvpn/templetes/client.ovpn.j2`:
```
client
proto {{ openvpn_proto }}
dev {{ openvpn_dev }}
remote {{ openvpn_server_subnet }}
nobind
persist-key
persist-tun
ca ca.crt
cert client.crt
key client.key
cipher AES-256-CBC
```

Параметрируем только что созданный шаблон в файле `/openvpn/vars/main.yml`:
```
openvpn_port: 1194
openvpn_proto: udp
openvpn_dev: tun
openvpn_user: nobody
openvpn_group: nogroup
openvpn_server_subnet: "10.15.0.0 255.255.255.0"
openvpn_ca: "/etc/openvpn/ca.crt"
openvpn_cert: "/etc/openvpn/server.crt"
openvpn_key: "/etc/openvpn/server.key"
openvpn_dh: "/etc/openvpn/dh.pem"
openvpn_status_log: "/var/log/openvpn/openvpn-status.log"
openvpn_log: "/var/log/openvpn/openvpn.log"
openvpn_persist_key: true
openvpn_persist_tun: true
openvpn_keepalive: "10 120"
```

2. Через Molecule инициировать структуру для роли и вынести задачи поднятия сервера, с конфигурацией через переменные
В папке `/roles/openvpn` инициируем сценарий для molecule при помощи команды:
```
molecule init scenario
```

Изменим конфигурационный файл `molecule.yml` для тестрования ansible роли:
```
dependency:
  name: galaxy
  options:
    ignore-certs: True
    ignore-errors: True
    role-file: requirements.yml

driver:
  name: docker

platforms:
  - name: instance
    image: geerlingguy/docker-ubuntu2004-ansible
    privileged: true
    pre_build_image: true

provisioner:
  name: ansible

verifier:
  name: ansible
```

Параметризируем плейбук, который разворачивает роль openvpn, следующими переменными в файле `/roles/openvpn/defaults/main.yml`:

```
openvpn_packages:
  - openvpn
  - easy-rsa

openvpn_config_path: /etc/openvpn/
easy_rsa_path: /etc/openvpn/easy-rsa
easy_rsa_pki_path: /etc/openvpn/easy-rsa/pki
```

3. При завершении работы плейбук должен разместить в playbook-dir артефактный ovpn-файл для подключения
Из домашней директории подключаемся к первому воркеру с мастер-ноды при помощи команды:
```
ssh vagrant@192.168.0.101
```
На клиент-сервере получает конфигурационный файл `client.ovpn` для подключения:
```
client
proto udp
dev tun
remote 10.15.0.0 255.255.255.0
nobind
persist-key
persist-tun
ca ca.crt
cert client.crt
key client.key
cipher AES-256-CBC
```

4. Роль должна проходить стандартные тестами molecule test
Для того, чтобы все молекул-тесты проходили, доработаем нашу роль в файле `/roles/openvpn/meta/main.yml`:
```
galaxy_info:
  author: Olga Konstantinova
  description: OpenVPN server setup role
  company: ITMO
  license: BSD
  min_ansible_version: 2.9
  platforms:
    - name: Ubuntu
      versions:
        - 20.04
  galaxy_tags:
    - vpn
    - openvpn
  role_name: openvpn
  namespace: itmo
dependencies: []
```

Запускаем тесты при помощи команды `molecule test`:
```
WARNING  Driver docker does not provide a schema.
INFO     default scenario test matrix: dependency, cleanup, destroy, syntax, create, prepare, converge, idempotence, side_effect, verify, cleanup, destroy
INFO     Performing prerun with role_name_check=0...
INFO     Running default > dependency
Starting galaxy role install process
- openvpn is already installed, skipping.
INFO     Dependency completed successfully.
WARNING  Skipping, missing the requirements file.
INFO     Running default > cleanup
WARNING  Skipping, cleanup playbook not configured.
INFO     Running default > destroy
INFO     Sanity checks: 'docker'

PLAY [Destroy] *****************************************************************

TASK [Set async_dir for HOME env] **********************************************
ok: [localhost]

TASK [Destroy molecule instance(s)] ********************************************
changed: [localhost] => (item=instance)

TASK [Wait for instance(s) deletion to complete] *******************************
FAILED - RETRYING: [localhost]: Wait for instance(s) deletion to complete (300 retries left).
ok: [localhost] => (item=instance)

TASK [Delete docker networks(s)] ***********************************************
skipping: [localhost]

PLAY RECAP *********************************************************************
localhost                  : ok=3    changed=1    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0

INFO     Running default > syntax

playbook: /home/vagrant/Devops_Labs/roles/openvpn/molecule/default/converge.yml
INFO     Running default > create

PLAY [Create] ******************************************************************

TASK [Set async_dir for HOME env] **********************************************
ok: [localhost]

TASK [Log into a Docker registry] **********************************************
skipping: [localhost] => (item=None)
skipping: [localhost]

TASK [Check presence of custom Dockerfiles] ************************************
ok: [localhost] => (item={'image': 'geerlingguy/docker-ubuntu2004-ansible', 'name': 'instance', 'pre_build_image': True, 'privileged': True})

TASK [Create Dockerfiles from image names] *************************************
skipping: [localhost] => (item={'image': 'geerlingguy/docker-ubuntu2004-ansible', 'name': 'instance', 'pre_build_image': True, 'privileged': True})
skipping: [localhost]

TASK [Synchronization the context] *********************************************
skipping: [localhost] => (item={'image': 'geerlingguy/docker-ubuntu2004-ansible', 'name': 'instance', 'pre_build_image': True, 'privileged': True})
skipping: [localhost]

TASK [Discover local Docker images] ********************************************
ok: [localhost] => (item={'changed': False, 'skipped': True, 'skip_reason': 'Conditional result was False', 'false_condition': 'not item.pre_build_image | default(false)', 'item': {'image': 'geerlingguy/docker-ubuntu2004-ansible', 'name': 'instance', 'pre_build_image': True, 'privileged': True}, 'ansible_loop_var': 'item', 'i': 0, 'ansible_index_var': 'i'})

TASK [Build an Ansible compatible image (new)] *********************************
skipping: [localhost] => (item=molecule_local/geerlingguy/docker-ubuntu2004-ansible)
skipping: [localhost]

TASK [Create docker network(s)] ************************************************
skipping: [localhost]

TASK [Determine the CMD directives] ********************************************
ok: [localhost] => (item={'image': 'geerlingguy/docker-ubuntu2004-ansible', 'name': 'instance', 'pre_build_image': True, 'privileged': True})

TASK [Create molecule instance(s)] *********************************************
changed: [localhost] => (item=instance)

TASK [Wait for instance(s) creation to complete] *******************************
FAILED - RETRYING: [localhost]: Wait for instance(s) creation to complete (300 retries left).
changed: [localhost] => (item={'failed': 0, 'started': 1, 'finished': 0, 'ansible_job_id': 'j10968399957.1918', 'results_file': '/home/vagrant/.ansible_async/j10968399957.1918', 'changed': True, 'item': {'image': 'geerlingguy/docker-ubuntu2004-ansible', 'name': 'instance', 'pre_build_image': True, 'privileged': True}, 'ansible_loop_var': 'item'})

PLAY RECAP *********************************************************************
localhost                  : ok=6    changed=2    unreachable=0    failed=0    skipped=5    rescued=0    ignored=0

INFO     Running default > prepare
WARNING  Skipping, prepare playbook not configured.
INFO     Running default > converge

PLAY [Converge] ****************************************************************

TASK [Gathering Facts] *********************************************************
ok: [instance]

TASK [openvpn : Update apt cache] **********************************************
ok: [instance]

TASK [openvpn : Install OpenVPN and Easy-RSA] **********************************
changed: [instance]

TASK [openvpn : Check if Easy-RSA directory exists] ****************************
ok: [instance]

TASK [openvpn : Create Easy-RSA directory if it doesn't exist] *****************
changed: [instance]

TASK [openvpn : Check if PKI directory exists] *********************************
ok: [instance]

TASK [openvpn : Create PKI directory if it doesn't exist] **********************
changed: [instance]

TASK [openvpn : Check if CA certificate already exists] ************************
ok: [instance]

TASK [openvpn : Build CA without password and common name] *********************
changed: [instance]

TASK [openvpn : Check if server key already exists] ****************************
ok: [instance]

TASK [openvpn : Generate server request without password] **********************
changed: [instance]

TASK [openvpn : Check if server certificate already exists] ********************
ok: [instance]

TASK [openvpn : Request sign for server certificate] ***************************
changed: [instance]

TASK [openvpn : Check if Diffie-Hellman parameters already exist] **************
ok: [instance]

TASK [openvpn : Generate Diffie-Hellman parameters] ****************************
changed: [instance]

TASK [openvpn : Check if TLS key already exists] *******************************
ok: [instance]

TASK [openvpn : Generate TLS key] **********************************************
changed: [instance]

TASK [openvpn : Ensure OpenVPN directory] **************************************
ok: [instance]

TASK [openvpn : Copy required files to OpenVPN directory] **********************
changed: [instance] => (item={'src': '/etc/openvpn/easy-rsa/pki/ca.crt', 'dest': '/etc/openvpn//ca.crt'})
changed: [instance] => (item={'src': '/etc/openvpn/easy-rsa/pki/issued/server.crt', 'dest': '/etc/openvpn//server.crt'})
changed: [instance] => (item={'src': '/etc/openvpn/easy-rsa/pki/private/server.key', 'dest': '/etc/openvpn//server.key'})
changed: [instance] => (item={'src': '/etc/openvpn/easy-rsa/pki/dh.pem', 'dest': '/etc/openvpn//dh.pem'})
changed: [instance] => (item={'src': '/etc/openvpn/easy-rsa/pki/ta.key', 'dest': '/etc/openvpn//ta.key'})

TASK [openvpn : Apply OpenVPN server configuration] ****************************
changed: [instance]

TASK [openvpn : Check if OpenVPN is running] ***********************************
ok: [instance]

TASK [openvpn : Start OpenVPN server manually] *********************************
skipping: [instance]

TASK [openvpn : Create client configuration file] ******************************
changed: [instance]

PLAY RECAP *********************************************************************
instance                   : ok=22   changed=11   unreachable=0    failed=0    skipped=1    rescued=0    ignored=0

INFO     Running default > idempotence

PLAY [Converge] ****************************************************************

TASK [Gathering Facts] *********************************************************
ok: [instance]

TASK [openvpn : Update apt cache] **********************************************
ok: [instance]

TASK [openvpn : Install OpenVPN and Easy-RSA] **********************************
ok: [instance]

TASK [openvpn : Check if Easy-RSA directory exists] ****************************
ok: [instance]

TASK [openvpn : Create Easy-RSA directory if it doesn't exist] *****************
skipping: [instance]

TASK [openvpn : Check if PKI directory exists] *********************************
ok: [instance]

TASK [openvpn : Create PKI directory if it doesn't exist] **********************
skipping: [instance]

TASK [openvpn : Check if CA certificate already exists] ************************
ok: [instance]

TASK [openvpn : Build CA without password and common name] *********************
skipping: [instance]

TASK [openvpn : Check if server key already exists] ****************************
ok: [instance]

TASK [openvpn : Generate server request without password] **********************
skipping: [instance]

TASK [openvpn : Check if server certificate already exists] ********************
ok: [instance]

TASK [openvpn : Request sign for server certificate] ***************************
skipping: [instance]

TASK [openvpn : Check if Diffie-Hellman parameters already exist] **************
ok: [instance]

TASK [openvpn : Generate Diffie-Hellman parameters] ****************************
skipping: [instance]

TASK [openvpn : Check if TLS key already exists] *******************************
ok: [instance]

TASK [openvpn : Generate TLS key] **********************************************
skipping: [instance]

TASK [openvpn : Ensure OpenVPN directory] **************************************
ok: [instance]

TASK [openvpn : Copy required files to OpenVPN directory] **********************
ok: [instance] => (item={'src': '/etc/openvpn/easy-rsa/pki/ca.crt', 'dest': '/etc/openvpn//ca.crt'})
ok: [instance] => (item={'src': '/etc/openvpn/easy-rsa/pki/issued/server.crt', 'dest': '/etc/openvpn//server.crt'})
ok: [instance] => (item={'src': '/etc/openvpn/easy-rsa/pki/private/server.key', 'dest': '/etc/openvpn//server.key'})
ok: [instance] => (item={'src': '/etc/openvpn/easy-rsa/pki/dh.pem', 'dest': '/etc/openvpn//dh.pem'})
ok: [instance] => (item={'src': '/etc/openvpn/easy-rsa/pki/ta.key', 'dest': '/etc/openvpn//ta.key'})

TASK [openvpn : Apply OpenVPN server configuration] ****************************
ok: [instance]

TASK [openvpn : Check if OpenVPN is running] ***********************************
ok: [instance]

TASK [openvpn : Start OpenVPN server manually] *********************************
skipping: [instance]

TASK [openvpn : Create client configuration file] ******************************
ok: [instance]

PLAY RECAP *********************************************************************
instance                   : ok=15   changed=0    unreachable=0    failed=0    skipped=8    rescued=0    ignored=0

INFO     Idempotence completed successfully.
INFO     Running default > side_effect
WARNING  Skipping, side effect playbook not configured.
INFO     Running default > verify
INFO     Running Ansible Verifier
WARNING  Skipping, verify action has no playbook.
INFO     Verifier completed successfully.
INFO     Running default > cleanup
WARNING  Skipping, cleanup playbook not configured.
INFO     Running default > destroy

PLAY [Destroy] *****************************************************************

TASK [Set async_dir for HOME env] **********************************************
ok: [localhost]

TASK [Destroy molecule instance(s)] ********************************************
changed: [localhost] => (item=instance)

TASK [Wait for instance(s) deletion to complete] *******************************
FAILED - RETRYING: [localhost]: Wait for instance(s) deletion to complete (300 retries left).
changed: [localhost] => (item=instance)

TASK [Delete docker networks(s)] ***********************************************
skipping: [localhost]

PLAY RECAP *********************************************************************
localhost                  : ok=3    changed=2    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0

INFO     Pruning extra files from scenario ephemeral directory
```
5. Плейбук подключает роль через ansible-galaxy

В файле `/openvpn/requirements.yml` указываем путь к гитхаб репозиторию, который хранит роль openvpn:
```
- src: git+https://github.com/okonstantinova/ansible-role-openvpn.git
  scm: git
  name: openvpn
```

И в файле `/openvpn/molecule/default/molecule.yml` подключаем requirements.yml:
```
dependency:
  name: galaxy
  options:
    ignore-certs: True
    ignore-errors: True
    role-file: requirements.yml

driver:
  name: docker

platforms:
  - name: instance
    image: geerlingguy/docker-ubuntu2004-ansible
    privileged: true
    pre_build_image: true

provisioner:
  name: ansible

verifier:
  name: ansible
```
