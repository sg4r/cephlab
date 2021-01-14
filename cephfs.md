# CephFS
CephFS, est un système de fichiers compatible POSIX permettant de fournir un stockage de fichiers polyvalent, hautement disponible pour l'utilisation 
de répertoires personnels partagés, d'espace de travail HPC. c'est une alternative au service NFS.

CephFS nécessite au moins deux pools RADOS, un pour les données et un pour les métadonnées.
Le pool de métadonnées est utilisé pour le stockage des informations des répertoires et des fichiers : les inodes, les
proprétaires, les dates de création et de modification et les ACLs. Ce pool doit être de type réplication.
Le pool de données est utilisé par par défaut pour le stockage des données.
Il est possible d’utiliser des pools en erasure code avec CephFS, mais il est préférable d'utiliser un pool en réplication
pour la racine du stockage et de définir par la suite des pools en erasure code pour des répertoires spécifiques.

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
## Montage CephFS

## Documentation
Pour plus d’information voir https://docs.ceph.com/en/latest/cephfs/
