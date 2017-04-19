=== Working in Progress ===


# Manage DataStax Enterprise (DSE) Installation and Version Upgrade Using Ansible Playbook

## 1. Overview

It is not a trivial task to provision a large DSE cluster, to manage the key configuration item (e.g. dse.yaml, cassandra.yaml, etc.) changes, and to do version upgrade.  Historically (pre DSE 5.0), there is really no out-of-the-box method to manage these tasks automatically (as much as possible). It is operationally a time-consuming and error-prone work to manually run these tasks on every DSE node instance, not even to mention that for many cases, you also have to consider the task execution sequence among different DES nodes (e.g. DC by DC, rack by rack, seed vs. no-seed and so on).

Starting from DSE 5.0, OpsCenter 6.0 (as part of DSE 5.0 release) introduces a new component called Life Cycle Manager (LCM) that is designed for efficient installation and configuration of a DSE cluster through a web based GUI. It aims to solve the above issue as an easy-to-use, out-of-the-box solution offered by DataStax. Although there are some limitations with the current version (OpsCenter 6.0.8 as the time of this writing), it is in general a good solution for automatic DSE cluster provisioning and configuration management. For more detailed information about this product, please reference to LCM's documentation page (https://docs.datastax.com/en/latest-opscenter/opsc/LCM/opscLCM.html). 

Unfortunately, while the current version of LCM does a great job in provisioning new cluster and managing configuration changes, it does not support DSE version upgrade yet. Meanwhile, the way that LCM schedules the execution of a job (provisioning new node or making configuration change) across the entire DSE cluster is in serial mode. This means that before finishing the execution of a job on one DSE node, LCM won't kick off the job on another node. Although correct, such a scheduling mechanism is not flexible and less efficient. 

In an attempt to overcome the limitations of the (current version of) LCM and bring some flexibility in job execution scheduling across the entire DSE cluster, I created an Ansible framework (playbook) for:
* Installing a brand new DSE cluster
* Upgrading an existing DSE cluster to a newer version 

**Note 1**: The framework is currently based on Debian packaged DSE installation or upgrade.

**Note 2**: This framework does have some configuration management capabilities (e.g. to make changes in cassandra.yaml file), but it is not yet a general tool to manage all DSE components related configuration files. However, using Ansible makes this job much easier and quite straightforward. 

## 2. Ansible Introduction  

Unlike some other orchestration and configuration management (CM) tools like Puppet or Chef, Ansible is very lightweight. It doesn't require any CM related modeuls, daemons, or agents to be installed on the managed machines. It is based on Python and pushes YAML based script commands over SSH protocol to remote hosts for execution. Ansible only needs to be installed on one workstation machine (could be your laptop) and as long as the SSH access from the workstation to the managed machines is enabled, Ansible based orchestration and CM will work.

Ansible uses an ***inventory*** file to group the systems or host machines to be managed. Ansible commands (either through ad-hoc or in Ansible ***module***) are executed against the systems or host machines specified in the inventory file. For more advanced orchestration and/or CM scenario, Ansible ***playbook*** is always used. By using the original words from Ansible [documentation](http://docs.ansible.com/ansible/intro.html), the relationship among Ansible inventory, module, and playbook can be summarized as:

> *If Ansible modules are the tools in your workshop, playbooks are your instruction manuals, and your inventory of hosts are your raw material.*

## 3. DSE Install/Upgrade Framework

This framework is based on Ansible playbook. Following the Ansible [best practice principles](http://docs.ansible.com/ansible/playbooks_best_practices.html), the framework playbook directory structure is designed as below. Please note that this is just a starting point of this framework and the directory structure will be subject to future changes.

```
├── ansible.cfg
├── dse_install.yml
├── dse_upgrade.yml
├── group_vars
│   └── all
├── hosts
└── roles
    ├── base_dse
    │   └── tasks
    │       └── main.yml
    ├── dse_install
    │   ├── meta
    │   │   └── main.yml
    │   └── tasks
    │       └── main.yml
    ├── dse_install_config
    │   ├── tasks
    │   │   └── main.yml
    │   └── vars
    │       └── main.yml
    ├── dse_start_svc
    │   └── tasks
    │       └── main.yml
    ├── dse_stop_svc
    │   └── tasks
    │       └── main.yml
    ├── dse_upgrade_binary
    │   ├── meta
    │   │   └── main.yml
    │   ├── tasks
    │   │   └── main.yml
    ├── dse_upgrade_bkup
    │   ├── tasks
    │   │   └── main.yml
    │   └── vars
    │       └── main.yml
    ├── dse_upgrade_mergconf
    │   ├── tasks
    │   │   └── main.yml
    ├── dse_upgrade_sstable
        ├── tasks
            └── main.yml

```

The table below gives the high level description of the top level elements in the framework.

| Top Level Item Name | Type   | Description  |
| ------------------- | ------ | ------------ |
| ansible.cfg         | File   | Configuration setting file to control Ansible execution behavior |
| dse_install.yml     | File   | Playbook for fresh DSE installation |
| dse_upgrade.yml     | File   | Playbook for upgrading DSE from an older version to a newer version |
| group_vars          | Folder | Global variables that are available to all playbooks |
| hosts               | File   | Ansible inventory file |
| roles               | Folder | Ansible roles (organization units) included in the playbooks | 

Among them, "ansible.cfg" is not DSE related. It is used in this framework to control the execution behavior of Ansible. For others, I will go through in more details in the following sections.

### 3.1. Define Inventory File (hosts)

The key point in creating Ansible inventory file for DSE installation and upgrade is to identify different groups (of hosts) that may have different execution orders, such as:  
* When installing a new DSE cluster, the seed nodes have to be started before non-seed nodes.
* When doing an DSE version upgrade for a multi-DC cluster, it is better to finish upgrading one DC before moving on to another DC.

For example, for a 1-DC DSE cluster, we could define the inventory file to be something like:

```
[dse:children]
dse_seed
dse_nonseed

[dse_seed]
seed_node1_ip
seed_node2_ip
...

[dse_nonseed]
nonseed_node1_ip
nonseed_node2_ip
...
```

### 3.2. Group Variables

Ansible uses the combination of ***hosts*** and ***group_vars*** to define group specific groups. Also, ***group_vars/all*** is used to specify the variables that are applicable to all managed hosts.

For this framework, there are no group specific variables and all variables are applicable to all hosts. Some examples of these variables are:
* The DataStax academy user email and password that are used to download DSE repository.
* The DSE version to be installed or upgraded.

### 3.3. Playbook and Roles 

The Ansible roles defines the execution modules that can be included (and shared) in playbooks. The roles defined in this framework are:

| Role name            | Description |
| -------------------- | ----------- |
| base_dse             | Common execution steps required for DSE installation and upgrade |
| dse_install          | DSE binary installation (Debian package) |
| dse_install_config   | Customized configuration setup for DSE installation |
| dse_start_svc        | Start DSE service and verify system log file |
| dse_stop_svc         | Stop DSE service and verify system log file |
| dse_upgrade_binary   | DSE binary version upgrade |
| dse_upgrade_bkup     | Configuration and data backup before DSE version upgrade |
| dse_upgrade_mergconf | Merge DSE configuration files after version upgrade |
| dse_upgrade_sstable  | SSTable upgrade |
