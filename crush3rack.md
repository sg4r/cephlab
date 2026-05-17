# Introduction
On m'a demandé lors d'une discussion le comportement de CEPH dans une installation sur 3 racks et que ce qui allait se passer si un rack n'est plus disponible pendant 2 jours max.  
Pour illustrer le fonctionnement j'ai fait un petit lab pour vérifier le fonctionnement avec 6 nodes dans 3 racks avec des pools en réplication x3  
En résumé, CEPH fonctionne s'il y a une réplication par 3 avec un domaine de défaillance par rack.  
Une défaillance par host ne fonctionne pas, mais je voulais le mettre en déviance ;)  
Avec 3 racks, il n'y auras pas une redistribution et de déplacement des données, par contre s'il y a plus que 3 racks, CEPH va le faire automatiquement.  

Remarques:  
ça devrais fonctionne pour 3DC.  
je pense que dans ce cas, il faudrait intégrer une disposition 3DC avec plusieurs racks dans chaque DC  

reste encore a regarder comment faire avec 2 racks.  
si quelque'un a déjà de l'expérience dans ce domaine ?  

Pour plus d'info sur la crushmap voir  
https://docs.ceph.com/en/latest/rados/operations/crush-map/  

## TP
```
# nouveau cluster ceph avec 6 nodes cn1..6 qui ont chacun 2 disques qui ne sont pas de la même taille.

[ceph: root@cn1 /]# ceph osd tree
ID   CLASS  WEIGHT   TYPE NAME           STATUS  REWEIGHT  PRI-AFF
 -1         0.52734  root default                                
-13         0.08789          host cn1                            
  5    hdd  0.03909              osd.5       up   1.00000  1.00000
 11    hdd  0.04880              osd.11      up   1.00000  1.00000
-11         0.08789          host cn2                            
  3    hdd  0.03909              osd.3       up   1.00000  1.00000
 10    hdd  0.04880              osd.10      up   1.00000  1.00000
 -9         0.08789          host cn3                            
  4    hdd  0.03909              osd.4       up   1.00000  1.00000
  8    hdd  0.04880              osd.8       up   1.00000  1.00000
 -3         0.08789          host cn4                            
  0    hdd  0.03909              osd.0       up   1.00000  1.00000
  6    hdd  0.04880              osd.6       up   1.00000  1.00000
 -7         0.08789          host cn5                            
  2    hdd  0.03909              osd.2       up   1.00000  1.00000
  9    hdd  0.04880              osd.9       up   1.00000  1.00000
 -5         0.08789          host cn6                            
  1    hdd  0.03909              osd.1       up   1.00000  1.00000
  7    hdd  0.04880              osd.7       up   1.00000  1.00000


# changement des rêgles

ceph osd crush add-bucket rack1 rack
ceph osd crush add-bucket rack2 rack
ceph osd crush add-bucket rack3 rack
ceph osd crush move rack1 root=default
ceph osd crush move rack2 root=default
ceph osd crush move rack3 root=default
ceph osd crush move cn1 rack=rack1
ceph osd crush move cn2 rack=rack1
ceph osd crush move cn3 rack=rack2
ceph osd crush move cn4 rack=rack2
ceph osd crush move cn5 rack=rack3
ceph osd crush move cn6 rack=rack3

[ceph: root@cn1 /]# ceph osd tree
ID   CLASS  WEIGHT   TYPE NAME           STATUS  REWEIGHT  PRI-AFF
 -1         0.52734  root default                                
-15         0.17578      rack rack1                              
-13         0.08789          host cn1                            
  5    hdd  0.03909              osd.5       up   1.00000  1.00000
 11    hdd  0.04880              osd.11      up   1.00000  1.00000
-11         0.08789          host cn2                            
  3    hdd  0.03909              osd.3       up   1.00000  1.00000
 10    hdd  0.04880              osd.10      up   1.00000  1.00000
-16         0.17578      rack rack2                              
 -9         0.08789          host cn3                            
  4    hdd  0.03909              osd.4       up   1.00000  1.00000
  8    hdd  0.04880              osd.8       up   1.00000  1.00000
 -3         0.08789          host cn4                            
  0    hdd  0.03909              osd.0       up   1.00000  1.00000
  6    hdd  0.04880              osd.6       up   1.00000  1.00000
-17         0.17578      rack rack3                              
 -7         0.08789          host cn5                            
  2    hdd  0.03909              osd.2       up   1.00000  1.00000
  9    hdd  0.04880              osd.9       up   1.00000  1.00000
 -5         0.08789          host cn6                            
  1    hdd  0.03909              osd.1       up   1.00000  1.00000
  7    hdd  0.04880              osd.7       up   1.00000  1.00000

[ceph: root@cn1 /]# ceph osd getcrushmap -o crushmap
27
# remaque c'est la version 27 de crushmap
[ceph: root@cn1 /]# crushtool -i crushmap --test --show-statistics --show-mappings --rule 0 --min-x 1 --max-x 4 --num-rep 3
rule 0 (replicated_rule), x = 1..4, numrep = 3..3
CRUSH rule 0 x 1 [3,0,11]
CRUSH rule 0 x 2 [8,10,6]
CRUSH rule 0 x 3 [0,2,7]
CRUSH rule 0 x 4 [10,7,6]
rule 0 (replicated_rule) num_rep 3 result size == 3:        4/4

# remarque
[3,0,11] rack=>1,2,1 double 1
[8,10,6] rack=>2,1,2 double 2
[0,2,7]  rack=>2,3,3 double 3
[10,7,6] rack=>1,3,2 distrib sur les 3 par hasard!

[ceph: root@cn1 /]# ceph osd crush rule ls
replicated_rule
[ceph: root@cn1 /]# ceph osd crush rule create-replicated reprack default rack hdd
# remarque rêgle "reprack" en replication x3 avec un domain de défaillance rack et une classe hdd
[ceph: root@cn1 /]# ceph osd getcrushmap -o crushmap
28
# remarque, a cahque modificationde la crushmap la version est augmenté
[ceph: root@cn1 /]# crushtool -i crushmap --test --show-statistics --show-mappings --rule 1 --min-x 1 --max-x 4 --num-rep 3
rule 1 (reprack), x = 1..4, numrep = 3..3
CRUSH rule 1 x 1 [4,11,9]
CRUSH rule 1 x 2 [9,3,6]
CRUSH rule 1 x 3 [10,6,7]
CRUSH rule 1 x 4 [8,11,2]
rule 1 (reprack) num_rep 3 result size == 3:        4/4

# vérification
[4,11,9] rack=>2,1,3 ok
[9,3,6]  rack=>3,1,2 ok
[10,6,7] rack=>1,2,3 ok
[8,11,2] rack=>2,1,3 ok
# remarque c'est bien distribué sur les racks

# Y a plus qu'a vérifier comment ça réagit si le rack est off
[ceph: root@cn1 /]# ceph osd pool create prbd 16 16 replicated replicated_rule
pool 'prbd' created
[ceph: root@cn1 /]# ceph osd pool application enable prbd rbd
enabled application 'rbd' on pool 'prbd'
[ceph: root@cn1 /]# rbd create --size 5G --image-feature layering,exclusive-lock prbd/foo
[ceph: root@cn1 /]# ceph osd pool create prack 16 16 replicated reprack
pool 'prack' created
[ceph: root@cn1 /]# ceph osd pool application enable prack rbd
enabled application 'rbd' on pool 'prack'
[ceph: root@cn1 /]# rbd create --size 5G --image-feature layering,exclusive-lock prack/vrack
[ceph: root@cn1 /]# ceph df
--- RAW STORAGE ---
CLASS     SIZE    AVAIL     USED  RAW USED  %RAW USED
hdd    540 GiB  536 GiB  3.5 GiB   3.5 GiB       0.64
TOTAL  540 GiB  536 GiB  3.5 GiB   3.5 GiB       0.64
 
--- POOLS ---
POOL   ID  PGS   STORED  OBJECTS     USED  %USED  MAX AVAIL
.mgr    1    1  449 KiB        2  1.3 MiB      0    170 GiB
prbd    2   16     35 B        4   24 KiB      0    170 GiB
prack   3   16     36 B        4   24 KiB      0    170 GiB

#je passe la configuration du client, pour le test j'ai utilisé le clé ceph.client.admin depuis le client ;)
# montage des data
[root@cephclt ~]# mkdir /mnt/rephost
[root@cephclt ~]# rbd device map prbd/foo
/dev/rbd0
[root@cephclt ~]# mkfs.xfs /dev/rbd/prbd/foo
meta-data=/dev/rbd/prbd/foo      isize=512    agcount=8, agsize=163840 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=0 inobtcount=0
data     =                       bsize=4096   blocks=1310720, imaxpct=25
         =                       sunit=16     swidth=16 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=16 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
Discarding blocks...Done.
[root@cephclt ~]# mount /dev/rbd/prbd/foo /mnt/rephost
[root@cephclt ~]# mkdir /mnt/reprack
[root@cephclt ~]# rbd device map prack/vrack
/dev/rbd1
[root@cephclt ~]# mkfs.xfs /dev/rbd/prack/vrack
meta-data=/dev/rbd/prack/vrack   isize=512    agcount=8, agsize=163840 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=0 inobtcount=0
data     =                       bsize=4096   blocks=1310720, imaxpct=25
         =                       sunit=16     swidth=16 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=16 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
Discarding blocks...Done.
[root@cephclt ~]# mount /dev/rbd/prack/vrack /mnt/reprack
[root@cephclt ~]# df -h |grep mnt
/dev/rbd0       5.0G   69M  5.0G   2% /mnt/rephost
/dev/rbd1       5.0G   69M  5.0G   2% /mnt/reprack
[root@cephclt ~]# dd if=/dev/urandom of=/mnt/rephost/sample.txt bs=1M count=50
52428800 bytes (52 MB, 50 MiB) copied, 4.4752 s, 11.7 MB/s
[root@cephclt ~]# dd if=/dev/urandom of=/mnt/reprack/sample.txt bs=1M count=50
52428800 bytes (52 MB, 50 MiB) copied, 4.45099 s, 11.8 MB/s
[root@cephclt ~]# echo rephost1 >/mnt/rephost/info.txt
[root@cephclt ~]# echo reprack1 >/mnt/reprack/info.txt

# que se passe t il si un rack est down ?


mon.cn1 [WRN] Health detail: HEALTH_WARN 2/5 mons down, quorum cn1,cn2,cn3
mon.cn1 [WRN] [WRN] MON_DOWN: 2/5 mons down, quorum cn1,cn2,cn3
mon.cn1 [WRN]     mon.cn6 (rank 3) addr [v2:192.168.111.16:3300/0,v1:192.168.111.16:6789/0] is down (out of quorum)
mon.cn1 [WRN]     mon.cn5 (rank 4) addr [v2:192.168.111.15:3300/0,v1:192.168.111.15:6789/0] is down (out of quorum)
mon.cn1 [WRN] Health check failed: 4 osds down (OSD_DOWN)
mon.cn1 [WRN] Health check failed: 2 hosts (4 osds) down (OSD_HOST_DOWN)
mon.cn1 [WRN] Health check failed: 1 rack (4 osds) down (OSD_RACK_DOWN)
mon.cn1 [WRN] Health check update: Degraded data redundancy: 188/522 objects degraded (36.015%), 28 pgs degraded (PG_DEGRADED)

[root@cephclt ~]# ceph -s
  cluster:
    id:     13500f16-9169-11ee-aa87-525400474436
    health: HEALTH_WARN
            2 hosts fail cephadm check
            2/5 mons down, quorum cn1,cn2,cn3
            4 osds down
            2 hosts (4 osds) down
            1 rack (4 osds) down
            Reduced data availability: 5 pgs inactive
            Degraded data redundancy: 188/522 objects degraded (36.015%), 28 pgs degraded, 29 pgs undersized
 
  services:
    mon: 5 daemons, quorum cn1,cn2,cn3 (age 4m), out of quorum: cn6, cn5
    mgr: cn1.golfzw(active, since 4h), standbys: cn2.hukvwq
    osd: 12 osds: 8 up (since 4m), 12 in (since 4h)
 
  data:
    pools:   3 pools, 33 pgs
    objects: 174 objects, 598 MiB
    usage:   5.2 GiB used, 535 GiB / 540 GiB avail
    pgs:     15.152% pgs not active
             188/522 objects degraded (36.015%)
             23 active+undersized+degraded
             5  undersized+degraded+peered
             4  active+clean
             1  active+undersized

# remarque: il y a seulement 5pgs inactiven c'est a dire qu'il n'y qu'une copie en dessous de minsize qui est de 2

[root@cephclt ~]# echo rephost2 >>/mnt/rephost/info.txt
[root@cephclt ~]# echo reprack2 >>/mnt/reprack/info.txt
[root@cephclt ~]# cat /mnt/rephost/info.txt
rephost1
rephost2
[root@cephclt ~]# cat /mnt/reprack/info.txt
reprack1
reprack2
# remarque : ça écrit tout de même...

[root@cephclt ~]# dd if=/dev/urandom of=/mnt/reprack/sample.txt bs=1M count=50
52428800 bytes (52 MB, 50 MiB) copied, 4.3907 s, 11.9 MB/s
[root@cephclt ~]# dd if=/dev/urandom of=/mnt/rephost/sample.txt bs=1M count=50
52428800 bytes (52 MB, 50 MiB) copied, 4.51888 s, 11.6 MB/s
# remarque: on peux penser que cela fonctionne, mais cela viens que les données ne sont pas validées sur le dique.
[root@cephclt ~]# dd if=/dev/urandom of=/mnt/reprack/sample.txt conv=fdatasync bs=1M count=50
52428800 bytes (52 MB, 50 MiB) copied, 4.63294 s, 11.3 MB/s
[root@cephclt ~]# dd if=/dev/urandom of=/mnt/rephost/sample.txt conv=fdatasync bs=1M count=50
# avec l'option fdatasync c'est bloquant car il y a 5 pgs qui sont inactive

# redemarrage du rack3 avec cn5,cn6
[root@cephclt ~]# ceph -s
  cluster:
    id:     13500f16-9169-11ee-aa87-525400474436
    health: HEALTH_WARN
            clock skew detected on mon.cn6, mon.cn5
 
  services:
    mon: 5 daemons, quorum cn1,cn2,cn3,cn6,cn5 (age 97s)
    mgr: cn1.golfzw(active, since 4h), standbys: cn2.hukvwq
    osd: 12 osds: 12 up (since 91s), 12 in (since 5h)
 
  data:
    pools:   3 pools, 33 pgs
    objects: 186 objects, 648 MiB
    usage:   5.8 GiB used, 534 GiB / 540 GiB avail
    pgs:     33 active+clean

# remarque: j'ai rien fait et ceph indique  "pgs:     33 active+clean"

[root@cephclt ~]# dd if=/dev/urandom of=/mnt/reprack/sample.txt conv=fdatasync bs=1M count=50
52428800 bytes (52 MB, 50 MiB) copied, 4.41363 s, 11.9 MB/s
[root@cephclt ~]# dd if=/dev/urandom of=/mnt/rephost/sample.txt conv=fdatasync bs=1M count=50
52428800 bytes (52 MB, 50 MiB) copied, 4.33407 s, 12.1 MB/s

# remarque : ça écrit normalement;)
```
