## OpenStack-Ansible All-in-one
* References: [Link](https://docs.openstack.org/openstack-ansible/latest/user/aio/quickstart.html)

## Building an AIO
**Running an AIO build:**
* Prepare the host
* Bootstrap Ansible and the required roles
* Bootstrap the AIO configuration
* Run playbooks

My computer:
* Ubuntu 22.04 LTS
* CPU: 32GB

### 1. Prepare the host
```
; Update kernel
$ sudo apt-get update
$ sudo apt-get dist-upgrade
$ sudo reboot now
```

### 2. Bootstrap Ansible and the required roles
```
; Cloning the OpenStack-Ansible
$ sudo git clone https://opendev.org/openstack/openstack-ansible -b stable/2024.2 /opt/openstack-ansible
$ cd /opt/openstack-ansible
```

Bootstrap Ansible and the Ansible roles for the development environment.
```
; Run the following to bootstrap Ansible and the required roles:
$ sudo scripts/bootstrap-ansible.sh
```

### 3. Bootstrap the AIO configuration
The host must be prepared with the appropriate disks partitioning, packages, network configuration and configurations for the OpenStack Deployment.

Extra storage: 500GB (Optional)
``` 
; Get path to removable storage
$ lsblk
sda           8:0    0 465,8G  0 disk <-- Using this 
├─sda1        8:1    0 319,3G  0 part 
└─sda2        8:2    0 146,5G  0 part
$ export BOOTSTRAP_OPTS="bootstrap_host_data_disk_device=sda"
; For the default AIO scenario, the AIO configuration preparation is completed by executing:
$ sudo scripts/bootstrap-aio.sh
```

In order to implement any other services, add the name of the conf.d file name without the .yml.aio extension into the SCENARIO environment variable. Each key word should be delimited by an underscore. For example, the following will implement an AIO with barbican, cinder, glance, horizon, neutron, and nova. It will set the cinder storage back-end to ceph and will make use of LXC as the container back-end.
```
$ export SCENARIO='aio_lxc_barbican_ceph_lxb'
$ sudo scripts/bootstrap-aio.sh
```

To add any global overrides, over and above the defaults for the applicable scenario, edit `/etc/openstack_deploy/user_variables.yml`. In order to understand the various ways that you can override the default behaviour set out in the roles, playbook and group variables, see `Overriding default configuration`.

See the Deployment Guide for a more detailed break down of how to implement your own configuration rather than to use the AIO bootstrap.

### 4. Run playbooks

```
$ sudo openstack-ansible openstack.osa.setup_hosts
PLAY RECAP *************************************************************************************************************************************************************************************************
aio1                       : ok=218  changed=43   unreachable=0    failed=0    skipped=124  rescued=0    ignored=0   
aio1-cinder-api-container-73731bbf : ok=108  changed=54   unreachable=0    failed=0    skipped=36   rescued=0    ignored=0   
aio1-galera-container-f4b1d506 : ok=108  changed=53   unreachable=0    failed=0    skipped=36   rescued=0    ignored=0   
aio1-glance-container-4e235b91 : ok=108  changed=53   unreachable=0    failed=0    skipped=36   rescued=0    ignored=0   
aio1-horizon-container-b9f7323c : ok=108  changed=53   unreachable=0    failed=0    skipped=36   rescued=0    ignored=0   
aio1-keystone-container-95884f9e : ok=108  changed=53   unreachable=0    failed=0    skipped=36   rescued=0    ignored=0   
aio1-memcached-container-6582d3f8 : ok=108  changed=53   unreachable=0    failed=0    skipped=36   rescued=0    ignored=0   
aio1-neutron-ovn-northd-container-c4a07430 : ok=108  changed=53   unreachable=0    failed=0    skipped=36   rescued=0    ignored=0   
aio1-neutron-server-container-684ec5f1 : ok=108  changed=53   unreachable=0    failed=0    skipped=36   rescued=0    ignored=0   
aio1-nova-api-container-1d8dd40c : ok=108  changed=53   unreachable=0    failed=0    skipped=36   rescued=0    ignored=0   
aio1-placement-container-676fbe07 : ok=108  changed=53   unreachable=0    failed=0    skipped=36   rescued=0    ignored=0   
aio1-rabbit-mq-container-7def8e92 : ok=111  changed=53   unreachable=0    failed=0    skipped=36   rescued=0    ignored=0   
aio1-repo-container-4133d95d : ok=108  changed=53   unreachable=0    failed=0    skipped=36   rescued=0    ignored=0   
aio1-utility-container-67dd2a9c : ok=108  changed=53   unreachable=0    failed=0    skipped=36   rescued=0    ignored=0   
localhost                  : ok=22   changed=0    unreachable=0    failed=0    skipped=25   rescued=0    ignored=0   



EXIT NOTICE [Playbook execution success] **************************************
===============================================================================
$ sudo openstack-ansible openstack.osa.setup_infrastructure
PLAY RECAP *************************************************************************************************************************************************************************************************
aio1                       : ok=64   changed=10   unreachable=0    failed=0    skipped=54   rescued=0    ignored=0   
aio1-galera-container-f4b1d506 : ok=62   changed=1    unreachable=0    failed=0    skipped=32   rescued=0    ignored=0   
aio1-memcached-container-6582d3f8 : ok=14   changed=0    unreachable=0    failed=0    skipped=7    rescued=0    ignored=0   
aio1-rabbit-mq-container-7def8e92 : ok=51   changed=0    unreachable=0    failed=0    skipped=31   rescued=0    ignored=0   
aio1-repo-container-4133d95d : ok=62   changed=0    unreachable=0    failed=0    skipped=42   rescued=1    ignored=0   
aio1-utility-container-67dd2a9c : ok=54   changed=14   unreachable=0    failed=0    skipped=22   rescued=0    ignored=0   



EXIT NOTICE [Playbook execution success] **************************************
===============================================================================
$ sudo openstack-ansible openstack.osa.setup_openstack
```

## Interacting with an AIO
### 1. Using a GUI


### 2. Using a client or library

## Rebooting an AIO
The horizon web interface provides a graphical interface for interacting with the AIO deployment. By default, the horizon API is available on port 443 of the host (or port 80, if SSL certificate configuration was disabled). As such, to interact with horizon, simply browse to the IP of the host.