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
[vagrant@cna1 ~]$ sudo yum -y install https://download.ceph.com/rpm-16.2.7/el8/noarch/cephadm-16.2.7-0.el8.noarch.rpm
[vagrant@cna1 ~]$ sudo cephadm bootstrap --mon-ip 192.168.111.11 --skip-monitoring-stack
 	     URL: https://cna1:8443/
	     User: admin
	Password: 039ovs9fjy => passsitea
 #noter le pass par defaut pour le changer ensuite en passsitea

[vagrant@cna1 ~]$ for i in {2..3}; do ssh-copy-id -f -i /etc/ceph/ceph.pub root@cna$i; done
[vagrant@cna1 ~]$ for i in {2..3}; do ssh root@cna$i dnf -y install https://download.ceph.com/rpm-16.2.7/el8/noarch/cephadm-16.2.7-0.el8.noarch.rpm; done
[vagrant@cna1 ~]$ sudo cephadm shell
#depuis le shell du conteneur 
ceph orch host add cna2
ceph orch host add cna3
ceph orch host ls
ceph orch ls
ceph orch device ls
#attendre que les nodes soient dans le cluster
ceph orch apply osd --all-available-devices
ceph osd tree


# connexion au deuxième cluster B
vagrant ssh cnb1
# configuration de tous les nodes Ceph
[vagrant@cnb1 ~]$ for i in {1..3}; do ssh root@cnb$i dnf install -y podman lvm2; done
[vagrant@cnb1 ~]$ for i in {1..3}; do ssh root@cnb$i systemctl stop firewalld.service; done
[vagrant@cnb1 ~]$ for i in {1..3}; do ssh root@cnb$i systemctl disable firewalld.service; done
[vagrant@cnb1 ~]$ sudo yum -y install https://download.ceph.com/rpm-16.2.7/el8/noarch/cephadm-16.2.7-0.el8.noarch.rpm
[vagrant@cnb1 ~]$ sudo cephadm bootstrap --mon-ip 192.168.111.20 --skip-monitoring-stack
 	     URL: https://cnb1:8443/
	     User: admin
	Password: 2f4dq012jk  => passsiteb
 #noter le pass par defaut pour le changer ensuite en passsiteb

[vagrant@cnb1 ~]$ for i in {2..3}; do ssh-copy-id -f -i /etc/ceph/ceph.pub root@cnb$i; done
[vagrant@cnb1 ~]$ for i in {2..3}; do ssh root@cna$i dnf -y install https://download.ceph.com/rpm-16.2.7/el8/noarch/cephadm-16.2.7-0.el8.noarch.rpm; done
[vagrant@cnb1 ~]$ sudo cephadm shell
#depuis le shell du conteneur 
ceph orch host add cnb2
ceph orch host add cnb3
ceph orch host ls
ceph orch ls
ceph orch device ls
#attendre que les nodes soient dans le cluster
ceph orch apply osd --all-available-devices
ceph osd tree


# connexion a cephclt
vagrant ssh cephclt
```

# CephFS-mirroring
TP pour la configuration de CephFS-mirroring.  
Attention les opérations sont a faire soit sur cluster A ou cluster B ou cephclt. soyez vigilant ;)

```
[root@cephclt ~]# yum install -y centos-release-ceph-pacific.noarch
[root@cephclt ~]# yum install -y ceph-common

[root@cephclt ~]# mkdir /prod/
[root@cephclt ~]# mkdir /back/

[ceph: root@cna1 /]# grep key /etc/ceph/ceph.keyring 
	key = AQCV3WNi1L5XFhAAvjg10OgvYJOcFHpVkdey0w==
[root@cephclt ~]# echo AQCV3WNi1L5XFhAAvjg10OgvYJOcFHpVkdey0w== >/etc/ceph/sitea.cephfs
[root@cephclt ~]# mount -t ceph cna1,cna2,cna3,cna4:/ /prod -o name=admin,secretfile=/etc/ceph/sitea.cephfs

[root@cephclt ~]# mkdir /prod/rep1
[root@cephclt ~]# mkdir /prod/rep2
[root@cephclt ~]# echo file1 >/prod/rep1/file1
[root@cephclt ~]# echo file2 >/prod/rep2/file2

[ceph: root@cnb1 /]# grep key /etc/ceph/ceph.keyring 
	key = AQBL4mNiG0fJGhAAUrzMZdzm1zlha/4ypSg0Mg==
[root@cephclt ~]# echo AQBL4mNiG0fJGhAAUrzMZdzm1zlha/4ypSg0Mg== >/etc/ceph/siteb.cephfs
[root@cephclt ~]# mount -t ceph cnb1,cnb2,cnb3,cnb4:/ /back/ -o name=admin,secretfile=/etc/ceph/siteb.cephfs

# vérification si deja des fichiers ?
[root@cephclt ~]# grep -R . /back
# activation du mirror de rep1
[ceph: root@cna1 /]#  ceph fs snapshot mirror add cephfs /rep1
{}
[root@cephclt ~]# grep -R . /back
# pas encore de mirror
[root@cephclt ~]# mkdir /prod/rep1/.snap/s1
[root@cephclt ~]# grep -R . /back
/back/rep1/file1:file1
# ok pour rep1 il faut faire un snap pour que la copie se réalise.
#activation de rep2
[ceph: root@cna1 /]#  ceph fs snapshot mirror add cephfs /rep2
{}
[root@cephclt ~]# grep -R . /back
/back/rep1/file1:file1
[root@cephclt ~]# mkdir /prod/rep2/.snap/s1
[root@cephclt ~]# grep -R . /back
/back/rep2/file2:file2
/back/rep1/file1:file1
[root@cephclt ~]# ls  /back/rep1/.snap/
s1
# ok le mirror snap fait sont travail

# utilisation du module pour la gestion automatique des snaps sous CephFS
[ceph: root@cna1 /]# ceph fs set cephfs allow_new_snaps true
enabled new snapshots
[ceph: root@cna1 /]# ceph mgr module enable snap_schedule
[ceph: root@cna1 /]# ceph fs snap-schedule add /rep1 1h
Schedule set for path /rep1
[ceph: root@cna1 /]# ceph fs snap-schedule retention add /rep1 h 24 
Retention added to path /rep1
[ceph: root@cna1 /]# ceph fs snap-schedule add /rep2 2h 
Schedule set for path /rep2
[ceph: root@cna1 /]# ceph fs snap-schedule retention add /rep2 h 24 
Retention added to path /rep2
[ceph: root@cna1 /]# ceph fs snap-schedule list / --recursive=true
/rep1 1h 24h
/rep2 2h 24h

# par defaut, utiliser le module snap_schedule pour créer les snaps sur chaque branche.
# il n'est pas possible de gérer une prédiode inférieur à l'heure

[root@cephclt ~]# echo siterpod >/prod/rep1/prod.txt
[root@cephclt ~]# mkdir /prod/rep1/.snap/s2
[root@cephclt ~]# grep -R . /back
/back/rep2/file2:file2
/back/rep1/file1:file1
/back/rep1/prod.txt:siterpod

# Cluster A est en production. tout va bien ;) siteb recois les copies des fichiers.
# On va faire un basculement vers le cluster B (siteb) 

[root@cephclt ~]# umount /prod
[root@cephclt ~]# umount /back 

[ceph: root@cna1 /]# ceph fs snapshot mirror disable cephfs
{}
[ceph: root@cna1 /]#  ceph fs snapshot mirror peer_list cephfs
Error EINVAL: filesystem cephfs is not mirrored

[root@cephclt ~]# mount -t ceph cnb1,cnb2,cnb3,cnb4:/ /prod -o name=admin,secretfile=/etc/ceph/siteb.cephfs
[root@cephclt ~]# mount -t ceph cna1,cna2,cna3,cna4:/ /back -o name=admin,secretfile=/etc/ceph/sitea.cephfs

[root@cephclt ~]# echo "basculement siteb prod" >>/prod/rep1/prod.txt 
[root@cephclt ~]# grep -R . /prod/
/prod/rep2/file2:file2
/prod/rep1/file1:file1
/prod/rep1/prod.txt:siterpod
/prod/rep1/prod.txt:basculement siteb prod

# ok cluster B (siteb) est en production.
# cluster A est de nouveau disponible et retrouve sont Cephfs, mais ne recois pas les snaps.

# le retour, visiblement il n'est pas possible d'utiliser cephFs-mirror, donc on peux faire cela avec rsync
[root@cephclt ~]# mkdir /prod/rep1/.snap/sitebprod
[root@cephclt ~]# rsync -arv /prod/ /back
sending incremental file list
./
rep1/
rep1/prod.txt
rep2/

sent 249 bytes  received 50 bytes  598.00 bytes/sec
total size is 44  speedup is 0.15

[root@cephclt ~]# umount /prod
[root@cephclt ~]# umount /back 

[root@cephclt ~]# mount -t ceph cna1,cna2,cna3,cna4:/ /prod -o name=admin,secretfile=/etc/ceph/sitea.cephfs
[root@cephclt ~]# mount -t ceph cnb1,cnb2,cnb3,cnb4:/ /back/ -o name=admin,secretfile=/etc/ceph/siteb.cephfs
[root@cephclt ~]# echo "reprise prod sitea" >>/prod/rep1/prod.txt 
[root@cephclt ~]# grep -R . /prod/
/prod/rep2/file2:file2
/prod/rep1/file1:file1
/prod/rep1/prod.txt:siterpod
/prod/rep1/prod.txt:basculement siteb prod
/prod/rep1/prod.txt:reprise prod sitea

# Activation du mirror sur sitea
[ceph: root@cna1 /]# ceph fs snapshot mirror enable cephfs
{}
[ceph: root@cna1 /]# ceph fs snapshot mirror peer_bootstrap import cephfs eyJmc2lkIjogIjEzNzc4ZDcwLWMyZjgtMTFlYy05MTVkLTUyNTQwMGE1N2Q4YSIsICJmaWxlc3lzdGVtIjogImNlcGhmcyIsICJ1c2VyIjogImNsaWVudC5taXJyb3JfcmVtb3RlIiwgInNpdGVfbmFtZSI6ICJyZW1vdGUtc2l0ZSIsICJrZXkiOiAiQVFBWWFHMWlETWd5TXhBQVNXUmJiUUUzSGpMRlQzSjRyclA0blE9PSIsICJtb25faG9zdCI6ICJbdjI6MTkyLjE2OC4xMTEuMjA6MzMwMC8wLHYxOjE5Mi4xNjguMTExLjIwOjY3ODkvMF0gW3YyOjE5Mi4xNjguMTExLjIxOjMzMDAvMCx2MToxOTIuMTY4LjExMS4yMTo2Nzg5LzBdIFt2MjoxOTIuMTY4LjExMS4yMjozMzAwLzAsdjE6MTkyLjE2OC4xMTEuMjI6Njc4OS8wXSJ9
{}


[root@cephclt ~]# mkdir  /prod/rep1/.snap/siteaprod
[root@cephclt ~]# grep -R . /back
/back/rep2/file2:file2
/back/rep1/file1:file1
/back/rep1/prod.txt:siterpod
/back/rep1/prod.txt:basculement siteb prod
# il manque la ligne reprise prod sitea !
# probablement un probleme de point de reprise avec les snapshots

[root@cephclt ~]# rmdir /back/rep1/.snap/s3
[root@cephclt ~]# mkdir /prod/rep1/.snap/sitea2
[root@cephclt ~]# grep -R . /back
/back/rep2/file2:sitearep2
/back/rep1/file1:file1
/back/rep1/prod.txt:siterpod
/back/rep1/prod.txt:basculement siteb prod
/back/rep1/prod.txt:reprise prod sitea
[root@cephclt ~]# ls /back/rep1/.snap
s1  s2  sitea2  siteaprod

# voila c'est corrigé.
# sitea est a nouveau le primaire et les snaps sont a nouveau en mirroir vers siteb
```
Documentation :

https://ceph.io/en/news/blog/2021/new-in-pacific-cephfs-updates/

https://access.redhat.com/documentation/zh-cn/red_hat_ceph_storage/5/html/file_system_guide/ceph-file-system-administration#ceph-file-system-mirroring_fs

Remarques: 
- cephfs-mirroring est stable depuis la version 16.2.5 d'apres les remarques sur les forums
- c'est dommange qu'il n'y a pas de mode tow-way pour gérer le changement de site comme sous rnd-mirroring
- pas d'information de le dashbord sur ce service
- fonctionnement simple et plutot facile a mettre en place. a noté que c'est sur sitea ou il faut installer le service cepfs-mirroring
- la fonction 'mirror peer_bootstrap import' fonctionne du premier cout;)
