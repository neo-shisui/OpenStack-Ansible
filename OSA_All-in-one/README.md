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

Extra storage: 500GB
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
