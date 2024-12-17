## **Ход работы**

1. Развернуть виртуальную машину для приложения (app) и виртуальную машину для будущей системы базы данных (db) через Vagrantfile
2. Написать роль установки **PostgreSQL** и плейбук, который ее разворачивает
3. Все параметры должны быть вынесены в переменные, все переменные должны быть префиксованы, значения переменных устанавливаются через group_vars для плейбука, роль должна быть покрыта тестами
4. Добавить возможность смены директории с данными на кастомную
5. Добавить возможность создания баз данных и пользователей
6. ***Добавить функционал настройки streaming-репликации***
7. ***Продумать логику определения master и replica нод СУБД и их настройки при работе роли***

## **Выполнение лабораторной работы**

### 1) Развернуть виртуальную машину для приложения (app) и виртуальную машину для будущей системы базы данных (db) через Vagrantfile:

Так как в дальнейшем нам понадобится 3 сервера, один для приложения (app), второй для master node (master), а также третий для replica node (replica), немного изменим файл hosts, создав группы для соответствующих серверов:

![image](https://github.com/user-attachments/assets/d35083d8-ede7-40c9-94fb-d54e814d92a3)

### 2) Написать роль установки **PostgreSQL** и плейбук, который ее разворачивает:

Создадим роль postgresql:

![image](https://github.com/user-attachments/assets/f21c9165-99df-425b-8c02-7a3b49a3eb68)

Playbook для развертывания роли:

```
---
# Общие задачи для всех серверов
- name: Ensure Python is installed
  raw: test -e /usr/bin/python || (apt -y update && apt install -y python3)
  changed_when: false

- name: Install necessary system packages
  apt:
    name:
      - "{{ postgresql_common_package }}"
      - gnupg
      - curl
      - ca-certificates
      - python3-pip
      - acl
    state: present
    update_cache: yes

- name: Install psycopg2-binary for app servers
  pip:
    name: psycopg2-binary
    state: present
    executable: /usr/bin/pip3

# Установить PostgreSQL-клиент и утилиты на app
- name: Install PostgreSQL client on app servers
  apt:
    name: postgresql-client
    state: present
  when: "'app' in group_names"

# Задачи для master и replica
- name: Install PostgreSQL server and configure common settings
  block:
    - name: Run PostgreSQL PGDG script
      shell: "{{ postgresql_repo_path }} -y"
      args:
        creates: /etc/apt/sources.list.d/pgdg.list
      changed_when: false

    - name: Create directory for PostgreSQL repository key
      file:
        path: /usr/share/postgresql-common/pgdg
        state: directory
        mode: '0755'
      changed_when: false

    - name: Download the PostgreSQL repository signing key
      get_url:
        url: "{{ postgresql_key_url }}"
        dest: "{{ postgresql_key }}"
        mode: '0644'
      changed_when: false

    - name: Check if PGDG repository exists
      stat:
        path: "{{ postgresql_sources_list }}"
      register: pgdg_repo

    - name: Add PostgreSQL repository to sources list
      shell: |
        echo "deb [signed-by={{ postgresql_key }}] {{ postgresql_repo_url }} $(lsb_release -cs)-pgdg main" > {{ postgresql_sources_list }}
      when: not pgdg_repo.stat.exists
      changed_when: false

    - name: Update package list
      apt:
        update_cache: yes

    - name: Install PostgreSQL server
      apt:
        name: "{{ postgresql_package }}"
        state: present
        force: yes

    - name: Configure pg_hba.conf
      template:
        src: templates/pg_hba.conf
        dest: "{{ pg_hba_conf_path }}"
        owner: postgres
        group: postgres
        mode: '0644'
      notify:
        - Restart PostgreSQL service
  when: "'master' in group_names or 'replica' in group_names"

# Конфигурация Master Node
- name: Configure master node
  block:
    - name: Ensure PostgreSQL service is running on master
      shell: |
        pg_ctlcluster {{ postgresql_version }} main start
      args:
        executable: /bin/bash
      register: postgresql_start_master
      changed_when: "'server starting' in postgresql_start_master.stdout"
      failed_when: "'already running' not in postgresql_start_master.stdout and postgresql_start_master.rc != 0 and 'server starting' not in postgresql_start_master.stdout"

    - name: Configure postgresql.conf for Master
      blockinfile:
        path: "{{ pg_conf_path }}"
        block: |
          listen_addresses = '{{ postgresql_master_listen_addresses }}'
          wal_level = {{ postgresql_master_wal_level }}
          archive_mode = {{ postgresql_master_archive_mode }}
          archive_command = '{{ postgresql_master_archive_command }}'
          max_wal_senders = {{ postgresql_master_max_wal_senders }}
          hot_standby = {{ postgresql_master_hot_standby }}
      changed_when: false

    - name: Create PostgreSQL database
      become_user: postgres
      postgresql_db:
        name: "{{ postgresql_db_name }}"
      register: db_creation
      changed_when: db_creation.changed

    - name: Create PostgreSQL user
      become_user: postgres
      postgresql_user:
        name: "{{ postgresql_user_name }}"
      register: user_creation
      changed_when: user_creation.changed

    - name: Restart PostgreSQL service on master
      shell: |
        pg_ctlcluster {{ postgresql_version }} main restart
      args:
        executable: /bin/bash
      register: restart_result
      changed_when: "'server starting' in restart_result.stdout or 'restarted' in restart_result.stdout"
      failed_when: "'stopped' in restart_result.stdout or restart_result.rc != 0"
  when: "'master' in group_names"

# Конфигурация Replica Node
- name: Configure replica node
  block:
    - name: Check PostgreSQL service status on replica
      shell: |
        pg_lsclusters | grep "{{ postgresql_version }}" | grep main | awk '{print $3}'
      args:
        executable: /bin/bash
      register: replica_status
      changed_when: false

    - name: Stop PostgreSQL service for replica if running
      shell: |
        pg_ctlcluster {{ postgresql_version }} main stop
      args:
        executable: /bin/bash
      register: replica_stop
      changed_when: "'server stopped' in replica_stop.stdout"
      failed_when: "'is not running' not in replica_stop.stdout and replica_stop.rc != 0"
      when: replica_status.stdout.strip() == "online"

    - name: Check if PostgreSQL data directory exists
      stat:
        path: "{{ postgresql_data_dir }}"
      register: data_dir_status
      become_user: postgres

    - name: Remove existing PostgreSQL data and create new directory
      shell: |
        rm -rf {{ postgresql_data_dir }} &&
        mkdir -p {{ postgresql_data_dir }} &&
        chmod go-rwx {{ postgresql_data_dir }}
      args:
        executable: /bin/bash
      become_user: postgres
      when: data_dir_status.stat.exists
      changed_when: false

    - name: Check if standby.signal exists
      stat:
        path: "{{ postgresql_data_dir }}/standby.signal"
      register: standby_signal_check
      become_user: postgres

    - name: Perform base backup from Master to Replica
      command: >
        pg_basebackup -P -R -X stream -c fast -h {{ master_host }} -U {{ replication_user }} -D "{{ postgresql_data_dir }}"
      args:
        creates: "{{ postgresql_data_dir }}/standby.signal"
      become_user: postgres
      when: not standby_signal_check.stat.exists
      changed_when: false

    - name: Configure postgresql.conf for Replica
      blockinfile:
        path: "{{ pg_conf_path }}"
        block: |
          primary_conninfo = 'host={{ master_host }} port=5432 user={{ replication_user }}'
          hot_standby = on
      changed_when: false

    - name: Ensure no existing PostgreSQL processes conflict on replica
      shell: |
        ps aux | grep '[p]ostgres' | awk '{print $2}' | xargs -r kill -9
        ps aux | grep '[p]ostgres' | wc -l
      args:
        executable: /bin/bash
      register: postgres_kill
      ignore_errors: true
      changed_when: postgres_kill.stdout | int > 0

    - name: Start PostgreSQL service for replica
      shell: |
        pg_ctlcluster {{ postgresql_version }} main start
      args:
        executable: /bin/bash
      register: replica_start
      changed_when: "'server starting' in replica_start.stdout"
      failed_when: "'already running' not in replica_start.stdout and 'port conflict' not in replica_start.stderr and replica_start.rc != 0"
  when: "'replica' in group_names"
```

### 3) Все параметры должны быть вынесены в переменные, все переменные должны быть префиксованы, значения переменных устанавливаются через group_vars для плейбука, роль должна быть покрыта тестами

Параметризируем playbook:

![image](https://github.com/user-attachments/assets/68f4f898-37a3-48d0-8018-1180fd85bb13)

Значения для переменных, установленные через group_vars:
1) Для серверов группы all:

![image](https://github.com/user-attachments/assets/8a3e4661-4372-46a6-ba69-586ecf84b8bf)

2) Для серверов группы master:

![image](https://github.com/user-attachments/assets/ee5a45e9-2a7d-4318-9774-a681af661a05)

3) Для серверов группы replica:

![image](https://github.com/user-attachments/assets/14dd1a16-d6f6-4aa8-931f-0a1c78cb54ee)

Все molecule test проходят успешно:

![image](https://github.com/user-attachments/assets/cff1811d-628c-46e2-8fc9-a3c87bcd7730)
![image](https://github.com/user-attachments/assets/df664648-887c-4981-ab12-2068ac99f90f)
![image](https://github.com/user-attachments/assets/2e6c3db2-38d7-471d-b36b-30799823c17f)

Идемпотентность:

![image](https://github.com/user-attachments/assets/ebeb08de-639e-4303-a57b-97aef2eb1e72)
![image](https://github.com/user-attachments/assets/22dc5c7e-2925-4b9e-9b7e-2e484735558e)
![image](https://github.com/user-attachments/assets/4ccff534-5ed5-402d-8123-ac92fcba095e)

Дестрой:

![image](https://github.com/user-attachments/assets/9efcaf63-8a5a-4127-a4eb-2223b7ebb277)

### 4) Добавить возможность смены директории с данными на кастомную

Данная переменная отвечает за кастомную директорию, в которой будут храниться данные:

![image](https://github.com/user-attachments/assets/37b98249-e686-48bf-9f99-7d51d07621a7)

### 5) Добавить возможность создания баз данных и пользователей

Только что была добавлена новая база данных, под названием "test_master", она успешно создалась, можем увидеть ее в списке всех бд:

![image](https://github.com/user-attachments/assets/881a1b39-bccc-4bb6-a8e2-f5aec8159a59)

Также попробуем добавить нового пользователя:

Сейчас нового пользователя нет в списке:

![image](https://github.com/user-attachments/assets/aae3d4b0-88a0-4fa7-9113-9daa7f094faf)

Создадим нового пользователя, дадим ему права для новой базы данных, а также подключимся к этой базе данных:

![image](https://github.com/user-attachments/assets/795a8aff-d7f0-4496-9e37-59f167d149b1)

Дадим права для чтения и записи данных:

![image](https://github.com/user-attachments/assets/a0db3c2c-b2a9-4098-a88a-d704f3f9de08)

Можем увидеть, что пользователь создался, с правами на тестовую базу данных:

![image](https://github.com/user-attachments/assets/35297bc8-6b88-4981-8c96-8726cbaf9309)

### 6) Добавить функционал настройки streaming-репликации

Чтобы проверить, что наша роль корректно настроила streaming-репликацию, проделаем следующие действия:

Нами была добавлена новая база данных на master node, которая называется "test_master":

![image](https://github.com/user-attachments/assets/dbdb21e2-a8c1-45f8-a70b-2c4f1cd294af)

Подключимся к "test_master" и добавим новые записи в таблицу:

![image](https://github.com/user-attachments/assets/2480f318-3fda-4d3b-9435-584c8cb7b745)

Были добавлены 2 записи: "Replication test 1" и "Replication test 2"

Теперь подключимся к replica node и проверим, есть ли эти же записи в этой базе данных:

![image](https://github.com/user-attachments/assets/f69359fd-7009-4963-aa36-fec679052624)

Можем увидеть, что они успешно перенеслись на replica node

Также правильность настройки нашей streaming-репликации подтверждает тот факт, что нельзя создать какую либо базу данных на replica node, так как она настроена только на чтение данных с master node:

![image](https://github.com/user-attachments/assets/28117c3c-8437-42c3-86b8-fadf9acd2df9)

### 7) Продумать логику определения master и replica нод СУБД и их настройки при работе роли

Вся логика и настройка нашей роли хранится в папке tasks, в файле main.yml, там же можно увидеть конфигурацию наших master и replica node
