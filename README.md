# Cephlab
Ce lab virtuel Vagrant contient un environnement pour installer et configurer un cluster Ceph en version octopus en quelques étapes accompagné de commentaires pour vous guider à chaque étape. ces informations ne sont pas une procédure d'installation automatisée pour une utilisation en production. Vous trouverez les différentes étapes configurer les services RBD, CEPFS, S3, un Dashboard avec prometheus et grafana, ainsi que la mise en HA de 2 RGW. 
## Installation de vagrant pour un poste Linux
cephlab est supporté avec libvirt ou virtualbox. En fonction de votre environement de virtualisation il faudra installer vagrant puis virt-manager ou virtualbox et définir le bon provider dans le fichier Vagrantfile.  
Installation de vagrant :
```
# Pour debian/ubuntu
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install vagrant
# Pour redhat/oraclelinux/centos
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo yum -y install vagrant
```
Install virt-manager
```
# Pour debian/ubuntu
sudo apt-get install virt-manager
# Pour redhat/oraclelinux/centos
sudo yum install virt-manager
```
Installtion de virtualbox  
Pour l'installtion de virtualbox, se référer à la documentation https://www.virtualbox.org/

## Clone cephlab
```
git clone https://github.com/sg4r/cephlab.git
cd cephlab
```
## Définir votre provider pour Vagrant
Editer le fichier Vagrantfile et selectionner le provider situé en début du fichier en fonction de votre environement de virtualisation et commenter la ligne avec le provider non utilisé.

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
[vagrant@cn1 ~]$ for i in {1..4}; do ssh root@cn$i dnf install -y podman lvm2; done
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
* [Mirroring](ceph-mirror.md)

## Fin de partie
Pour supprimer les vms ou tout simplement faire le ménage avant de refaire une configuration , utiliser vagrant pour détruire toutes les vms
```
# suppressions de toutes les vms
vagrant destroy
# si vous voulez tout supprimer
cd ..
rm -fr ./cephlab
```
## Changements
- ajout de [rbd live migration](cephrbd.md#rbd-live-migration)
- début de [ceph-mirror](ceph-mirror.md)
## Evolutions
Les évolutions concernant ce lab sont :
- Pool Erasure code pour rgw
- Configuration HA des rgw S3
- Réplication multi-sites S3
- [Convertir un cluster Ceph existant](upgrade.nautilus2octopus.md)
- [Exemple d'installation avec ceph-ansible](cephansible.md)
