# Manage DataStax Enterprise (DSE) Installation and Version Upgrade Using Ansible Playbook

## 1. Overview

It is not a trivial task to provision a large DSE cluster, to manage the key configuration item changes (e.g. dse.yaml, cassandra.yaml, etc.), and to do version upgrade.  Historically (pre DSE 5.0), there is really no out-of-the-box method to manage these tasks automatically. It is operationally a time-consuming and error-prone work to manually run these tasks on every DSE node instance, not even to mention that for many cases, you also have to consider the task execution sequence among different DES nodes (e.g. DC by DC, rack by rack, seed vs. no-seed and so on).

Starting from DSE 5.0, OpsCenter 6.0 (as part of DSE 5.0 release) introduces a new component called Life Cycle Manager (LCM) that is designed for efficient installation and configuration of a DSE cluster through a web based GUI. It aims to solve the above issue as an easy-to-use, out-of-the-box solution offered by DataStax. Although there are some limitations with the current version (OpsCenter 6.0.8 as the time of this writing), it is in general a good solution for automatic DSE cluster provisioning and configuration management. For more detailed information about this product, please reference to LCM's documentation page (https://docs.datastax.com/en/latest-opscenter/opsc/LCM/opscLCM.html). 

Unfortunately, while the current version of LCM does a great job in provisioning new cluster and managing configuration changes, it does not support DSE version upgrade yet. Meanwhile, the way that LCM schedules the execution of a job (provisioning new node or making configuration change) across the entire DSE cluster is in serial mode. This means that before finishing the execution of a job on one DSE node, LCM won't kick off the job on another node. Although correct, such a scheduling mechanism is not very flexible and a little bit less efficient. 

In an attempt to overcome the limitations of the (current version of) LCM and bring some flexibility in job execution scheduling across the entire DSE cluster, an Ansible framework (as presented in this document) is created for:
* Installing a brand new DSE cluster
* Upgrading an existing DSE cluster to a newer version 

**Note 1**: The framework is currently based on Debian packaged DSE installation or upgrade.

**Note 2**: This framework does have some configuration management capabilities (e.g. to make changes in cassandra.yaml file), but it is not yet a general tool to manage all DSE components related configuration files. However, using Ansible makes this job much easier and quite straightforward. 

## 2. Ansible Introduction  

Unlike some other orchestration and configuration management (CM) tools like Puppet or Chef, Ansible is very lightweight. It doesn't require any CM related modeuls, daemons, or agents to be installed on the managed machines. It is based on Python and pushes YAML based script commands over SSH protocol to remote hosts for execution. Ansible only needs to be installed on one workstation machine (could be your laptop) and as long as the SSH access from the workstation to the managed machines is enabled, Ansible based orchestration and CM will work.

Ansible uses an ***inventory*** file to group the host machines to be managed. Ansible commands (either through ad-hoc or in Ansible ***module***) are executed against the systems or host machines specified in the inventory file. For more advanced orchestration and/or CM scenario, Ansible ***playbook*** is always used. By using the original words from Ansible [documentation](http://docs.ansible.com/ansible/intro.html), the relationship among Ansible inventory, module, and playbook can be summarized as:

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
    ├── datastax_pkg
    │   ├── tasks
    │   │   └── main.yml
    │   └── vars
    │       └── main.yml
    ├── dse_common
    │   ├── tasks
    │   │   └── main.yml
    │   └── templates
    │       └── seeds.j2
    ├── dse_instbin
    │   └── tasks
    │       └── main.yml
    ├── dse_updcfg
    │   └── tasks
    │       └── main.yml
    ├── dse_upgbin
    │   └── tasks
    │       └── main.yml
    ├── start_srvc
    │   └── tasks
    │       └── main.yaml
    └── stop_srvc
        └── tasks
            └── main.yaml
```

The table below gives the high level description of the top level elements in the framework.

| Top Level Item Name | Type   | Description  |
| ------------------- | ------ | ------------ |
| ansible.cfg         | File   | Configuration setting file to control Ansible execution behavior |
| dse_install.yml     | File   | Playbook for brand new DSE installation |
| dse_upgrade.yml     | File   | Playbook for upgrading DSE from an older version to a newer version |
| group_vars          | Folder | Global variables that are available to all playbooks |
| hosts               | File   | Ansible inventory file |
| roles               | Folder | Ansible roles (organization units) included in the playbooks | 


### 3.1. Define Inventory File (hosts)

The managed hosts can be put in different groups. Each managed host and the group it belongs to can have their own properties (or variables). Based on this understanding, the hosts file in this framework is organized such that:
1. the managed hosts into DSE data-centers (DCs). 
2. each DC defines the following attributes:
   1. whether Solr workload isenabled in the DC
   2. whether Spark workload is enabled in the DC
   3. whether Graph workload is enabled in the DC (only applicable to DSE version 5.0+)
   4. whether the DC should be "auto_bootstrap"-ped (e.g. whehter the DC is to join an existing cluster)
3. each managed host defines the following attributes:
   1. the private IP, if any, of the host
   2. whether the host is a seed node
   3. the DSE DC name that the host belongs to
   4. the rack name that the host belongs to
   5. whether the host is using VNode. if true (1), "intial_token" property (below) must be empty
   6. what is the initial_token value. if specified, the "vnode" property must be 0

Below is an exmple for a 2-DC DSE cluster. Eac DC has 2 nodes. The 1st DC (DC1) is VNode enabled and has only Cassandra workload. The 2nd DC (DC2) is using single-token setup and also has Solr workload enabled. 

```
[dse:children]
dse_dc1
dse_dc2

[dse_dc1]
<node1_public_ip> private_ip=<node1_private_ip> seed=true dc=DC1 rack=RAC1 vnode=1 initial_token=
<node2_public_ip> private_ip=<node2_private_ip> seed=false dc=DC1 rack=RAC1 vnode=1 initial_token=

[dse_dc2]
<node3_public_ip> private_ip=<node3_private_ip> seed=true dc=DC2 rack=RAC1 vnode=0 initial_token=-9223372036854775808
<node4_public_ip> private_ip=<node4_private_ip> seed=false dc=DC2 rack=RAC1 vnode=0 initial_token=0

[dse_dc1:vars]
solr_enabled=0
spark_enabled=0
graph_enabled=0
auto_bootstrap=1

[dse_dc2:vars]
solr_enabled=1
spark_enabled=0
graph_enabled=0
auto_bootstrap=1
...
```

### 3.2. Group Variables

Other than the host-specific variables as defined in the inventory file (***hosts***). Group and global variables can also be put in ***group_vars*** sub-folder. In particular, ***group_vars/all*** file is used to specify the variables that are applicable to all managed hosts, no matter what groups the hosts belong to. 

Within this framework, the following global variables are defined:

| Global Varialbe       | Default Value | Description   |
| --------------------- | ------------- | ------------- |
| ansible_connection    | ssh | what connection method is used for ansible |
| ansible_user          | N/A | the SSH user used to connect to the managed hosts |
| backup_homedir        | N/A | the home diretory for backing up DSE configuration files |
| dse_ver_target        | N/A | the target DSE version to be installed or upgraded to |
| dse_config_dir        | /etc/dse | the home directory of DSE configuraion files |
| dse_default_dir       | /etc/default | the home directory of DSE default setting file |
| vnode_token_num       | 128 | num_token value setting for VNode setup | 
| vnode_token_num_solr  | 8 | num_token value setting for VNode setup when Solr workload is enabled |
| cluster_name          | N/A | DSE cluster name |
| data_file_directories | N/A | Cassandra data directory (note: JBOD is NOT supported yet) |
| commitlog_directory   | N/A | Cassandra commitlog directory |
| saved_caches_directory| N/A | Cassandra saved_caches directory |
| hints_directory       | N/A | Cassandra hints directory (note: only applicable to DSE 5.0+ |


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
