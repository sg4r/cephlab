# Ceph mirroring
Un cluster Ceph distribue sont stockage sur différents serveurs en mode synchrone limitant ainsi les distances entre chaque serveur. C'est pourquoi la mise en place d'un cluster étendu sur de longues distances n'est pas une bonne idée car cela augmente les latences et dégrade les performances.
Pour garantir une disponibilité des données, il est possible de faire de la réplication asynchrone entre deux clusters Ceph distant de plusieurs centaines de kilomètres.  

**Avantage :**  
- _Plan de reprise d’activité_ : un site de secours peut reprendre l'activité rapidement. Le site dispose d'une copie des données et il n'est pas nécessaire d'avoir la même infrastructure ou la même version de Ceph au niveau des 2 clusters. Chaque cluster peut évoluer à son rythme.  
- _Haute disponibilité_ : Avec 2 clusters Ceph distant, il est possible que chacun réplique les données et ce qui permet des clients par zones.  

Il y a un type de réplication en fonction du type de stockage :  

**[rbd-mirroring](https://docs.ceph.com/en/latest/rbd/rbd-mirroring/)** est disponible en deux modes :
Mode basé sur un _journal_ : Ce mode utilise la fonctionnalité de journalisation de l'image RBD afin de garantir une réplication. Les modifications sont appliquées en continue par relecture du journal sur le site distant. Il est possible de choisir de répliquer tout un pool ou une série d’image rbd d’un pool.  
Mode basé sur _snapschot_ : Ce mode utilise des snapshots d'image RBD planifiés régulièrement pour répliquer des images RBD. En fonction de l’intervalle, les modifications ne sont pas présentées sur le site secondaire s’il y a un désastre sur le site primaire.  

Selon les besoins de réplication, la mise en miroir RBD peut être configurée pour une réplication unidirectionnelle ou bidirectionnelle :  
Réplication unidirectionnelle : Lorsque les données sont mises en miroir uniquement à partir du cluster primaire vers le cluster secondaire  
Réplication bidirectionnelle : Lorsque les données sont mises en miroir à partir des images primaires d’un cluster vers le cluster secondaire et inversement. 
Il n’est possible que de réaliser une réplication par sélection d’une série image RBD depuis un ou plusieurs pools.

**[cephfs-mirroring](https://docs.ceph.com/en/latest/dev/cephfs-mirroring/)** : réplication des données CephFS. Disponible depuis la version Ceph Pacific.   
 Fonctionne seulement vers un site de secours avec la possibilité de faire un plan de reprise d'activité. Il possible de sélectionner différents répertoires avec une définition de différents règles de snapshots.

**[radosgw-multisites](https://docs.ceph.com/en/latest/radosgw/multisite-sync-policy/)** : à compléter.

# Environement 
Pour tester ces fonctions, il est nécessaire de disposer de 2 cluster Ceph.  
Le fichier vagrant « cephmirror » est disponible dans le dossier le dossier. Avant de l’utiliser détruisez votre environnement actuel avec la commande vagrant halt et vagrant destoy. Puis remplacer le fichier Vagrant avec l'environement cephenvmirror qui contient la définition de 2 clusters.  
Le cluster A avec les nodes cna1,2,3 et le cluster B avec les nodes cnb1,2,3  et un serveur cephclt  qui serra le client pour accéder à ces 2 cluster Ceph A,B.

```
# arrêt et suppression de l'environement vfcephenvstart
vagrant halt
vagrant destroy
# remplacement de l'environement
mv Vagrantfile vfcephenvstart
mv vfcephenvmirror Vagrantfile
```

## Création des 2 clusters A et B
```
# connexion au premier cluster A
vagrant ssh cna1
# configuration de tous les nodes Ceph
[vagrant@cna1 ~]$ for i in {1..3}; do ssh root@cna$i dnf install -y podman lvm2; done
[vagrant@cna1 ~]$ for i in {1..3}; do ssh root@cna$i systemctl stop firewalld.service; done
[vagrant@cna1 ~]$ for i in {1..3}; do ssh root@cna$i systemctl disable firewalld.service; done

# connexion au deuxième cluster B
vagrant ssh cnb1
# configuration de tous les nodes Ceph
[vagrant@cnb1 ~]$ for i in {1..3}; do ssh root@cnb$i dnf install -y podman lvm2; done
[vagrant@cnb1 ~]$ for i in {1..3}; do ssh root@cnb$i systemctl stop firewalld.service; done
[vagrant@cnb1 ~]$ for i in {1..3}; do ssh root@cnb$i systemctl disable firewalld.service; done

# connexion a cephclt
vagrant ssh cephclt
```
