## ansible使用基本介绍

### 1、ansible概述

Ansible 是一款开源运维自动化工具，通过Ansible可以实现运维自动化，提高运维工程师的工作效率，减少人为失误。

Ansible 通过本身集成的非常丰富的模块可以实现各种管理任务，其自带模块超过上千个。更为重要的是，它操作非常简单，即使小白也可以轻松上手，但它提供的功能又非常丰富，在运维领域，几乎可以做任何事。

#### 1.1、ansible特点

1. Ansible 基于 Python 开发，运维工程师对其二次开发相对比较容易；
2. Ansible 丰富的内置模块，几乎可以满足一切要求；
3. 管理模式非常简单，一条命令可以影响上千台主机；
4. 无客户端模式，底层通过 SSH 通信。

#### 1.2、ansible常用的两种交互方式

1. ad-hoc：用户直接通过ad-hoc命令集调用ansible工具集来完成任务；
2. playbook：用户预先编写好playbook，通过执行playbook中的任务集，按序执行任务。

### 2、ansible使用示例

#### 2.1、ad-hoc示例



#### 2.2、playbook布局

##### 2.2.1、全局目录结构

```shell
ansible-playbook/
├── group_vars
│   └── all
├── hosts
├── roles
│   └── zookeeper
│       ├── files
│       │   └── apache-zookeeper-3.7.0-bin.tar.gz
│       ├── handlers
│       │   └── main.yml
│       ├── tasks
│       │   └── main.yml
│       └── templates
│           ├── myid.j2
│           └── zoo.cfg.j2
└── site.yml
```

##### 2.2.2、二级目录结构

```shell
ansible-playbook/
├── group_vars
│   └── all
├── hosts
├── roles
│   └── zookeeper
└── site.yml
```

1. group_vars：目录默认的all文件定义了所有安装过程中的全局变量；
2. hosts：主机清单文件；
3. roles：定义了执行的实体内容，每个子目录代表一个部署实体的执行规则；
4. site.yml：这是一个入口文件，定义了主机组、执行顺序、以及对应的规则（roles）、添加标签（能够实现自由选择rules部署）

##### 2.2.3、三级目录结构

这一级的目录是ansible固化的目录结构，主要有files、handlers、tasks、templates四种，tasks必须有，其他三种根据脚本需要再决定要不要。

```shell
ansible-playbook/roles/zookeeper/
├── files
│   └── apache-zookeeper-3.7.0-bin.tar.gz
├── handlers
│   └── main.yml
├── tasks
│   └── main.yml
└── templates
    ├── myid.j2
    └── zoo.cfg.j2
```

1. files：放文件的地方，当前roles下默认文件源地址；
2. handler：定义当ansible执行状态为changed时触发的规则，和notify配合使用，必须有入口文件main.yml；
3. tasks：定义主要的主要的执行规则，必须有入口文件main.yml；
4. templates：使用jinja2的语法规则，存放j2模板的地方。

#### 2.2、playbook灵活执行

##### 2.2.1、使用自定义hosts

```shell
ansible-playbook -i hosts site.yml
```

##### 2.2.2、执行部分rules

```shell
ansible-playbook -i hosts site.yml --tags="install"
```

##### 2.2.3、跳过执行部分rules

```shell
ansible-playbook -i hosts site.yml --skip-tags="install,upgrade"
```

#### 2.3、playbook之zookeeper部署

##### 2.3.1、group_vars/all

```shell
group_vars/all 
#zookeeper vars
zoo_port1: 2888
zoo_port2: 3888
zoo_data_dir: /data/zookeeperData
zoo_listen_port: 2181
zookeeper_file: apache-zookeeper-3.7.0-bin.tar.gz
```

##### 2.3.2、handler/main.yml

```yaml
---
- name: start zookeeper
  shell: /bin/bash /usr/local/zookeeper/bin/zkServer.sh restart
```

##### 2.3.3、tasks/main.yml

```yaml
---
- name: Check Idle port for zookeeper
  shell: ss -ntl|grep -e {{ zoo_listen_port }} -e {{ zoo_port2 }}
  ignore_errors: True
  register: zoo_port

- name: WARN for port check
  debug:
    msg: "WARNING：Port {{ zoo_listen_port }} or {{ zoo_port2 }} is not idle, Service zookeeper will not be deploy"
  when: zoo_port == "0"

- name: Copy zookeeper pkgs to all hosts
  copy:
    src: "{{ zookeeper_file }}"
    dest: /tmp/{{ zookeeper_file }}
    mode: 0755
  when: zoo_port != "0"

- name: Unzip zookeeper pkgs in all hosts
  unarchive:
    src: /tmp/{{ zookeeper_file }}
    dest: /usr/local
    remote_src: yes
    mode: 0755
  when: zoo_port != "0"

- name: Register var zookeeper_path
  shell: echo {{ zookeeper_file }}|awk -F'.tar.gz' '{print $1}'
  register: zookeeper_path
  when: zoo_port != "0"

- name: Create symbolic link
  file:
   src: /usr/local/{{ zookeeper_path.stdout }}
   dest: /usr/local/zookeeper
   state: link

- name: Create a data_dir
  file:
    path: "{{ zoo_data_dir }}"
    state: directory
    mode: 0755
  when: zoo_port != "0"

- name: Config zookeeper myid
  template:
    src: myid.j2
    dest: "{{ zoo_data_dir }}/myid"
    mode: 0755
  when: zoo_port != "0"

- name: Config zookeeper service
  template:
    src: zoo.cfg.j2
    dest: /usr/local/zookeeper/conf/zoo.cfg
    mode: 0755
  notify: start zookeeper
  when: zoo_port != "0"
```

##### 2.3.4、templates/myid.j2

```jinja2
{{ host_id }}
```

##### 2.3.5、templates/zoo.cfg.j2

```jinja2
tickTime=2000
initLimit=10
syncLimit=5
dataDir={{ zoo_data_dir }}
clientPort={{ zoo_listen_port }}
{% for host in groups['zoo_hosts'] %}
server.{{ hostvars[host]['host_id'] }}={{ host }}:{{ zoo_port1 }}:{{ zoo_port2 }}
{% endfor %}
```

##### 2.3.6、运行结果

```shell
ansible-playbook -i hosts site.yml

PLAY [zoo_hosts] *********************************************************************************************************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************************************************************************************
ok: [192.168.31.4]
ok: [192.168.31.5]
ok: [192.168.31.3]

TASK [Check Idle port for zookeeper] *************************************************************************************************************************************************************************************************************
fatal: [192.168.31.4]: FAILED! => {"changed": true, "cmd": "ss -ntl|grep -e 2181 -e 3888", "delta": "0:00:00.004427", "end": "2021-07-02 11:57:39.293079", "msg": "non-zero return code", "rc": 1, "start": "2021-07-02 11:57:39.288652", "stderr": "", "stderr_lines": [], "stdout": "", "stdout_lines": []}
...ignoring
fatal: [192.168.31.5]: FAILED! => {"changed": true, "cmd": "ss -ntl|grep -e 2181 -e 3888", "delta": "0:00:00.004500", "end": "2021-07-02 11:57:39.310440", "msg": "non-zero return code", "rc": 1, "start": "2021-07-02 11:57:39.305940", "stderr": "", "stderr_lines": [], "stdout": "", "stdout_lines": []}
...ignoring
fatal: [192.168.31.3]: FAILED! => {"changed": true, "cmd": "ss -ntl|grep -e 2181 -e 3888", "delta": "0:00:00.015239", "end": "2021-07-02 11:57:39.359132", "msg": "non-zero return code", "rc": 1, "start": "2021-07-02 11:57:39.343893", "stderr": "", "stderr_lines": [], "stdout": "", "stdout_lines": []}
...ignoring

TASK [zookeeper : WARN for port check] ***********************************************************************************************************************************************************************************************************
skipping: [192.168.31.3]
skipping: [192.168.31.4]
skipping: [192.168.31.5]

TASK [Copy zookeeper pkgs to all hosts] **********************************************************************************************************************************************************************************************************
ok: [192.168.31.4]
ok: [192.168.31.3]
ok: [192.168.31.5]

TASK [Unzip zookeeper pkgs in all hosts] *********************************************************************************************************************************************************************************************************
changed: [192.168.31.5]
changed: [192.168.31.4]
changed: [192.168.31.3]

TASK [Register var zookeeper_path] ***************************************************************************************************************************************************************************************************************
changed: [192.168.31.5]
changed: [192.168.31.3]
changed: [192.168.31.4]

TASK [zookeeper : Create symbolic link] **********************************************************************************************************************************************************************************************************
changed: [192.168.31.4]
changed: [192.168.31.3]
changed: [192.168.31.5]

TASK [zookeeper : Create a data_dir] *************************************************************************************************************************************************************************************************************
changed: [192.168.31.4]
changed: [192.168.31.5]
changed: [192.168.31.3]

TASK [Config zookeeper myid] *********************************************************************************************************************************************************************************************************************
changed: [192.168.31.4]
changed: [192.168.31.5]
changed: [192.168.31.3]

TASK [Config zookeeper service] ******************************************************************************************************************************************************************************************************************
changed: [192.168.31.4]
changed: [192.168.31.3]
changed: [192.168.31.5]

RUNNING HANDLER [start zookeeper] ****************************************************************************************************************************************************************************************************************
changed: [192.168.31.4]
changed: [192.168.31.3]
changed: [192.168.31.5]

PLAY [zoo_hosts] *********************************************************************************************************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************************************************************************************
ok: [192.168.31.4]
ok: [192.168.31.5]
ok: [192.168.31.3]

TASK [Check Idle port for zookeeper] *************************************************************************************************************************************************************************************************************
changed: [192.168.31.4]
changed: [192.168.31.5]
changed: [192.168.31.3]

TASK [zookeeper : WARN for port check] ***********************************************************************************************************************************************************************************************************
skipping: [192.168.31.3]
skipping: [192.168.31.4]
skipping: [192.168.31.5]

TASK [Copy zookeeper pkgs to all hosts] **********************************************************************************************************************************************************************************************************
ok: [192.168.31.4]
ok: [192.168.31.5]
ok: [192.168.31.3]

TASK [Unzip zookeeper pkgs in all hosts] *********************************************************************************************************************************************************************************************************
ok: [192.168.31.4]
ok: [192.168.31.5]
ok: [192.168.31.3]

TASK [Register var zookeeper_path] ***************************************************************************************************************************************************************************************************************
changed: [192.168.31.4]
changed: [192.168.31.3]
changed: [192.168.31.5]

TASK [zookeeper : Create symbolic link] **********************************************************************************************************************************************************************************************************
ok: [192.168.31.4]
ok: [192.168.31.3]
ok: [192.168.31.5]

TASK [zookeeper : Create a data_dir] *************************************************************************************************************************************************************************************************************
ok: [192.168.31.4]
ok: [192.168.31.5]
ok: [192.168.31.3]

TASK [Config zookeeper myid] *********************************************************************************************************************************************************************************************************************
ok: [192.168.31.4]
ok: [192.168.31.3]
ok: [192.168.31.5]

TASK [Config zookeeper service] ******************************************************************************************************************************************************************************************************************
ok: [192.168.31.5]
ok: [192.168.31.4]
ok: [192.168.31.3]

PLAY RECAP ***************************************************************************************************************************************************************************************************************************************
192.168.31.3               : ok=19   changed=10   unreachable=0    failed=0    skipped=2    rescued=0    ignored=1   
192.168.31.4               : ok=19   changed=10   unreachable=0    failed=0    skipped=2    rescued=0    ignored=1   
192.168.31.5               : ok=19   changed=10   unreachable=0    failed=0    skipped=2    rescued=0    ignored=1   
```

```shell
[root@ansible-playbook]# /usr/local/zookeeper/bin/zkServer.sh status
/usr/bin/java
ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper/bin/../conf/zoo.cfg
Client port found: 2181. Client address: localhost. Client SSL: false.
Mode: follower
```