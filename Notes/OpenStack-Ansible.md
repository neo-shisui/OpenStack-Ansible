# Install OpenStack-Ansible

## Machine Install Info

* Ubuntu 24.04
* 200GB Disk SSD.
* 32GB RAM.

## I. Prepare the deployment host

[Link](https://docs.openstack.org/project-deploy-guide/openstack-ansible/latest/deploymenthost.html)

Cài đặt các phần mềm sau (nếu chưa có):

```
$ sudo apt install build-essential git chrony openssh-server python3-dev sudo
```

Đồng bộ thời gian bằng `ntp`:

```
$ sudo apt install ntp
; Kiểm tra ntp chạy bình thường
$ ntpq -p
```

Cấu hình SSH Keys: https://docs.ansible.com/ansible/latest/inventory_guide/connection_details.html

* Ansible sử dụng SSH public key authentication để kết nối từ `deployment host` đến `target hosts`.
* Passphrase for SSH key:
  * Không sử dụng: Tăng tính tự động, giảm các tương tác của người dùng khi chạy Ansible.
  * Sử dụng: Sử dụng `ssh-agent` và `ssh-add` để lưu tạm passphrase và chạy Ansible. 

```
; Generate SSH key (deployment host)
$ ssh-keygen -f ~/.ssh/ansible -t ed25519
; Copy public key lên máy target host. Format: user@hostname hoặc user@ip 
$ ssh-copy-id shisui@127.0.0.1
; Kiểm tra kết nối
$ ssh shisui@127.0.0.1
```

Cấu hình network:

* Ansible cần SSH từ máy deployment đến `các container` bằng SSH để triển khai.

* Mặc định mạng của container là `br-mgmt`. Chọn và cấu hình IP cho deployment host ==> Cấu hình sau.
  ```
  Container management: 172.29.236.0/22 (VLAN 10) 
  ```


Cài OpenStack-Ansible trên deployment host:

```
; Clone source
$ sudo git clone -b stable/2024.2 https://opendev.org/openstack/openstack-ansible /opt/openstack-ansible
; Run Ansible bootstrap script
$ cd /opt/openstack-ansible
$ sudo scripts/bootstrap-ansible.sh
... Chạy thành công
PLAY RECAP **************************************************************************************************
localhost             : ok=9    changed=3    unreachable=0    failed=0    skipped=5    rescued=0    ignored=0   

System is bootstrapped and ready for use.
```

* Vấn đề: Lỗi cài Galaxy Ansible (Do Ansible) ==> https://bugs.launchpad.net/openstack-ansible/+bug/1971606
* ==> Chạy lại script bootstrap.

## II. Prepare the target hosts

[Link](https://docs.openstack.org/project-deploy-guide/openstack-ansible/latest/targethosts.html)

**Cài các phần mềm cần thiết:**

```
$ sudo apt install bridge-utils debootstrap openssh-server tcpdump vlan python3
```

**Cấu hình SSH Key:**

```
; Copy public key lên máy target host. Format: user@hostname hoặc user@ip 
$ ssh-copy-id shisui@127.0.0.1
```

**Cấu hình Storage:**

* **LVM (Logical Volume Manager)**: Quản lý ổ đĩa, cho phép tạo các ổ đĩa logic trên một ổ đĩa vật lý.

  * **Block Storage (Cinder):** Có thể sử dụng LVM để lưu trữ dữ liệu.
  * **LXC Containers:** Có thể sử dụng LVM để lưu trữ hệ thống tệp của các container.

* Cấu hình LVM cho Cinder:

  * Tạo một nhóm volume LVM có tên `cinder-columes` trên máy chủ lưu trữ (storage host).
    ```
    ; Cài đặt LVM
    $ sudo apt install lvm2
    ; Xác định ổ đĩa để dùng LVM
    $ sudo lsblk
    sda           8:0    0 465,8G  0 disk 
    ├─sda1        8:1    0 319,3G  0 part /media/shisui/New Volume
    └─sda2        8:2    0 146,5G  0 part 
    ; Unmount /dev/sda2 (nếu đang mount)
    $ sudo umount /dev/sda2
    ; Tạo physical volume
    $ sudo pvcreate --metadatasize 2048 /dev/sda2
    WARNING: ext4 signature detected on /dev/sda2 at offset 1080. Wipe it? [y/n]: y
      Wiping ext4 signature on /dev/sda2.
      Physical volume "/dev/sda2" successfully created.
    $ sudo vgcreate cinder-volumes /dev/sda2
      Volume group "cinder-volumes" successfully created
    ```
    

* Tùy chọn, cấu hình LVM cho **lxc**:

  * *Create an LVM volume group named `lxc` for container file systems and set `lxc_container_backing_store: lvm` in user_variables.yml if you want to use LXC with LVM. If the `lxc` volume group does not exist, containers are automatically installed on the file system under `/var/lib/lxc` by default.*


**Cấu hình network:**

* Cấu hình các card mạng bridge sau để connect với các node trong container.

  | Bridge name | Best configured on    | With a static IP                    |
  | ----------- | --------------------- | ----------------------------------- |
  | br-mgmt     | On every node         | Always                              |
  | br-storage  | On every storage node | When component is deployed on metal |
  |             | On every compute node | Always                              |
  | br-vxlan    | On every network node | When component is deployed on metal |
  |             | On every compute node | Always                              |
  | br-vlan     | On every network node | Never                               |
  |             | On every compute node | Never                               |

* **lxcbr0:** LXC internal

  * Cho phép các container (lxc) giao tiếp với nhau.
  * OpenStack-Ansible tự động cấu hình interface này.
  * Attach vào `eth0` trong mỗi container.

* **br-mgmt:** Container management

  * Quản lý container.
  * Attach vào `eth1` trong mỗi container.

* **br-storage:**

  * Kết nối giữa OpenStack service và Block Storage devices.
  * Attach vào `eth2` trong container storage.

* **br-vxlan:** OpenStack Networking tunnel

  * Dùng cho mạng VXLAN.

* **br-vlan:** OpenStack Networking provider

  * Dùng cho mạng VLAN.

Cách 1: Dùng commandline.

```
$ sudo nmcli connection add type bridge con-name br-mgmt ifname br-mgmt
; Thêm interface vật lý vào bridge br-mgmt (ví dụ: wlp0s20f3 cho Wi-Fi)
;$ sudo nmcli connection add type ethernet con-name br-mgmt-slave ifname wlp0s20f3 master br-mgmt

$ sudo nmcli connection modify br-mgmt ipv4.method manual ipv4.addresses 172.29.236.10/22
$ sudo nmcli connection modify br-mgmt ipv4.gateway 172.29.236.1
$ sudo nmcli connection modify br-mgmt ipv4.dns "8.8.8.8 8.8.4.4"
$ sudo nmcli connection modify br-mgmt ipv4.method manual

; Up interface
$ sudo nmcli connection up br-mgmt
```

Cách 2: Cấu hình bằng file netplan: `/etc/netplan/01-netcfg.yaml`

Tham khảo: [Ansible OpenStack Networking](https://github.com/openstack/openstack-ansible/blob/stable/2024.2/etc/netplan/01-static.yml)

```yaml
network:
    version: 2
    ethernets:
        eno1:
            mtu: 9000
        eno2:
            mtu: 9000
    bonds:
        bond0:
            interfaces:
            - eno1
            - eno2
            mtu: 9000
            parameters:
                lacp-rate: fast
                mii-monitor-interval: 100
                mode: 802.3ad
                transmit-hash-policy: layer3+4
    vlans:
        bond0.10:
            id: 10
            link: bond0
        bond0.20:
            id: 20
            link: bond0
        bond0.30:
            id: 30
            link: bond0
        bond0.40:
            id: 40
            link: bond0
    bridges:
        br-mgmt:
            addresses:
            - 172.29.236.100/22   # Primary IP of br-mgmt
            - 172.29.236.101/22   # Internal LB VIP
            interfaces:
            - bond0.10
            mtu: 9000
            nameservers:
                addresses:
                - 8.8.8.8
                - 8.8.4.4
                search:
                - example.com
        br-storage:
            addresses:
            - 172.29.244.100/22  # Corrected to match cidr_networks
            interfaces:
            - bond0.20
            mtu: 9000
        br-vxlan:
            addresses:
            - 172.29.240.100/22
            interfaces:
            - bond0.30
            mtu: 9000
        br-ext:
            addresses:
            - 192.1.1.100/22
            interfaces:
            - bond0.40
            routes:
              - to: default
                via: 192.1.1.1
            #gateway4: 192.1.1.1
        br-vlan:
            interfaces: 
              - bond0
            mtu: 9000
        br-dbaas:
            addresses:
            - 172.29.252.100/22  # Added to match cidr_networks
            mtu: 9000
```

```
$ sudo netplan apply
$ sudo nmcli connection show
NAME                UUID                                  TYPE      DEVICE     
netplan-br-vlan     d7656275-8c26-36f5-8d48-60947be35b40  bridge    br-vlan    
Duong 5G            b71390bb-c14e-42b6-a9b9-b6f43031812f  wifi      wlp0s20f3  
netplan-br-ext      ccc2d2ce-c198-3f16-aec3-920b74017c24  bridge    br-ext     
netplan-br-mgmt     8cb6a060-3ec6-3b6b-93f0-ce345bf70e78  bridge    br-mgmt    
netplan-br-storage  d2c50d30-d214-3e5a-95d3-39fce462f768  bridge    br-storage 
netplan-br-vxlan    164001fc-1472-3ac4-ab1b-8a1eadcbfa74  bridge    br-vxlan   
netplan-bond0       ed99d67c-a858-3e46-8bab-6a5caa421e47  bond      bond0      
netplan-bond0.10    5f316ba2-f712-3dab-95c7-6f926fbd15f5  vlan      bond0.10   
netplan-bond0.20    11fe96d7-e0e1-32d7-8fee-5a5927478c8b  vlan      bond0.20   
netplan-bond0.30    0eb42139-82eb-387c-a597-711a86f3bfe1  vlan      bond0.30   
netplan-bond0.40    a7acbc26-d26c-31f6-abc3-a8cb23ad5866  vlan      bond0.40 
```

## III. Configure the deployment

[Link](https://docs.openstack.org/project-deploy-guide/openstack-ansible/latest/configure.html). Bao gồm:

* Cấu hình **target host networking**:  to define bridge interfaces and networks.
* A list of target hosts on which to install the software.
* Virtual and physical network relationships for OpenStack Networking (neutron).
* Passwords for all services.

**Cấu hình môi trường cài đặt:**

```
$ sudo cp -r /opt/openstack-ansible/etc/openstack_deploy/* /etc/openstack_deploy/
$ cd /etc/openstack_deploy/
$ sudo cp openstack_user_config.yml.example openstack_user_config.yml
```

Cấu hình file `/etc/openstack_deploy/openstack_user_config.yml`: [Tham khảo](https://github.com/openstack/openstack-ansible/blob/stable/2024.2/etc/openstack_deploy/openstack_user_config.yml.aio)

```yaml
---
cidr_networks:
  bmaas: 172.29.228.0/22
  lbaas: 172.29.232.0/22
  dbaas: 172.29.252.0/22
  management: 172.29.236.0/22
  tunnel: 172.29.240.0/22
  storage: 172.29.244.0/22

# For dbaas and lbaas network, most part of the network we should make available for
# neutron subnet as guest/amphora instances will be spawned within this network. We
# reserve network keeping in mind the following:
#   - 2x2.1 -> 10 : physical routers / SVI (or br-[l|d]baas if you route via the infra host)
#   - 2x2.11 -> 49 : available for container interface ip auto-allocation in LXC deploys
#   - 2x2.50 -? 2x5.254 : available for neutron allocation
used_ips:
  - "172.29.228.1,172.29.228.10"
  - "172.29.229.50,172.29.231.255"
  - "172.29.228.100"
  - "172.29.232.1,172.29.232.10"
  - "172.29.232.50,172.29.235.255"
  - "172.29.252.1,172.29.252.10"
  - "172.29.252.50,172.29.255.255"
  - "172.29.236.1,172.29.236.50"
  - "172.29.236.100"
  - "172.29.240.1,172.29.240.50"
  - "172.29.240.100"
  - "172.29.244.1,172.29.244.50"
  - "172.29.244.100"
  - "172.29.248.1,172.29.248.50"
  - "172.29.248.100"

global_overrides:
  internal_lb_vip_address: 172.29.236.101
  # The external IP is quoted simply to ensure that the .aio file can be used as input
  # dynamic inventory testing.
  external_lb_vip_address: "{{ bootstrap_host_public_address | default(ansible_facts['default_ipv4']['address']) }}"
  management_bridge: "br-mgmt"
  provider_networks:
    - network:
        container_bridge: "br-mgmt"
        container_type: "veth"
        container_interface: "eth1"
        ip_from_q: "management"
        type: "raw"
        group_binds:
          - all_containers
          - hosts
        is_management_address: true
    - network:
        container_bridge: "br-vxlan"
        container_type: "veth"
        container_interface: "eth10"
        ip_from_q: "tunnel"
        type: "vxlan"
        range: "1:1000"
        net_name: "vxlan"
        group_binds:
          - neutron_linuxbridge_agent
    - network:
        container_bridge: "br-dbaas"
        container_type: "veth"
        container_interface: "eth13"
        host_bind_override: "eth13"
        ip_from_q: "dbaas"
        type: "flat"
        net_name: "dbaas-mgmt"
        group_binds:
          - neutron_linuxbridge_agent
          - rabbitmq
    - network:
        container_bridge: "br-lbaas"
        container_type: "veth"
        container_interface: "eth14"
        host_bind_override: "eth14"
        ip_from_q: "lbaas"
        type: "flat"
        net_name: "lbaas"
        group_binds:
          - neutron_linuxbridge_agent
          - octavia-worker
          - octavia-housekeeping
          - octavia-health-manager
    - network:
        container_bridge: "br-bmaas"
        container_type: "veth"
        container_interface: "eth15"
        host_bind_override: "eth15"
        ip_from_q: "bmaas"
        type: "flat"
        net_name: "bmaas"
        group_binds:
          - ironic_inspector
    - network:
        container_bridge: "br-vlan"
        container_type: "veth"
        container_interface: "eth12"
        host_bind_override: "eth12"
        type: "flat"
        net_name: "physnet1"
        group_binds:
          - neutron_linuxbridge_agent
    - network:
        container_bridge: "br-vlan"
        container_type: "veth"
        container_interface: "eth11"
        type: "vlan"
        range: "101:200,301:400"
        net_name: "physnet2"
        group_binds:
          - neutron_linuxbridge_agent
    - network:
        container_bridge: "br-storage"
        container_type: "veth"
        container_interface: "eth2"
        ip_from_q: "storage"
        type: "raw"
        group_binds:
          - glance_api
          - cinder_api
          - cinder_volume
          - manila_share
          - nova_compute
          - swift_proxy

# galera, memcache, rabbitmq, utility
shared-infra_hosts:
  shisui:
    ip: 172.29.236.100

repo-infra_hosts:
  shisui:
    ip: 172.29.236.100

load_balancer_hosts:
  shisui:
    ip: 172.29.236.100
```

Cấu hình file `/etc/openstack_deploy/user_variables.yml`: 

* https://github.com/neo-shisui/OpenStack-Ansible/blob/main/etc/openstack_deploy/user_variables.yml

**Cài đặt các service khác:**

* Cách để cài thêm service: 

  * Tạo file config trong folder: `/etc/openstack_deploy/conf.d/`.

* HaProxy service: `/etc/openstack_deploy/conf.d/haproxy.yml`
  ```yaml
  ---
  load_balancer_hosts:
    shisui:
      ip: 172.29.236.10
  ```

**Cấu hình service nâng cao:** Cập nhật sau.

**Cấu hình service credential:**

* Cấu hình credential cho các service trong file `/etc/openstack_deploy/user_secrets.yml`.

* `keystone_auth_admin_password` là password của user `admin` cho OpenStack API và Dashboard.

* Sử dụng script để generate credential cho các service:
  ```
  $ cd /opt/openstack-ansible
  $ sudo ./scripts/pw-token-gen.py --file /etc/openstack_deploy/user_secrets.yml
  Creating backup file [ /etc/openstack_deploy/user_secrets.yml.tar ]
  Operation Complete, [ /etc/openstack_deploy/user_secrets.yml ] is ready
  ```

## IV. Run playbooks

Chạy 3 playbook:

* The `openstack.osa.setup_hosts` Ansible foundation playbook prepares the target hosts for infrastructure and OpenStack services, builds and restarts containers on target hosts, and installs common components into containers on target hosts.
* The `openstack.osa.setup_infrastructure` Ansible infrastructure playbook installs infrastructure services: Memcached, the repository server, Galera and RabbitMQ.
* The `openstack.osa.setup_openstack` OpenStack playbook installs OpenStack services, including Identity (keystone), Image (glance), Block Storage (cinder), Compute (nova), Networking (neutron), etc.

### Chạy playbook để cài OpenStack

**Chạy host setup playbook:**

```
$ sudo openstack-ansible openstack.osa.setup_hosts
PLAY RECAP **************************************************************************************************
localhost                  : ok=22   changed=0    unreachable=0    failed=0    skipped=25   rescued=0    ignored=0   
shisui                     : ok=179  changed=2    unreachable=0    failed=0    skipped=114  rescued=0    ignored=0   
shisui-galera-container-202ccd05 : ok=96   changed=5    unreachable=0    failed=0    skipped=35   rescued=0    ignored=0   
shisui-memcached-container-2ff984ea : ok=93   changed=5    unreachable=0    failed=0    skipped=35   rescued=0    ignored=0   
shisui-rabbit-mq-container-dbb60554 : ok=99   changed=28   unreachable=0    failed=0    skipped=33   rescued=0    ignored=0   
shisui-repo-container-09b29ca1 : ok=93   changed=5    unreachable=0    failed=0    skipped=35   rescued=0    ignored=0   
shisui-utility-container-7da7102e : ok=93   changed=5    unreachable=0    failed=0    skipped=35   rescued=0    ignored=0   



EXIT NOTICE [Playbook execution success] **************************************
===============================================================================
```

Debug lỗi chạy container `shisui-rabbit-mq-container-dbb60554`:

```
; Check status của container
$ sudo lxc-info -n shisui-rabbit-mq-container-dbb60554 
State: STOPPED
; Thử chạy lại container và in lỗi (nếu có)
$ sudo lxc-start -n shisui-rabbit-mq-container-dbb60554 -v
lxc-start: shisui-rabbit-mq-container-dbb60554: network.c: netdev_configure_server_veth: 711 No such file or directory - Failed to attach "dbb60554_eth13" to bridge "br-dbaas", bridge interface doesn't exist
-> Chưa có interface "br-dbaas"
-> Update file /etc/netplan/01-netcfg.yaml
```

**Chạy infrastructure setup playbook:**

```
$ sudo openstack-ansible openstack.osa.setup_infrastructure
PLAY RECAP *********************************************************************
localhost                  : ok=22   changed=0    unreachable=0    failed=0    skipped=25   rescued=0    ignored=0   
shisui                     : ok=177  changed=2    unreachable=0    failed=0    skipped=116  rescued=0    ignored=0   
shisui-galera-container-3ef45882 : ok=93   changed=7    unreachable=0    failed=0    skipped=35   rescued=0    ignored=0   
shisui-memcached-container-87605c08 : ok=93   changed=6    unreachable=0    failed=0    skipped=35   rescued=0    ignored=0   
shisui-rabbit-mq-container-cb81a39e : ok=93   changed=6    unreachable=0    failed=0    skipped=35   rescued=0    ignored=0   
shisui-repo-container-7f0c3f7c : ok=93   changed=6    unreachable=0    failed=0    skipped=35   rescued=0    ignored=0   
shisui-utility-container-38cb37ec : ok=96   changed=6    unreachable=0    failed=0    skipped=35   rescued=0    ignored=0   



EXIT NOTICE [Playbook execution success] **************************************
===============================================================================
```

**Kiểm tra database cluster:**

```
$ ansible galera_container -m shell  -a "mysql -h localhost -e 'show status like \"%wsrep_cluster_%\";'"
```



## V. Verify OpenStack operation



## References

[OpenStack-Ansible Deployment Guide](https://docs.openstack.org/project-deploy-guide/openstack-ansible/latest/overview.html)

[OpenStack-Ansible Architecture](https://docs.openstack.org/openstack-ansible/latest/reference/architecture/index.html)

[]