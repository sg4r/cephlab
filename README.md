# Ceph-octopus Labs
Matériel pour commencer à jouer avec Ceph. Ce labs virtuelle Vagrant contient un environnement pour faire l'installation de Ceph octopus en quelque étapes. 
## Installation de vagrant et libvirt sur votre poste Ubuntu
```
sudo apt-get install virt-manager
sudo apt install vagrant
```
## Clone ceph-octopus labs
```
git clone https://github.com/sg4r/ceph-octopus.git
cd ceph-octopus
```
## Start labs
```
#start all Vms 
vagrant up
#status des Vms
vagrant status
```
## Préparation de la configuration des nodes du cluster Ceph.
```
#connexion au premier node
vagrant ssh cn1
#configuration de tous les nodes Ceph
[vagrant@cn1 ~]$ for i in {1..4}; do ssh root@cn$i dnf install -y podman; done
[vagrant@cn1 ~]$ for i in {1..4}; do ssh root@cn$i systemctl stop firewalld.service; done
[vagrant@cn1 ~]$ for i in {1..4}; do ssh root@cn$i systemctl disable firewalld.service; done
```
## Etapes
différentes étapes
* [Création du cluster Ceph](cephcreate.md)
* Stockage RBD
* Stockage CephFS
* Stockage S3
