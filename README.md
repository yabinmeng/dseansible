# Manage DataStax Enterprise (DSE) Installation and Version Upgrade Using Ansible Playbook

It is not a trivial task to provision a large DSE cluster, to manage the key configuration item (e.g. dse.yaml, cassandra.yaml, etc.) changes, and to do version upgrade.  Historically (pre DSE 5.0), there is really no out-of-the-box method to manage these tasks automatically (as much as possible). It is operationally a time-consuming and error-prone work to manually run these tasks on every DSE node instance, not even to mention that for many cases, you also have to consider the task execution sequence among different DES nodes (e.g. DC by DC, rack by rack, seed vs. no-seed and so on).

Starting from DSE 5.0, OpsCenter 6.0 (as part of DSE 5.0 release) introduces a new component called Life Cycle Manager (LCM) that is designed for efficient installation and configuration of a DSE cluster through a web based GUI. It aims to solve the above issue as an easy-to-use, out-of-the-box solution offered by DataStax. Although there are some limitations with the current version (OpsCenter 6.0.8 as the time of this writing), it is in general a good solution for automatic DSE cluster provisioning and configuration management. For more detailed information about this product, please reference to LCM's documentation page (https://docs.datastax.com/en/latest-opscenter/opsc/LCM/opscLCM.html). 

  Unfortunately, while the current version of LCM does a great job in provisioning new cluster and managing configuration changes, it does not support DSE version upgrade yet. Meanwhile, the way that LCM schedules the execution of a job (provisioning new node or making configuration change) across the entire DSE cluster is in serial mode. This means that before finishing the execution of a job on one DSE node, LCM won't kick off the job on another node. Although correct, such a scheduling mechanism is not flexible and less efficient. 

In attempt to overcome the limitations of the (current version of) LCM and bring some flexibility in job execution scheduling across the entire DSE cluster, I created an Ansible framework (playbook) for:
* Installing a brand new DSE cluster
* Upgrading an existing DSE cluster to a newer version 

This framework does have some configuration management capabilities (e.g. to make changes in cassandra.yaml file), but it is not yet a general tool to manage all DSE components related configuration files. However, using Ansible makes this job much easier and quite straightforward. I will cover this 

## Ansible Introduction  

Unlike some other orchestration and configuration management (CM) tools like Puppet or Chef, Ansible is very lightweight. It doesn't require any CM related modeuls, daemons, or agents to be installed on the managed machines. It is based on Python and pushes YAML based script commands over SSH protocol to remote hosts for execution. Ansible only needs to be installed on one workstation machine (could be your laptop) and as long as the SSH access from the workstation to the managed machines is enabled, Ansible based orchestration and CM will work.

Ansible uses an ***inventory*** file to group the systems or host machines to be managed. Ansible commands (either through ad-hoc or in Ansible ***module***) are executed against the systems or host machines specified in the inventory file. For more advanced orchestration and/or CM scenario, Ansible ***playbook*** is always used. By using the original words from Ansible [documentation](http://docs.ansible.com/ansible/intro.html), the relationship among Ansible inventory, module, and playbook can be summarized as:

> *If Ansible modules are the tools in your workshop, playbooks are your instruction manuals, and your inventory of hosts are your raw material.*

## DSE Install/Upgrade Framework

This framework is based on Ansible playbook, following the Ansible playbook [best practice principles](http://docs.ansible.com/ansible/playbooks_best_practices.html). The playbook folder structure is listed as the tree structure below. 

```
├── ansible.cfg
├── dse_install.yml
├── dse_upgrade.yml
├── group_vars
│   └── all
├── hosts
└── roles
    ├── base_common
    │   └── tasks
    │       └── main.yml
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
    │   ├── tasks
    │   │   └── main.yml
    └── oracle_java
        ├── defaults
        │   └── main.yml
        ├── meta
        │   └── main.yml
        └── tasks
            └── main.yml
```

