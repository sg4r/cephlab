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
```
# Documentation
## ceph-ansible
Veuillez consulter la documentation hébergée ici: https://docs.ceph.com/projects/ceph-ansible/en/latest/
## readhat
https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/4/html/installation_guide/installing-red-hat-ceph-storage-using-ansible
