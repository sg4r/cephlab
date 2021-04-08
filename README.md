# Ceph-octopus Lab
Matériel pour commencer à jouer avec Ceph. Le lab virtuel Vagrant contient un environnement pour installer et configurer un cluster Ceph en version octopus en quelques étapes accompagné de commentaires pour vous guider à chaque étape. ces informations ne sont pas une procédure d'installation automatisée pour une utilisation en production. 
## Installation de vagrant et libvirt sur votre poste Ubuntu
```
sudo apt-get install virt-manager
sudo apt install vagrant
```
## Clone ceph-octopus lab
```
git clone https://github.com/sg4r/ceph-octopus.git
cd ceph-octopus
```
## Start lab
```
#start all Vms 
vagrant up
#status des Vms
vagrant status
```
## Préparation de la configuration des noeuds du cluster Ceph
```
#connexion au premier node
vagrant ssh cn1
#configuration de tous les nodes Ceph
[vagrant@cn1 ~]$ for i in {1..4}; do ssh root@cn$i dnf install -y podman; done
[vagrant@cn1 ~]$ for i in {1..4}; do ssh root@cn$i systemctl stop firewalld.service; done
[vagrant@cn1 ~]$ for i in {1..4}; do ssh root@cn$i systemctl disable firewalld.service; done
```
## Etapes
Les différentes étapes :
* [Création du cluster Ceph](cephcreate.md)
* [Stockage RBD](cephrbd.md)
* [Stockage CephFS](cephfs.md)
* [Stockage S3](cephs3.md)
* [Ceph Dashboard](cephdashboard.md)
* [Mise a jour](upgrade.md)

## Fin de partie
Pour supprimer les vms ou tout simplement faire le ménage avant de refaire une configuration , utiliser vagrant pour détruire toutes les vms
```
# suppressions de toutes les vms
vagrant destroy
# si vous voulez tout supprimer
cd ..
rm -fr ./ceph-octopus
```
## Evolutions
Les évolutions concernant ce lab sont :
- Pool Erasure code pour rgw
- Configuration HA des rgw S3
- Réplication multi-sites S3
- Convertir un cluster Ceph existant
- Exemple d'installation avec ceph-ansible
