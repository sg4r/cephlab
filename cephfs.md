# CephFS
CephFS, est un système de fichiers compatible POSIX permettant de fournir un stockage de fichiers polyvalent, hautement disponible pour l'utilisation 
de répertoires personnels partagés, d'espace de travail HPC. c'est une alternative au service NFS.

CephFS nécessite au moins deux pools RADOS, un pour les données et un pour les métadonnées.
Le pool de métadonnées est utilisé pour le stockage des informations des répertoires et des fichiers : les inodes, les
proprétaires, les dates de création et de modification et les ACLs. Ce pool doit être de type réplication.
Le pool de données est utilisé par par défaut pour le stockage des données.
Il est possible d’utiliser des pools en erasure code avec CephFS, mais il est préférable d'utiliser un pool en réplication
pour la racine du stockage et de définir par la suite des pools en erasure code pour des répertoires spécifiques.
![cephfsarchitecture](cephfs-architecture.svg)
## création CephFS
Depuis la version Octopus, la création d'un CephFS se fait en une ligne de commande.
Ceph Orchestrator créera et configurera automatiquement MDS (pour MetaData Server) pour votre système de fichiers.
```
[ceph: root@cn1 /]# ceph fs volume create moncfs   
```
Vérification, on constat un nouveau service mds qui est installé sur 2 node. voir la ligne ```mds: moncfs:1 {0=moncfs.cn4.qziwip=up:active} 1 up:standby```   
```
[ceph: root@cn1 /]# ceph -s
  cluster:
    id:     2e90db8c-541a-11eb-bb6e-525400ae1f18
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum cn1,cn3,cn2 (age 17m)
    mgr: cn2.rkgnmp(active, since 22m), standbys: cn1.pnzyvw
    mds: moncfs:1 {0=moncfs.cn4.qziwip=up:active} 1 up:standby
    osd: 8 osds: 7 up (since 14m), 7 in (since 4m)
 
  data:
    pools:   5 pools, 113 pgs
    objects: 65 objects, 15 MiB
    usage:   7.2 GiB used, 313 GiB / 320 GiB avail
    pgs:     113 active+clean
```
Vérification, avec ```ceph df```, on retrouve bien ```cephfs.moncfs.meta``` et ```cephfs.moncfs.data```
```
[ceph: root@cn1 /]# ceph df
--- RAW STORAGE ---
CLASS  SIZE     AVAIL    USED     RAW USED  %RAW USED
hdd    360 GiB  352 GiB  237 MiB   8.2 GiB       2.29
TOTAL  360 GiB  352 GiB  237 MiB   8.2 GiB       2.29
 
--- POOLS ---
POOL                   ID  PGS  STORED   OBJECTS  USED     %USED  MAX AVAIL
device_health_metrics   1    1      0 B        0      0 B      0    111 GiB
prbd                    2   16  4.9 MiB       25   15 MiB      0    134 GiB
prbdec                  3   32  8.2 MiB       18   24 MiB      0    190 GiB
cephfs.moncfs.meta      4   32  2.4 KiB       22  1.2 MiB      0    133 GiB
cephfs.moncfs.data      5   32      0 B        0      0 B      0    111 GiB

```
## Création de la clé keyring d'acces à CephFS
le systeme client a besion du fichier /etc/ceph.conf et d'un fichier keyring pour le montage du filesystem sous CephFS.
```
[vagrant@cn1 ~]$ sudo ./cephadm shell ceph fs authorize moncfs client.cephclt / rw >ceph.client.cephclt.keyring
Inferring fsid 2e90db8c-541a-11eb-bb6e-525400ae1f18
Inferring config /var/lib/ceph/2e90db8c-541a-11eb-bb6e-525400ae1f18/mon.cn1/config
Using recent ceph image docker.io/ceph/ceph:v15
#vérification du contenu du fichier
[vagrant@cn1 ~]$ cat ceph.client.cephclt.keyring
[client.cephclt]
	key = AQABrABgEF2WBBAAR4KYvpf0i5NGWkvMwGatTg==
# copy du fichier sur le client
[vagrant@cn1 ~]$ scp ./ceph.client.cephclt.keyring root@cephclt:/etc/ceph/ceph.client.cephclt.keyring
ceph.client.cephclt.keyring                                               100%   65    24.9KB/s   00:00
# protection des informations
[vagrant@cn1 ~]$ ssh root@cephclt chmod -R 640 /etc/ceph
[vagrant@cn1 ~]$ ssh root@cephclt ls -l /etc/ceph
total 16
-rw-r-----. 1 root root  65 Jan 14 20:47 ceph.client.cephclt.keyring
-rw-r-----. 1 root root  62 Jan 12 07:14 ceph.client.prbd.keyring
-rw-r-----. 1 root root 175 Jan 12 07:13 ceph.conf
-rw-r-----. 1 root root 215 Jan 12 13:14 rbdmap
```
## Montage et démontage manuel CephFS
```
[root@cephclt ~]# mkdir /mnt/moncfs
[root@cephclt ~]# mount -t ceph cn1,cn2,cn3,cn4:/ /mnt/moncfs -o name=cephclt
[root@cephclt ~]# df -h /mnt/moncfs
Filesystem                                             Size  Used Avail Use% Mounted on
192.168.0.11,192.168.0.12,192.168.0.13,192.168.0.14:/  111G     0  111G   0% /mnt/moncfs
[root@cephclt ~]# umount /mnt/moncfs
```
## CephFS dans /etc/fstab
Pour monter automatiquement CephFS au démarrage du poste client, insérer la ligne correspondante dans le fichiers /etc/fstab
```
vi /etc/fstab
# ajouter la ligne a la fin du fichier /etc/fstab
cn1,cn2,cn3,cn4:/ /mnt/moncfs ceph name=cephclt,noatime,_netdev 0 2
[root@cephclt ~]# mount -a
[root@cephclt ~]# df -h /mnt/moncfs
Filesystem                                             Size  Used Avail Use% Mounted on
192.168.0.11,192.168.0.12,192.168.0.13,192.168.0.14:/  111G     0  111G   0% /mnt/moncfs
```
## Définition des quotas
CephFS permet de définir des quotas sur n'importe quel répertoire. Le quota peut restreindre le nombre d'octets ou le nombre de fichiers stockés sous ce répertoires. Pour définir un quota à un repertoire, il est nécéssaire d'utiliser la clé ```ceph.client.admin.keyring```.  Par défaut seulement ce compte à les authorisations nécéssaires.

Les quotas dans CephFS ont des limites :
- Les quotas sont coopératifs et non concurrentiels. Les quotas Ceph s'appuient sur le client qui monte le
système de fichiers pour arrêter l’écriture lorsqu'une limite est atteinte.
- Les quotas sont imprécis. Les processus qui écrivent sur le système de fichiers seront arrêtés peu de temps
après que la limite du quota aura été atteinte.
- Les quotas sont implémentés dans le kernel du client à partir de la version 4.17. Les quotas sont pris en
charge par le client de l'espace utilisateur (libcephfs, ceph-fuse).
- Le cluster doit être dans une version mimic ou plus récente. Les anciens clients ne prennent pas en charge
les quotas.
```
# Pour définir un quota de 100 Mo, exécutez :
setfattr -n ceph.quota.max_bytes -v 100000000 /rep/autre
# Pour définir un quota de 10 000 fichiers, exécutez :
setfattr -n ceph.quota.max_files -v 10000 /rep/autre
# Pour afficher la configuration de quota, exécutez :
getfattr -n ceph.quota.max_bytes /rep/autre
getfattr -n ceph.quota.max_files /rep/autre
# Pour supprimer un quota, exécutez :
setfattr -n ceph.quota.max_bytes -v 0 /rep/autre
setfattr -n ceph.quota.max_files -v 0 /rep/autre
```
Appliquer un quota de 2Go à rep1
```
[ftceph@deploy ceph]$ ssh ceph1
[ftceph@deploy ~]$ sudo -i
[root@deploy ~]# yum -y install attr
[root@deploy ~]# setfattr -n ceph.quota.max_bytes -v 2000000000 /mnt/cfs/rep1/
[root@deploy ~]# getfattr -n ceph.quota.max_bytes /mnt/cfs/rep1/
# file: mnt/cfs/rep1/
ceph.quota.max_bytes="2000000000"
[root@ceph1 ~]# exit
logout
[ftceph@ceph1 ~]$ exit
logout
Connection to ceph1 closed.
# vérification de la valeur du quota
[ftceph@deploy ~]$ df -h /opt/cfsrep1/
Filesystem Size Used Avail Use% Mounted on
172.16.7.42,172.16.7.11,172.16.7.44:/rep1 1.9G 0 1.9G 0% /opt/cfsrep1

# vérification de la limite d’écriture
[ftceph@deploy ~]$ mkdir /opt/cfsrep1/ftceph
[ftceph@deploy ~]$ dd if=/dev/zero of=/opt/cfsrep1/ftceph/ddfile bs=1G count=1 oflag=direct
1+0 records in
1+0 records out
1073741824 bytes (1.1 GB) copied, 12.4922 s, 86.0 MB/s
[ftceph@deploy ~]$ dd if=/dev/zero of=/opt/cfsrep1/ftceph/ddfile bs=2G count=1 oflag=direct
dd: error writing ‘/opt/cfsrep1/ftceph/ddfile’: Disk quota exceeded
0+1 records in
0+0 records out
0 bytes (0 B) copied, 0.648858 s, 0.0 kB/s
```
Pour plus d'information sur les quotas voir https://docs.ceph.com/en/latest/cephfs/quota/
## Documentation
Pour plus d’information voir https://docs.ceph.com/en/latest/cephfs/
- le module de gestion des snapshots https://docs.ceph.com/en/latest/cephfs/snap-schedule/
- le module de gestion des replications CephFS https://docs.ceph.com/en/latest/cephfs/cephfs-mirroring/
- gestion de plusieurs pools avec le layout https://docs.ceph.com/en/latest/cephfs/file-layouts/




