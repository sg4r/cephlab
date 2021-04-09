# ceph-ansible
Vous pouvez utiliser Ansible avec le playbook ceph-ansible pour installer un cluster Ceph sur du métal nu ou dans des conteneurs.
Pour cela, on va utiliser l'infrastructure vagrant définit par defaut, et cette fois-ci utiliser ansible pour faire l'installation à la place des commandes cephadm.

# Préparer l'environement
```
# si vous avez deja un environement vargrant 
vagrant destroy
# démarrer les vms
vagrant up
# connexion au premier node
vagrant ssh cn1
# préparation 
[vagrant@cn1 ~]$ sudo dnf -y install ansible
[vagrant@cn1 ~]$ sudo dnf -y install git
[vagrant@cn1 ~]$ git clone https://github.com/ceph/ceph-ansible
Cloning into 'ceph-ansible'...
remote: Enumerating objects: 275, done.
remote: Counting objects: 100% (275/275), done.
remote: Compressing objects: 100% (148/148), done.
remote: Total 58782 (delta 177), reused 176 (delta 118), pack-reused 58507
Receiving objects: 100% (58782/58782), 10.99 MiB | 297.00 KiB/s, done.
Resolving deltas: 100% (40855/40855), done.
[vagrant@cn1 ~]$ cd ceph-ansible/
[vagrant@cn1 ceph-ansible]$ 
# remarque: stable-5.0 Supports Ceph version octopus. This branch requires Ansible version 2.9.
[vagrant@cn1 ceph-ansible]$ git checkout stable-5.0
Branch 'stable-5.0' set up to track remote branch 'stable-5.0' from 'origin'.
Switched to a new branch 'stable-5.0'
# fix probleme
[vagrant@cn1 ceph-ansible]$ sudo dnf -y install python3-netaddr.noarch
# install requirements
[vagrant@cn1 ceph-ansible]$ pip3 install --user -r requirements.txt
```
# Procédure
```
[vagrant@cn1 ceph-ansible]$ cp site.yml.sample site.yml
[vagrant@cn1 ceph-ansible]$ cp group_vars/all.yml.sample group_vars/all.yml
[vagrant@cn1 ceph-ansible]$ cp group_vars/osds.yml.sample group_vars/osds.yml

vi group_vars/all.yml
ceph_release_num: 15
ntp_service_enabled: true
ntp_daemon_type: chronyd
ceph_origin: repository
ceph_repository: community
ceph_mirror: http://download.ceph.com
ceph_stable_key: https://download.ceph.com/keys/release.asc
ceph_stable_release: octopus
monitor_interface: eth1
ip_version: ipv4
public_network: "192.168.0.0/24"
radosgw_interface: eth1
dashboard_enabled: True
dashboard_protocol: http
dashboard_admin_user: admindh
dashboard_admin_password: demodhp@swd
grafana_admin_user: admingf
grafana_admin_password: demogfp@swd


vi group_vars/osds.yml
osd_auto_discovery: true
# lvm_volumes: []
osds_per_device: 1

vi hosts
# Ceph admin user for SSH and Sudo
[all:vars]
ansible_ssh_user=root
ansible_become=true
ansible_become_method=sudo
ansible_become_user=root

# Ceph Monitor Nodes
[mons]
cn1
cn2
cn3

# MDS Nodes
[mdss]
cn1
cn2
cn3

# RGW
[rgws]
cnrgw1
cnrgw2

# Manager Daemon Nodes
[mgrs]
cn1
cn2

# set OSD (Object Storage Daemon) Node
[osds]
cn1
cn2
cn3
cn4

# Grafana server
[grafana-server]
cn4

[vagrant@cn1 ceph-ansible]$ ansible all -m ping -i hosts
[vagrant@cn1 ceph-ansible]$ ansible-playbook site.yml -i hosts
....
INSTALLER STATUS *****************************************************************************************
Install Ceph Monitor           : Complete (0:00:58)
Install Ceph Manager           : Complete (0:00:44)
Install Ceph OSD               : Complete (0:01:05)
Install Ceph MDS               : Complete (0:00:37)
Install Ceph RGW               : Complete (0:00:17)
Install Ceph Dashboard         : Complete (0:00:42)
Install Ceph Grafana           : Complete (0:01:00)
Install Ceph Node Exporter     : Complete (0:02:43)

Friday 09 April 2021  06:54:16 +0000 (0:00:00.054)       0:11:33.937 ********** 
=============================================================================== 
ceph-container-engine : install container packages ---------------------------------------------- 122.51s
ceph-common : install redhat ceph packages ------------------------------------------------------ 117.01s
ceph-mon : waiting for the monitor(s) to form the quorum... -------------------------------------- 20.72s
install ceph-mgr packages on RedHat or SUSE ------------------------------------------------------ 13.88s
ceph-prometheus : start prometheus services ------------------------------------------------------ 13.24s
ceph-osd : wait for all osd to be up ------------------------------------------------------------- 11.05s
ceph-grafana : start the grafana-server service -------------------------------------------------- 10.66s
ceph-osd : use ceph-volume lvm batch to create bluestore osds ------------------------------------ 10.23s
ceph-dashboard : create dashboard admin user ------------------------------------------------------ 9.07s
ceph-node-exporter : start the node_exporter service ---------------------------------------------- 8.19s
install ceph-mds package on redhat or SUSE/openSUSE ----------------------------------------------- 7.30s
ceph-grafana : wait for grafana to start ---------------------------------------------------------- 6.30s
ceph-mgr : wait for all mgr to be up -------------------------------------------------------------- 6.13s
install ceph-grafana-dashboards package on RedHat or SUSE ----------------------------------------- 4.37s
gather and delegate facts ------------------------------------------------------------------------- 4.18s
ceph-osd : apply operating system tuning ---------------------------------------------------------- 3.90s
ceph-mds : create filesystem pools ---------------------------------------------------------------- 3.75s
ceph-common : install centos dependencies --------------------------------------------------------- 3.39s
ceph-infra : install firewalld python binding ----------------------------------------------------- 3.28s
ceph-prometheus : service handler ----------------------------------------------------------------- 3.27s
[vagrant@cn1 ceph-ansible]$
# remarque : 11 minutes pour installer l'infrastrucuture
# remarque : verification
[vagrant@cn1 ceph-ansible]$ sudo ceph -s
  cluster:
    id:     9570f268-206e-42dd-af86-03d54532f21c
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum cn1,cn2,cn3 (age 9m)
    mgr: cn2(active, since 105s), standbys: cn1
    mds: cephfs:1 {0=cn2=up:active} 2 up:standby
    osd: 8 osds: 8 up (since 7m), 8 in (since 7m)
    rgw: 2 daemons active (cnrgw1.rgw0, cnrgw2.rgw0)
 
  task status:
 
  data:
    pools:   7 pools, 169 pgs
    objects: 213 objects, 9.5 KiB
    usage:   8.2 GiB used, 352 GiB / 360 GiB availx²x²
    pgs:     169 active+clean
 
```
# Documentation
## ceph-ansible
Veuillez consulter la documentation hébergée ici: https://docs.ceph.com/projects/ceph-ansible/en/latest/
## readhat
https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/4/html/installation_guide/installing-red-hat-ceph-storage-using-ansible
