# Manage DataStax Enterprise (DSE) Installation and Version Upgrade Using Ansible Playbook

It is not a trivial task to provision a large DSE cluster, to manage the key configuration item (e.g. dse.yaml, cassandra.yaml, etc.) changes, and to do version upgrade.  Historically (pre DSE 5.0), there is really no out-of-the-box method to manage these tasks automatically (as much as possible). It is opertaionally a time-consuming and error-prone job to manually run these tasks on every DSE node instance, not even to mention that for many cases, you also have to consider the task execution sequence among different DES nodes (e.g. DC by DC, rack by rack, seed vs. no-seed and so on).

Starting from DSE 5.0, OpsCenter 6.0 (as part of DSE 5.0 release) introduces a new component called Life Cycle Manager (LCM) that is designed for efficient installation and configuration of a DSE cluster through a web-browser based GUI. It aims to solve the above issue as an easy-to-use, out-of-the-box solution offered by DataStax. Although there are some limitations with the current version (OpsCenter 6.0.8 as the time of this writing), it is in general a good solution for automatic DSE cluster provisioning and configuration management. For more detailed information about this product, please reference to LCM's documentation page (https://docs.datastax.com/en/latest-opscenter/opsc/LCM/opscLCM.html). 

Unfortunately, however, while the current version of LCM does a great job in provisioning new cluster and managing configuration changes, it does not support DSE version upgrade yet. It also has a limitation of provisioning a maximum number of  300 DSE nodes through the Web UI. 

Meanwhile, the way that LCM schecules the excecution of a job (provisioning new node or making configuration change) across the entire DSE cluster is in serial mode. This means that before finishing the execution of a job on one DSE node, LCM won't kick off the job on another node. Although correct, such a scheduling mechanism is definitely not flexible and less efficient. 

In order to overcome the limitations of the (current version of) LCM and bring some flexibility in job executiong scheduling across the entire DSE cluster, I created an [Ansible](http://docs.ansible.com/ansible/intro.html) framework (playbook) for:
* Installing a brand new DSE cluster
* Upgrading an existing DSE cluster to a newer version 

This framework does have some configuration management capability (e.g. to make changes in cassandra.yaml file), it is far from being a general tool to manage all DSE components related configuration files. It is planned for future improvment.
