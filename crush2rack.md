# Changement de configuration pour passer de 3rack à 2rack
```
ceph osd crush move cn1 rack=rack1
ceph osd crush move cn2 rack=rack1
ceph osd crush move cn3 rack=rack1
ceph osd crush move cn4 rack=rack2
ceph osd crush move cn5 rack=rack2
ceph osd crush move cn6 rack=rack2
```
# Nouvelle structure
```
[root@cephclt ~]# ceph osd tree
ID   CLASS  WEIGHT   TYPE NAME           STATUS  REWEIGHT  PRI-AFF
 -1         0.52734  root default                                 
-15         0.26367      rack rack1                               
-13         0.08789          host cn1                             
  5    hdd  0.03909              osd.5       up   1.00000  1.00000
 11    hdd  0.04880              osd.11      up   1.00000  1.00000
-11         0.08789          host cn2                             
  3    hdd  0.03909              osd.3       up   1.00000  1.00000
 10    hdd  0.04880              osd.10      up   1.00000  1.00000
 -9         0.08789          host cn3                             
  4    hdd  0.03909              osd.4       up   1.00000  1.00000
  8    hdd  0.04880              osd.8       up   1.00000  1.00000
-16         0.26367      rack rack2                               
 -3         0.08789          host cn4                             
  0    hdd  0.03909              osd.0       up   1.00000  1.00000
  6    hdd  0.04880              osd.6       up   1.00000  1.00000
 -7         0.08789          host cn5                             
  2    hdd  0.03909              osd.2       up   1.00000  1.00000
  9    hdd  0.04880              osd.9       up   1.00000  1.00000
 -5         0.08789          host cn6                             
  1    hdd  0.03909              osd.1       up   1.00000  1.00000
  7    hdd  0.04880              osd.7       up   1.00000  1.00000
-17               0      rack rack3                               

[root@cephclt ~]# ceph -s
  cluster:
    id:     13500f16-9169-11ee-aa87-525400474436
    health: HEALTH_WARN
            Degraded data redundancy: 10 pgs undersized
 
  services:
    mon: 5 daemons, quorum cn1,cn2,cn3,cn6,cn5 (age 68s)
    mgr: cn1.golfzw(active, since 4m), standbys: cn2.hukvwq
    osd: 12 osds: 12 up (since 54s), 12 in (since 7h); 16 remapped pgs
 
  data:
    pools:   3 pools, 33 pgs
    objects: 186 objects, 648 MiB
    usage:   6.2 GiB used, 534 GiB / 540 GiB avail
    pgs:     100/608 objects misplaced (16.447%)
             17 active+clean
             16 active+undersized+remapped
             
# Remarque: ceph réplique toujours le pool 3 sur 3 rack
                     
[ceph: root@cn1 /]# ceph osd getcrushmap -o crushmap
[ceph: root@cn1 /]# crushtool -i crushmap --test --show-statistics --show-mappings --rule 1 --min-x 1 --max-x 4 --num-rep 3
rule 1 (reprack), x = 1..4, numrep = 3..3
CRUSH rule 1 x 1 [7,11]
CRUSH rule 1 x 2 [8,6]
CRUSH rule 1 x 3 [4,9]
CRUSH rule 1 x 4 [9,11]
rule 1 (reprack) num_rep 3 result size == 2:	4/4

# Remarque: logique, il y a seulement 2 racks, il faut donc faire une règle qui sélectionne 2 ods par rack puis sélectionne le reste dans l'autre rack.

[ceph: root@cn1 /]# crushtool -d crushmap -o crushmap.txt
[ceph: root@cn1 /]# cp crushmap.txt crushmap.txt.old
[ceph: root@cn1 /]# vi crushmap.txt
# je rajoute la règle à la fin du fichier
rule rep2rack {
        id 2
        type replicated
        step take default class hdd
        step choose firstn 0 type rack
        step chooseleaf firstn 2 type osd
        step emit
}
[ceph: root@cn1 /]# crushtool -c crushmap.txt -o crushmap.new
[ceph: root@cn1 /]# crushtool -i crushmap.new --test --show-statistics --show-mappings --rule 2 --min-x 1 --max-x 4 --num-rep 3
rule 2 (rep2rack), x = 1..4, numrep = 3..3
CRUSH rule 2 x 1 [7,9,5]
CRUSH rule 2 x 2 [8,3,0]
CRUSH rule 2 x 3 [4,10,2]
CRUSH rule 2 x 4 [9,7,8]
rule 2 (rep2rack) num_rep 3 result size == 3:	4/4

# Vérification

[7,9,5]  =>rack:2,2,1
[8,3,0]  =>rack:1,1,2
[4,10,2] =>rack:1,1,2
[9,7,8]  =>rack:2,2,1

# C'est Ok, mais un des racks n'a qu'une copie ce qui n'est pas top.

[ceph: root@cn1 /]# crushtool -i crushmap.new --test --show-statistics --show-mappings --rule 2 --min-x 1 --max-x 4 --num-rep 4
rule 2 (rep2rack), x = 1..4, numrep = 4..4
CRUSH rule 2 x 1 [7,9,5,11]
CRUSH rule 2 x 2 [8,3,0,7]
CRUSH rule 2 x 3 [4,10,2,0]
CRUSH rule 2 x 4 [9,7,8,11]
rule 2 (rep2rack) num_rep 4 result size == 4:	4/4

# Vérification

[7,9,5,11] =>2,2,1,1  attention 5 et 11 sur le même host!
[8,3,0,7]  =>1,1,2,2
[4,10,2,0] =>1,1,2,2
[9,7,8,11] =>2,2,1,1

# Remarque: il y a bien 2 copies sur chaque rack avec size=4

[ceph: root@cn1 /]# ceph osd setcrushmap -i crushmap.new
43
[ceph: root@cn1 /]# ceph -s
  cluster:
    id:     13500f16-9169-11ee-aa87-525400474436
    health: HEALTH_OK
 
  services:
    mon: 5 daemons, quorum cn1,cn2,cn3,cn6,cn5 (age 12s)
    mgr: cn1.golfzw(active, since 42m), standbys: cn2.hukvwq
    osd: 12 osds: 12 up (since 38m), 12 in (since 7h); 16 remapped pgs
 
  data:
    pools:   3 pools, 33 pgs
    objects: 186 objects, 648 MiB
    usage:   5.4 GiB used, 535 GiB / 540 GiB avail
    pgs:     50/558 objects misplaced (8.961%)
             17 active+clean
             16 active+clean+remapped

# Remarque: rien ne change, c'est normal. Par contre je ne trouve pas comment assigner une autre règle a un pool existant...

# pas grave, je change la règle assignée ;)

[ceph: root@cn1 /]# ceph osd getcrushmap -o crushmap
43
[ceph: root@cn1 /]# crushtool -d crushmap -o crushmap.txt
[ceph: root@cn1 /]# cp crushmap.txt crushmap.txt.old
[ceph: root@cn1 /]# vi crushmap.txt
# j'inverse l'id de la règle 2 avec la règle 1
[ceph: root@cn1 /]# crushtool -c crushmap.txt -o crushmap.new
[ceph: root@cn1 /]# crushtool -i crushmap.new --test --show-statistics --show-mappings --rule 1 --min-x 1 --max-x 4 --num-rep 4
rule 1 (rep2rack), x = 1..4, numrep = 4..4
CRUSH rule 1 x 1 [7,9,5,11]
CRUSH rule 1 x 2 [8,3,0,7]
CRUSH rule 1 x 3 [4,10,2,0]
CRUSH rule 1 x 4 [9,7,8,11]
rule 1 (rep2rack) num_rep 4 result size == 4:	4/4

# Remarque: c'est identique au test précédent mais avec le numéro de règle 1 au lieu de 2

[ceph: root@cn1 /]# ceph osd setcrushmap -i crushmap.new
44
[ceph: root@cn1 /]# ceph -s
  cluster:
    id:     13500f16-9169-11ee-aa87-525400474436
    health: HEALTH_OK
 
  services:
    mon: 5 daemons, quorum cn1,cn2,cn3,cn6,cn5 (age 26m)
    mgr: cn1.golfzw(active, since 68m), standbys: cn2.hukvwq
    osd: 12 osds: 12 up (since 64m), 12 in (since 8h)
 
  data:
    pools:   3 pools, 33 pgs
    objects: 186 objects, 648 MiB
    usage:   5.4 GiB used, 535 GiB / 540 GiB avail
    pgs:     33 active+clean

# Remarque: HEALTH_OK avec un size à 3

[ceph: root@cn1 /]#  ceph osd pool set prack size 4
set pool 3 size to 4
[ceph: root@cn1 /]# ceph -s
  cluster:
    id:     13500f16-9169-11ee-aa87-525400474436
    health: HEALTH_OK
 
  services:
    mon: 5 daemons, quorum cn1,cn2,cn3,cn6,cn5 (age 28m)
    mgr: cn1.golfzw(active, since 70m), standbys: cn2.hukvwq
    osd: 12 osds: 12 up (since 66m), 12 in (since 8h)
 
  data:
    pools:   3 pools, 33 pgs
    objects: 186 objects, 648 MiB
    usage:   5.6 GiB used, 534 GiB / 540 GiB avail
    pgs:     33 active+clean
 
  io:
    recovery: 15 MiB/s, 0 keys/s, 4 objects/s

# Ca augmente le nombre de copies

[ceph: root@cn1 /]# ceph -s
  cluster:
    id:     13500f16-9169-11ee-aa87-525400474436
    health: HEALTH_OK
 
  services:
    mon: 5 daemons, quorum cn1,cn2,cn3,cn6,cn5 (age 28m)
    mgr: cn1.golfzw(active, since 70m), standbys: cn2.hukvwq
    osd: 12 osds: 12 up (since 67m), 12 in (since 8h)
 
  data:
    pools:   3 pools, 33 pgs
    objects: 186 objects, 648 MiB
    usage:   5.6 GiB used, 534 GiB / 540 GiB avail
    pgs:     33 active+clean
[ceph: root@cn1 /]#  ceph osd dump | grep pool
pool 1 '.mgr' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 1 pgp_num 1 autoscale_mode on last_change 26 flags hashpspool stripe_width 0 pg_num_max 32 pg_num_min 1 application mgr
pool 2 'prbd' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 16 pgp_num 16 autoscale_mode on last_change 241 flags hashpspool,selfmanaged_snaps stripe_width 0 application rbd
pool 3 'prack' replicated size 4 min_size 2 crush_rule 1 object_hash rjenkins pg_num 16 pgp_num 16 autoscale_mode on last_change 397 flags hashpspool,selfmanaged_snaps stripe_width 0 application rbd

# Remarque: J'ai bien le pool 3 prack en size 4, reste a vérifier comment ça réagit si un rack est down

[root@cephclt ~]# rbd device map prack/vrack
/dev/rbd0
[root@cephclt ~]# mount /dev/rbd/prack/vrack /mnt/reprack
[root@cephclt ~]# df -h /mnt/reprack
Filesystem      Size  Used Avail Use% Mounted on
/dev/rbd0       5.0G  119M  4.9G   3% /mnt/reprack
[root@cephclt ~]#  dd if=/dev/urandom of=/mnt/reprack/sample.txt conv=fdatasync bs=1M count=50
52428800 bytes (52 MB, 50 MiB) copied, 4.8874 s, 10.7 MB/s
[root@cephclt ~]# echo rep2rackv1 >>/mnt/reprack/info.txt
[root@cephclt ~]# cat /mnt/reprack/info.txt 
reprack1
reprack2
rep2rackv1

# Remarque: Je stoppe cn4,cn5,cn6

[root@cephclt ~]# ceph -s
  cluster:
    id:     13500f16-9169-11ee-aa87-525400474436
    health: HEALTH_WARN
            failed to probe daemons or devices
            2/5 mons down, quorum cn1,cn2,cn3
            6 osds down
            3 hosts (6 osds) down
            1 rack (6 osds) down
            Reduced data availability: 7 pgs inactive, 1 pg stale
            Degraded data redundancy: 271/608 objects degraded (44.572%), 31 pgs degraded
 
  services:
    mon: 5 daemons, quorum cn1,cn2,cn3 (age 63s), out of quorum: cn6, cn5
    mgr: cn1.golfzw(active, since 91m), standbys: cn2.hukvwq
    osd: 12 osds: 6 up (since 62s), 12 in (since 8h)
 
  data:
    pools:   3 pools, 33 pgs
    objects: 186 objects, 650 MiB
    usage:   5.6 GiB used, 534 GiB / 540 GiB avail
    pgs:     21.212% pgs not active
             271/608 objects degraded (44.572%)
             24 active+undersized+degraded
             7  undersized+degraded+peered
             1  stale+active+clean
             1  active+clean

[root@cephclt ~]#  dd if=/dev/urandom of=/mnt/reprack/sample.txt conv=fdatasync bs=1M count=50
52428800 bytes (52 MB, 50 MiB) copied, 4.55039 s, 11.5 MB/s

# Remarque: Ca écris toujours, c'est pas bloquant

[root@cephclt ~]# echo rep2rackv2 >>/mnt/reprack/info.txt
[root@cephclt ~]# cat /mnt/reprack/info.txt
reprack1
reprack2
rep2rackv1
rep2rackv2

[root@cephclt ~]# ceph pg dump pgs |grep peered |awk {'print $1'}
dumped pgs
2.f
2.2
2.1
1.0
2.6
2.b
2.e

# Remarque: Les pgs mal placés sont seulement dans le pool 2 qui a une règle avec un domaine de défaillance en host, donc c'est logique que dans cette configuration il y a des pgs manquant.

# Je relance cn4,cn5,cn6

[root@cephclt ~]# ceph -s
  cluster:
    id:     13500f16-9169-11ee-aa87-525400474436
    health: HEALTH_WARN
            clock skew detected on mon.cn6, mon.cn5
 
  services:
    mon: 5 daemons, quorum cn1,cn2,cn3,cn6,cn5 (age 63s)
    mgr: cn1.golfzw(active, since 102m), standbys: cn2.hukvwq
    osd: 12 osds: 12 up (since 54s), 12 in (since 8h)
 
  data:
    pools:   3 pools, 33 pgs
    objects: 186 objects, 650 MiB
    usage:   7.2 GiB used, 533 GiB / 540 GiB avail
    pgs:     33 active+clean

# Remarque: CEPH retrouve ces petits.

# Ca fonctionne, mais c'est parce que j'avais assez de mon et de mgr dans rack1, ce qui n'ai pas le cas si je coupe rack1...

# Ajout des clé admin sur les nodes

[ceph: root@cn1 /]# ceph orch host label add cn4 _admin
Added label _admin to host cn4
[ceph: root@cn1 /]# ceph orch host label add cn5 _admin
Added label _admin to host cn5

# Ajouter un node supplémentaire pour faire le quorum si un rack est off. ici ce sera sur cnrgw1

# Définir ou placer les mon et les mgr => 2 dans chaque rack et 1 sur cnrgw1 qui est dans rack3

[ceph: root@cn1 /]# ceph orch apply mon --placement="5 cn2 cn3 cn4 cn5 cnrgw1"
Scheduled mon update...
[ceph: root@cn1 /]# ceph orch apply mgr --placement="5 cn2 cn3 cn4 cn5 cnrgw1"
Scheduled mgr update...
[ceph: root@cn1 /]# ceph orch ls
NAME                       PORTS        RUNNING  REFRESHED  AGE  PLACEMENT                       
alertmanager               ?:9093,9094      1/1  6m ago     3h   count:1                         
crash                                       7/7  6m ago     3d   *                               
grafana                    ?:3000           1/1  24s ago    3h   count:1                         
mgr                                         3/2  3m ago     33s  cn2;cn3;cn4;cn5;cnrgw1;count:5  
mon                                         5/5  6m ago     44s  cn2;cn3;cn4;cn5;cnrgw1;count:5  
node-exporter              ?:9100           7/7  6m ago     3d   *                               
osd.all-available-devices                    12  6m ago     3d   *                               
prometheus                 ?:9095           1/1  6m ago     3h   count:1                        

# Ca semble bien, je coupe cn1, cn2, cn3 qui sont dans le rack1

[root@cephclt ~]# dd if=/dev/urandom of=/mnt/reprack/sample.txt conv=fdatasync bs=1M count=50
52428800 bytes (52 MB, 50 MiB) copied, 4.56714 s, 11.5 MB/s

# C'est toujours possible d'écrire depuis le cephclt

[root@cephclt ~]# ceph -s
  cluster:
    id:     13500f16-9169-11ee-aa87-525400474436
    health: HEALTH_WARN
            3 hosts fail cephadm check
            clock skew detected on mon.cnrgw1
            2/5 mons down, quorum cn4,cnrgw1,cn5
            6 osds down
            3 hosts (6 osds) down
            1 rack (6 osds) down
            Reduced data availability: 9 pgs inactive
            Degraded data redundancy: 295/581 objects degraded (50.775%), 31 pgs degraded, 31 pgs undersized
 
  services:
    mon: 5 daemons, quorum cn4,cnrgw1,cn5 (age 9m), out of quorum: cn2, cn3
    mgr: cn5.wmhygl(active, since 8m), standbys: cnrgw1.dkgnot, cn4.ijljbv
    osd: 12 osds: 6 up (since 9m), 12 in (since 13h)
 
  data:
    pools:   3 pools, 33 pgs
    objects: 177 objects, 614 MiB
    usage:   2.7 GiB used, 267 GiB / 270 GiB avail
    pgs:     3.030% pgs unknown
             24.242% pgs not active
             295/581 objects degraded (50.775%)
             23 active+undersized+degraded
             8  undersized+degraded+peered
             1  active+clean
 
# Remarque: Il reste seulement 3 mon, le mgr actif est bien dans le rack2, il y a seulement 6 osd sur 12, 50% des 581 objects sont accessible.

# Je redémmarre cn1 cn2 et cn3

[root@cephclt ~]# ceph -s
  cluster:
    id:     13500f16-9169-11ee-aa87-525400474436
    health: HEALTH_WARN
            clock skew detected on mon.cn3, mon.cnrgw1
 
  services:
    mon: 5 daemons, quorum cn2,cn3,cn4,cnrgw1,cn5 (age 104s)
    mgr: cnrgw1.dkgnot(active, since 101s), standbys: cn2.hukvwq, cn5.wmhygl, cn3.pbcwyn, cn4.ijljbv
    osd: 12 osds: 12 up (since 80s), 12 in (since 13h)
 
  data:
    pools:   3 pools, 33 pgs
    objects: 186 objects, 650 MiB
    usage:   5.6 GiB used, 534 GiB / 540 GiB avail
    pgs:     33 active+clean

 
# Je stoppe cn4 cn5 cn6 du rack2

[root@cephclt ~]# dd if=/dev/urandom of=/mnt/reprack/sample.txt conv=fdatasync bs=1M count=50
52428800 bytes (52 MB, 50 MiB) copied, 4.48153 s, 11.7 MB/s
[root@cephclt ~]# ceph -s
  cluster:
    id:     13500f16-9169-11ee-aa87-525400474436
    health: HEALTH_WARN
            3 hosts fail cephadm check
            clock skew detected on mon.cnrgw1
            2/5 mons down, quorum cn2,cn3,cnrgw1
            6 osds down
            3 hosts (6 osds) down
            1 rack (6 osds) down
            Reduced data availability: 5 pgs inactive
            Degraded data redundancy: 276/608 objects degraded (45.395%), 32 pgs degraded
 
  services:
    mon: 5 daemons, quorum cn2,cn3,cnrgw1 (age 14s), out of quorum: cn4, cn5
    mgr: cnrgw1.dkgnot(active, since 3m), standbys: cn2.hukvwq, cn5.wmhygl, cn3.pbcwyn, cn4.ijljbv
    osd: 12 osds: 6 up (since 28s), 12 in (since 13h)
 
  data:
    pools:   3 pools, 33 pgs
    objects: 186 objects, 650 MiB
    usage:   5.6 GiB used, 534 GiB / 540 GiB avail
    pgs:     21.212% pgs not active
             276/608 objects degraded (45.395%)
             24 active+undersized+degraded
             7  undersized+degraded+peered
             1  stale+active+undersized+degraded
             1  active+clean
 
  io:
    client:   7.4 MiB/s wr, 0 op/s rd, 5 op/s wr

[root@cephclt ~]# dd if=/dev/urandom of=/mnt/reprack/sample.txt conv=fdatasync bs=1M count=50
52428800 bytes (52 MB, 50 MiB) copied, 4.4546 s, 11.8 MB/s

# Ca fonctionne toujours.

# Je relance cn4 cn5 cn6

[root@cephclt ~]# ceph -s
  cluster:
    id:     13500f16-9169-11ee-aa87-525400474436
    health: HEALTH_WARN
            clock skew detected on mon.cn4, mon.cnrgw1
 
  services:
    mon: 5 daemons, quorum cn2,cn3,cn4,cnrgw1,cn5 (age 40s)
    mgr: cnrgw1.dkgnot(active, since 7m), standbys: cn2.hukvwq, cn3.pbcwyn, cn4.ijljbv, cn5.wmhygl
    osd: 12 osds: 12 up (since 21s), 12 in (since 13h)
 
  data:
    pools:   3 pools, 33 pgs
    objects: 187 objects, 650 MiB
    usage:   5.6 GiB used, 534 GiB / 540 GiB avail
    pgs:     33 active+clean
 
  io:
    client:   5.9 KiB/s wr, 0 op/s rd, 0 op/s wr

# Il y a bien les 33 pgs en "active+clean"
```
