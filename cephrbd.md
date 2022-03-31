Le système de stockage en mode bloc de Ceph permet de monter une image RBD (RADOS Block Devices) comme un
périphérique de type bloc. Quand une application écrit des données sur Ceph en utilisant un périphérique de bloc,
Ceph entrelace et réplique les données à travers le cluster automatiquement. Ceph RBD est généralement utilisé
pour les disques de machines virtuelles.
![ceph-dashboard](cephrbd.png)

La commande **rbd** vous permet de créer, d’afficher ou de modifier les informations et de supprimer des images RBD.
Vous pouvez également l'utiliser pour la gestion des snapshots ou des fonctions de clonage des images.

```
# création du pool prbd en mode réplication
[ceph: root@cn1 /]# ceph osd pool create prbd 16          
pool 'prbd' created

[ceph: root@cn1 /]# ceph osd pool application enable prbd rbd
enabled application 'rbd' on pool 'prbd'
# création de l'image
[ceph: root@cn1 /]# rbd create --size 5G --image-feature layering,exclusive-lock prbd/foo

[ceph: root@cn1 /]# rbd info prbd/foo
rbd image 'foo':
	size 5 GiB in 1280 objects
	order 22 (4 MiB objects)
	snapshot_count: 0
	id: 5edc33207a70
	block_name_prefix: rbd_data.5edc33207a70
	format: 2
	features: layering, exclusive-lock
	op_features: 
	flags: 
	create_timestamp: Mon Jan 11 21:02:33 2021
	access_timestamp: Mon Jan 11 21:02:33 2021
	modify_timestamp: Mon Jan 11 21:02:33 2021

# création du pool prbdec en mode erasure code avec k=2 et m=2
[ceph: root@cn1 /]# ceph osd pool create prbdec erasure
pool 'prbdec' created

[ceph: root@cn1 /]# ceph osd pool application enable prbdec rbd
enabled application 'rbd' on pool 'prbdec'

[ceph: root@cn1 /]# ceph osd pool set prbdec allow_ec_overwrites true
set pool 3 allow_ec_overwrites to true
# creation de l'image en mode erasure code.
# remarque : l'image est dans le pool prbd et les data sont dans le pool prbdec
[ceph: root@cn1 /]# rbd create --size 5G --image-feature layering,exclusive-lock prbd/foo-ec --data-pool prbdec

[ceph: root@cn1 /]# rbd info prbd/foo-ec
rbd image 'foo-ec':
	size 5 GiB in 1280 objects
	order 22 (4 MiB objects)
	snapshot_count: 0
	id: 85d4c9b4eccb
	data_pool: prbdec
	block_name_prefix: rbd_data.2.85d4c9b4eccb
	format: 2
	features: layering, exclusive-lock, data-pool
	op_features: 
	flags: 
	create_timestamp: Mon Jan 11 21:32:41 2021
	access_timestamp: Mon Jan 11 21:32:41 2021
	modify_timestamp: Mon Jan 11 21:32:41 2021

# remarque : on retrouve bien l'info data_pool: prbdec

[ceph: root@cn1 /]# ceph df
--- RAW STORAGE ---
CLASS  SIZE     AVAIL    USED    RAW USED  %RAW USED
hdd    360 GiB  352 GiB  52 MiB   8.1 GiB       2.24
TOTAL  360 GiB  352 GiB  52 MiB   8.1 GiB       2.24
 
--- POOLS ---
POOL                   ID  PGS  STORED   OBJECTS  USED     %USED  MAX AVAIL
device_health_metrics   1    1      0 B        0      0 B      0    111 GiB
prbd                    2   16  6.3 KiB        6  595 KiB      0    111 GiB
prbdec                  3   32    8 KiB        1  256 KiB      0    166 GiB

# Remarque : l'espace n'est consommé qu'à l'utilisation.
# Remarque : on voit la différence d'espace disponible entre la replication x3 et l'erasure code k=2,m=2

# exit container
[ceph: root@cn1 /]# exit
exit
```
## préparer le client
Il est nécessaire d'installer les binaires de Ceph sur le client ainsi que les fichiers de configuration pour joindre le cluster Ceph.
```
# créé la clé pour le client pour l'acces au pool prbd
[vagrant@cn1 ~]$ sudo ./cephadm shell ceph auth get-or-create client.prbd mon 'profile rbd' osd 'profile rbd pool=prbd, profile rbd pool=prbdec' > ceph.client.prbd.keyring
Inferring fsid 2e90db8c-541a-11eb-bb6e-525400ae1f18
Inferring config /var/lib/ceph/2e90db8c-541a-11eb-bb6e-525400ae1f18/mon.cn1/config
Using recent ceph image docker.io/ceph/ceph:v15
# vérification de la structure du fichier
[vagrant@cn1 ~]$ cat ./ceph.client.prbd.keyring 
[client.prbd]
	key = AQCnx/xfhTjDDhAAis6rFEStxGIdORJ+TVpCig==

# remarque: la clé ne doit être accessible qu'à ceph. Dans ce TP on bypass quelques règles de sécurité et d'isolation ;)
# install repo ceph-octopus
[vagrant@cn1 ~]$ ssh root@cephclt dnf install centos-release-ceph-octopus.noarch
The authenticity of host 'cephclt (192.168.0.10)' can't be established.
RSA key fingerprint is SHA256:wJMZR1dtOjr0FNdEGJ7Lgg9f14+YDIUp+RbCvq3xQwM.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'cephclt,192.168.0.10' (RSA) to the list of known hosts.
# install du binaire sur le client
[vagrant@cn1 ~]$ ssh root@cephclt dnf install ceph-common
# vérifie ceph.conf
[vagrant@cn1 ~]$ ls /etc/ceph
ceph.client.admin.keyring  ceph.conf  ceph.pub
# copie ceph.conf sur le client
[vagrant@cn1 ~]$ scp /etc/ceph/ceph.conf root@cephclt:/etc/ceph
ceph.conf                                                                    100%  175    68.4KB/s   00:00    
# copie de la clé pour l'accès au pool prbd sur le client
[vagrant@cn1 ~]$ scp ./ceph.client.prbd.keyring root@cephclt:/etc/ceph/
ceph.client.prbd.keyring                                                    100%   62    37.2KB/s   00:00    
```
##  montage et démontage du volume
```
[vagrant@cn1 ~]$ ssh root@cephclt
Last login: Tue Jan 12 07:20:12 2021 from 192.168.0.11
# montage image rbd (réplication)
[root@cephclt ~]# mkdir /mnt/rbd
[root@cephclt ~]# rbd -n client.prbd device map prbd/foo
/dev/rbd0
[root@cephclt ~]# rbd -n client.prbd device ls
id  pool  namespace  image  snap  device   
0   prbd             foo    -     /dev/rbd0
[root@cephclt ~]# mkfs.xfs /dev/rbd/prbd/foo
meta-data=/dev/rbd/prbd/foo      isize=512    agcount=8, agsize=163840 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=1310720, imaxpct=25
         =                       sunit=16     swidth=16 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=16 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
Discarding blocks...Done.

[root@cephclt ~]# mount /dev/rbd0 /mnt/rbd
[root@cephclt ~]# df -h /mnt/rbd
Filesystem      Size  Used Avail Use% Mounted on
/dev/rbd0       5.0G   69M  5.0G   2% /mnt/rbd

[root@cephclt ~]# rbd -n client.prbd resize --size 10G prbd/foo
Resizing image: 100% complete...done.
[root@cephclt ~]# xfs_growfs /mnt/rbd
meta-data=/dev/rbd0              isize=512    agcount=8, agsize=163840 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=1310720, imaxpct=25
         =                       sunit=16     swidth=16 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=16 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 1310720 to 2621440
[root@cephclt ~]# df -h /mnt/rbd
Filesystem      Size  Used Avail Use% Mounted on
/dev/rbd0        10G  105M  9.9G   2% /mnt/rbd

[root@cephclt ~]# rbd -n client.prbd ls -l prbd
NAME    SIZE    PARENT  FMT  PROT  LOCK
foo     10 GiB            2        excl
foo-ec   5 GiB            2            

# Remarque : il y a un verrou sur l’image pour éviter qu’un autre client ne monte une image déjà utilisée, ceci afin d’éviter de corrompre les données.

# montage image rbdec (erasure code)
[root@cephclt ~]# mkdir /mnt/rbdec
[root@cephclt ~]# rbd -n client.prbd device map prbd/foo-ec
/dev/rbd1
# remarque: on indique seulement le nom de l'image. pas besion d'indiquer le nom du pool ec pour les datas
[root@cephclt ~]# rbd -n client.prbd device ls
id  pool  namespace  image   snap  device   
0   prbd             foo     -     /dev/rbd0
1   prbd             foo-ec  -     /dev/rbd1
[root@cephclt ~]# mkfs.xfs /dev/rbd/prbd/foo-ec 
meta-data=/dev/rbd/prbd/foo-ec   isize=512    agcount=8, agsize=163840 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=1310720, imaxpct=25
         =                       sunit=16     swidth=16 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=16 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
Discarding blocks...Done.
[root@cephclt ~]# mount /dev/rbd1 /mnt/rbdec/
[root@cephclt ~]#  df -h /mnt/rbdec/
Filesystem      Size  Used Avail Use% Mounted on
/dev/rbd1       5.0G   69M  5.0G   2% /mnt/rbdec
[root@cephclt ~]# rbd -n client.prbd resize --size 10G prbd/foo-ec
Resizing image: 100% complete...done.
[root@cephclt ~]# xfs_growfs /mnt/rbdec
meta-data=/dev/rbd1              isize=512    agcount=8, agsize=163840 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=1310720, imaxpct=25
         =                       sunit=16     swidth=16 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=16 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 1310720 to 2621440
[root@cephclt ~]# df -h /mnt/rbdec
Filesystem      Size  Used Avail Use% Mounted on
/dev/rbd1        10G  105M  9.9G   2% /mnt/rbdec
[root@cephclt ~]# rbd -n client.prbd ls -l prbd
NAME    SIZE    PARENT  FMT  PROT  LOCK
foo     10 GiB            2        excl
foo-ec  10 GiB            2        excl

# Démontage des  images
[root@cephclt ~]# umount /mnt/rbd
[root@cephclt ~]# rbd -n client.prbd device unmap prbd/foo
[root@cephclt ~]# umount /mnt/rbdec
[root@cephclt ~]# rbd -n client.prbd device unmap prbd/foo-ec
[root@cephclt ~]# rbd -n client.prbd device ls
```
## Montage des images RBD au démarrage du système
rbdmap est un script shell qui automatise les opérations rbd map et rbd unmap des images RBD pour un montage
automatique au démarrage ou pour le démontage à l’arrêt du système. La configuration du service rbdmap.service
pour systemd est fournie avec le paquetage ceph-common.
Le script shell accepte un argument unique, qui peut être map ou unmap. La configuration des images est définie
dans /etc/ceph/rbdmap.
```
# modifier le fichier /etc/ceph/rbdmap
[root@cephclt ~]# cat /etc/ceph/rbdmap
# RbdDevice		Parameters
#poolname/imagename	id=client,keyring=/etc/ceph/ceph.client.keyring
prbd/foo id=prbd,keyring=/etc/ceph/ceph.client.prbd.keyring
prbd/foo-ec id=prbd,keyring=/etc/ceph/ceph.client.prbd.keyring
[root@cephclt ~]# rbdmap map
# vérification : les devices sont bien montés
[root@cephclt ~]# rbd device ls
id  pool  namespace  image   snap  device   
0   prbd             foo     -     /dev/rbd0
1   prbd             foo-ec  -     /dev/rbd1
[root@cephclt ~]# rbdmap unmap
[root@cephclt ~]# rbd device ls
[root@cephclt ~]#
[root@cephclt ~]# systemctl enable rbdmap
Created symlink /etc/systemd/system/multi-user.target.wants/rbdmap.service → /usr/lib/systemd/system/rbdmap.service.
[root@cephclt ~]# systemctl start rbdmap
[root@cephclt ~]# echo /dev/rbd/prbd/foo /mnt/rbd xfs noauto 0 0 >>/etc/fstab
[root@cephclt ~]# echo /dev/rbd/prbd/foo-ec /mnt/rbdec xfs noauto 0 0 >>/etc/fstab
# vérification
[root@cephclt ~]# rbdmap unmap
[root@cephclt ~]# rbdmap map
[root@cephclt ~]# df -h |grep mnt
/dev/rbd0                     10G  105M  9.9G   2% /mnt/rbd
/dev/rbd1                     10G  105M  9.9G   2% /mnt/rbdec
# si le montage fonctionne, alors on peut tester via un reboot
[root@cephclt ~]# reboot
Connection to cephclt closed by remote host.
Connection to cephclt closed.
[vagrant@cn1 ~]$ 
# reconnexion à la vm cephclt
[vagrant@cn1 ~]$ ssh root@cephclt
Last login: Tue Jan 12 13:13:01 2021 from 192.168.0.11
# Vérification du montage automatique
[root@cephclt ~]# df -h |grep mnt
/dev/rbd0                     10G  105M  9.9G   2% /mnt/rbd
/dev/rbd1                     10G  105M  9.9G   2% /mnt/rbdec
# Yes ca marche ;)
```
## Gestion des snapshots
Un instantané est une copie en lecture seule de l'état d'une image à un moment donné. Il est ainsi possible de
conserver un historique de l'état d'une image en fonction du temps.
Avec XFS, il est nécessaire d’utiliser xfs_freeze pour garantir un état cohérent du système de fichiers pendant la
création du snapshot. Ceci a pour effet de vider les caches et de bloquer les modifications du système de fichiers
pendant la durée de création du snapshot.

```
[root@cephclt ~]# echo version1 >>/mnt/rbd/fichier.txt
[root@cephclt ~]# xfs_freeze -f /mnt/rbd
[root@cephclt ~]# rbd -n client.prbd snap create prbd/foo@snapv1
[root@cephclt ~]# xfs_freeze -u /mnt/rbd
[root@cephclt ~]# rbd -n client.prbd snap ls prbd/foo
SNAPID  NAME    SIZE    PROTECTED  TIMESTAMP               
     4  snapv1  10 GiB             Tue Jan 12 13:35:28 2021
[root@cephclt ~]# echo version2 >>/mnt/rbd/fichier.txt
[root@cephclt ~]# cat /mnt/rbd/fichier.txt
version1
version2

# retour à l'état initiale
[root@cephclt ~]# rbd -n client.prbd status prbd/foo
Watchers:
	watcher=192.168.0.10:0/862625393 client.44190 cookie=18446462598732840961
[root@cephclt ~]# rbdmap unmap
[root@cephclt ~]# rbd -n client.prbd snap rollback prbd/foo@snapv1
Rolling back to snapshot: 100% complete...done.
[root@cephclt ~]# rbdmap map
[root@cephclt ~]# cat /mnt/rbd/fichier.txt
version1
[root@cephclt ~]# 
# Yep c'est ok ;)
```
Remarque : Un retour à l'état initial d’un snapshot revient à écraser la version actuelle de l'image avec les données
d'un instantané. Le temps nécessaire à l'exécution d'un retour à l'état initial augmente avec la taille de l'image. Il est
plus rapide de cloner une image à partir d'un instantané que de rétablir une image vers un instantané.
Pour éviter de perdre toutes les modifications avant le rollback, il est conseillé de faire un snapshot avant le
démontage de l'image.
Le snapshot permet de revenir à une version stable sur l’ensemble du disque. Préférez l’utilisation de la fonction
clone pour récupérer seulement quelques éléments depuis un snapshot. Par exemple certains fichiers d’un
utilisateur, mais pas de tous les utilisateurs...
## Gestion des clones
Ceph offre la possibilité de créer de nombreux clones depuis un snapshot afin d’obtenir rapidement des images à
partir d'une version de référence. Un clone est une copie à l'écriture (COW) d'un instantané qui fait référence, donc
pour éviter tout désastre vous devez protéger cet instantané avant de le cloner.
```
[root@cephclt ~]# rbd -n client.prbd snap protect prbd/foo@snapv1
# Vous ne pouvez plus supprimer un instantané protégé.

[root@cephclt ~]# cat /mnt/rbd/fichier.txt
version1
# modification du fichier
[root@cephclt ~]# echo version2 >>/mnt/rbd/fichier.txt 
[root@cephclt ~]# rbd -n client.prbd clone prbd/foo@snapv1 prbd/fooclone
[root@cephclt ~]# mkdir /opt/fooclone
[root@cephclt ~]# rbd -n client.prbd map prbd/fooclone
/dev/rbd2
[root@cephclt ~]# mount -t xfs -o rw,nouuid /dev/rbd/prbd/fooclone /opt/fooclone
[root@cephclt ~]# cat /opt/fooclone/fichier.txt
version1
# On retrouve la version initiale du fichier
[root@cephclt ~]# echo version3>> /opt/fooclone/fichier.txt
[root@cephclt ~]# cat /opt/fooclone/fichier.txt
version1
version3
# il est même modifiable

#liste des clones d'un snapshot
[root@cephclt ~]# rbd -n client.prbd children prbd/foo@snapv1
prbd/fooclone
#suppression d’un snap
[root@cephclt ~]# rbd -n client.prbd snap rm prbd/foo@snapv1
Removing snap: 0% complete...failed.
rbd: snapshot 'snapv1' is protected from removal.
2021-01-12T16:10:49.449+0000 7fcc4ef3f500 -1 librbd::Operations: snapshot is protected
# pour faire un clone, il faut protéger le snapshot contre sa suppression ;)
[root@cephclt ~]# umount /opt/fooclone/
[root@cephclt ~]# rbd -n client.prbd unmap prbd/fooclone
[root@cephclt ~]# rbd -n client.prbd snap unprotect prbd/foo@snapv1
2021-01-12T16:11:32.107+0000 7f95927fc700 -1 librbd::SnapshotUnprotectRequest: cannot unprotect: at least 1 child(ren) [aca63043ec89] in pool 'prbd'
[...]
(16) Device or resource busy
# Normal, il reste un clone à ce snapshot
[root@cephclt ~]# rbd -n client.prbd rm prbd/fooclone
Removing image: 100% complete...done.
[root@cephclt ~]# rbd -n client.prbd snap unprotect prbd/foo@snapv1
[root@cephclt ~]# rbd -n client.prbd snap rm prbd/foo@snapv1
Removing snap: 100% complete...done.
# Et voila ;)

#purger tous les snap d'une image
[root@cephclt ~]# rbd -n client.prbd snap purge prbd/foo
[root@cephclt ~]# 
```

## rbd live migration
Lors de la création d'un pool, il est nécéssaire définir celui-ci en mode réplication ou en mode erasure. Si vous devez changer de type de paramètre, il est nécéssaire de migrer les données vers un autre pool avec minimun de temps d'indisponiblité.

Pour cela nous allons utiliser les commandes "rbd migration"

Remarque: krbd ne support pas le live migration. ce mode fonctionne seulement avec librbd et kvm. mais on verra qu'il est possible d'utiliser cette fonction malgré tout avec les montages rbd comme disque distant ;)

Le processus de migration se déroule en trois étapes :

- Préparer la migration : cela consiste a creer une nouvelle image cible (de destination) et de la lier a l'image source. l'image source sera ensuite en lecture seul et les ecritures seront diriger vers l'iamge de déstination. 
- Exécuter la migration:  il s'agit d'une opération en arrière-plan qui copie les blocs de l'image source vers l'image cible. il est possible de différenrer le lancement de cette etape pendant une période ou il y a moins d'activité.
- Terminer la migration : vous pouvez valider ou abandonner la migration une fois le processus de migration en arrière-plan terminé. La validation de la migration supprime les liens entre les images source et cible, et supprimera l'image source. L'abandon de la migration supprime les liens croisés et supprime l'image cible.

en resumé:
1) arrêté l'acces a l'image
2) pour une migration vers un pool en replication :
```bd migration prepare SRC_POOL/IMAGE TARGET_POOL/IMAGE```
ou pour une migration vers un pool EC 
```rbd migration prepare SRC_POOL/IMAGE --data-pool TARGET_POOL/IMAGE```
3) le client peux remonté l'image depuis target pool.
4) ```rbd migration execute SRC_POOL/IMAGE``` # migration en arriere plan
5) ```rbd migration commit SRC_POOL/IMAGE```  # valide la migration

Dans le TP précédant, on a utiliser un disque distant prbd/foo en repliquation. nous allons migrer vers le pool prbdec.

```
#vérification
[root@cephclt ~]# df -h |grep mnt
/dev/rbd0        10G  105M  9.9G   2% /mnt/rbd
/dev/rbd1        10G  105M  9.9G   2% /mnt/rbdec
[root@cephclt ~]# cat /mnt/rbd/fichier.txt 
version1
[root@cephclt ~]# echo "avant migration en ec" >> /mnt/rbd/fichier.txt

# les volumes sont montées via krbd

[ceph: root@cn1 /]# rbd -p prbd ls -l
NAME    SIZE    PARENT  FMT  PROT  LOCK
foo     10 GiB            2        excl
foo-ec  10 GiB            2        excl

# pour rappel l'image foo est en réplication et l'image foo-ec en erasure code

# démarrer la migration de l'image foo
[ceph: root@cn1 /]# rbd migration prepare prbd/foo prbd/foo2ec --data-pool prbdec
2022-03-31T18:01:22.728+0000 7f1ac0886500 -1 librbd::Migration: prepare: image has watchers - not migrating
rbd: preparing migration failed: (16) Device or resource busy

# l'erreur est normal étant donnée qu'il faut démonter l'iamge avant de commencer ;)

[root@cephclt ~]# umount /mnt/rbd
[root@cephclt ~]# rbd device unmap /dev/rbd/prbd/foo

#recommençons maintenant que l'image est démontée
[ceph: root@cn1 /]# rbd migration prepare prbd/foo prbd/foo2ec --data-pool prbdec
[ceph: root@cn1 /]# rbd info prbd/foo2ec
rbd image 'foo2ec':
	size 10 GiB in 2560 objects
	order 22 (4 MiB objects)
	snapshot_count: 0
	id: 2cff2f920bbc0
	data_pool: prbdec
	block_name_prefix: rbd_data.5.2cff2f920bbc0
	format: 2
	features: layering, exclusive-lock, data-pool, migrating

# on remarque que l'image a le flag migrating et qu'il y a bien data_pool

[ceph: root@cn1 /]# rbd status prbd/foo2ec
Watchers: none
Migration:
	source: prbd/foo (2cfb1364a966c)
	destination: prbd/foo2ec (2cff2f920bbc0)
	state: prepared

# l'image prbd/foo2ec a bien une image source et est en mode prepared.
# c'est tout bon, mais comment accéder maintenant a mon volume rbd depuis cephclt ?

[root@cephclt ~]# rbd -n client.prbd device map prbd/foo2ec
rbd: sysfs write failed
RBD image feature set mismatch. This image cannot be mapped because the following immutable features are unsupported by the kernel: migrating.
In some cases useful info is found in syslog - try "dmesg | tail".
rbd: map failed: (6) No such device or address

# effectivement krbd ne support pas cette fonction.

# installer le client rbd-ndb avec un acces via librbd en userspace
[root@cephclt ~]# yum install rbd-nbd

[root@cephclt ~]# rbd-nbd -n client.prbd  map prbd/foo2ec
/dev/nbd0
[root@cephclt ~]# rbd-nbd list-mapped
id    pool  namespace  image   snap  device   
4079  prbd             foo2ec  -     /dev/nbd0
[root@cephclt ~]# mount /dev/nbd0 /mnt/rbd
[root@cephclt ~]# cat /mnt/rbd/fichier.txt
version1
avant migration en ec
[root@cephclt ~]# echo "avant migration execute en ec" >> /mnt/rbd/fichier.txt

# on retrouve bien nos données depuis le client
# et depuis ceph le montage est confirmé

[ceph: root@cn1 /]# rbd status prbd/foo2ec
Watchers:
	watcher=192.168.111.10:0/1067399112 client.184320 cookie=140629845747984
Migration:
	source: prbd/foo (2cfb1364a966c)
	destination: prbd/foo2ec (2cff2f920bbc0)
	state: prepared

# migront les données dans le poolec

[ceph: root@cn1 /]# rbd migration execute prbd/foo2ec
Image migration: 100% complete...done.
[ceph: root@cn1 /]# rbd status prbd/foo2ec
Watchers:
	watcher=192.168.111.10:0/1067399112 client.184320 cookie=140629845747984
Migration:
	source: prbd/foo (2cfb1364a966c)
	destination: prbd/foo2ec (2cff2f920bbc0)
	state: executed

# si c'est ok, supprimon l'image source
[ceph: root@cn1 /]# rbd migration commit prbd/foo2ec
Commit image migration: 100% complete...done.
[ceph: root@cn1 /]# rbd status prbd/foo2ec
Watchers:
	watcher=192.168.111.10:0/1067399112 client.184320 cookie=140629845747984

# on peux garder le montage sur le client ou remonter avec krbd

[root@cephclt ~]# cat /mnt/rbd/fichier.txt
version1
avant migration en ec
avant migration execute en ec
[root@cephclt ~]# echo "apres migration commit en ec" >> /mnt/rbd/fichier.txt

[root@cephclt ~]# umount /mnt/rbd
[root@cephclt ~]# rbd-nbd unmap /dev/nbd0
[root@cephclt ~]# rbd-nbd list-mapped
[root@cephclt ~]# rbd -n client.prbd device map prbd/foo2ec
/dev/rbd0
[root@cephclt ~]# mount /dev/rbd/prbd/foo2ec /mnt/rbd
[root@cephclt ~]# cat /mnt/rbd/fichier.txt
version1
avant migration en ec
avant migration execute en ec
apres migration commit en ec

# Yes, on retrouve nos données

# remarque : la migration d'une image d'un pool ec vers un pool en réplication est également supporté.
```


## Documentation
Pour plus d’informations voir https://docs.ceph.com/en/latest/rbd/
