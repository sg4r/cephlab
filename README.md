# Ceph-octopus Labs
Matériel pour commencer à jouer avec Ceph. Ce labs virtuel Vagrant contient un environnement pour installer et configurer un cluster Ceph en version octopus en quelque étapes accopagné de commentaires pour vous guider à chaque étape. c'est informations ne pas une procédure d'installation automatisée pour une utilisation en production. 
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
guides pour les différentes étapes :
* [Création du cluster Ceph](cephcreate.md)
* [Stockage RBD](cephrbd.md)
* [Stockage CephFS](cephfs.md)
* Stockage S3

## Fin de partie
Pour surrimer les vms ou simplement faire le ménage avant de refaire une partie , utiliser vagrant pour détruire toutes les vms
```
# supressions de toutes les vms
vargrant destroy
# si vous voulez tout supprimer
cd ..
rm -fr ./ceph-octopus
```
