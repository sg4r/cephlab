# Mise à niveau
Cephadm est capable de mettre à niveau et en toute sécurité un cluster Ceph d'une version de correction de bogue à une autre.
Par exemple, vous pouvez passer de la v15.2.0 (la première version d'Octopus) à la prochaine version ponctuelle v15.2.1.
Vous pouvez aussi passer a une version majeur de la même maniere ;)

Le processus de mise à niveau automatisé suit les meilleures pratiques Ceph. Par exemple:
 - L'ordre de mise à niveau commence avec les gestionnaires, les moniteurs, puis les autres démons.
 - Chaque démon est redémarré uniquement après que Ceph indique que le cluster restera disponible.
 - Gardez à l'esprit que l'état d'intégrité du cluster Ceph est susceptible de basculer sur HEALTH_WARNING pendant la mise à niveau.

## Exemple de mise à niveau de la version 15.2.9 à 16.2.0 (Pacific)
mise à jour via la commande ```ceph orch upgrade``` vers la version 16.2.0
```
# vérification de l'etat du cluster
[ceph: root@cn1 /]# ceph -s
  cluster:
    id:     be024bba-5f15-11eb-ad27-5254001065ff
    health: HEALTH_OK
 
  services:
    mon: 5 daemons, quorum cn1,cn2,cn3,cnrgw1,cn4 (age 18m)
    mgr: cn1.vvraig(active, since 18m), standbys: cn2.zphxce
    osd: 8 osds: 8 up (since 18m), 8 in (since 3w)
    rgw: 2 daemons active (demodom.fr-est-1.cnrgw1.umtgun, demodom.fr-est-2.cnrgw2.fggict)
 
  task status:
 
  data:
    pools:   13 pools, 289 pgs
    objects: 1.28k objects, 29 KiB
    usage:   8.8 GiB used, 351 GiB / 360 GiB avail
    pgs:     289 active+clean
 
  io:
    client:   11 KiB/s rd, 0 B/s wr, 10 op/s rd, 10 op/s wr
 
  progress:
    Upgrade to docker.io/ceph/ceph:v16.2.0 (0s)
      [............................] 
 
# dans quelle version je suis 
[ceph: root@cn1 /]# ceph versions
{
    "mon": {
        "ceph version 15.2.9 (357616cbf726abb779ca75a551e8d02568e15b17) octopus (stable)": 5
    },
    "mgr": {
        "ceph version 15.2.9 (357616cbf726abb779ca75a551e8d02568e15b17) octopus (stable)": 2
    },
    "osd": {
        "ceph version 15.2.9 (357616cbf726abb779ca75a551e8d02568e15b17) octopus (stable)": 8
    },
    "mds": {},
    "rgw": {
        "ceph version 15.2.9 (357616cbf726abb779ca75a551e8d02568e15b17) octopus (stable)": 2
    },
    "overall": {
        "ceph version 15.2.9 (357616cbf726abb779ca75a551e8d02568e15b17) octopus (stable)": 17
    }
}

# migration avec une commande
[ceph: root@cn1 /]#  ceph orch upgrade start --ceph-version 16.2.0
Initiating upgrade to docker.io/ceph/ceph:v16.2.0

# vérification de ce qui se passe
[ceph: root@cn1 /]# ceph versions
{
    "mon": {
        "ceph version 15.2.9 (357616cbf726abb779ca75a551e8d02568e15b17) octopus (stable)": 3,
        "ceph version 16.2.0 (0c2054e95bcd9b30fdd908a79ac1d8bbc3394442) pacific (stable)": 2
    },
    "mgr": {
        "ceph version 16.2.0 (0c2054e95bcd9b30fdd908a79ac1d8bbc3394442) pacific (stable)": 2
    },
    "osd": {
        "ceph version 15.2.9 (357616cbf726abb779ca75a551e8d02568e15b17) octopus (stable)": 8
    },
    "mds": {},
    "rgw": {
        "ceph version 15.2.9 (357616cbf726abb779ca75a551e8d02568e15b17) octopus (stable)": 2
    },
    "overall": {
        "ceph version 15.2.9 (357616cbf726abb779ca75a551e8d02568e15b17) octopus (stable)": 13,
        "ceph version 16.2.0 (0c2054e95bcd9b30fdd908a79ac1d8bbc3394442) pacific (stable)": 4
    }
}

# Remarque: cela commence par les mgr, puis les mon

[ceph: root@cn1 /]# ceph -s
  cluster:
    id:     be024bba-5f15-11eb-ad27-5254001065ff
    health: HEALTH_OK
 
  services:
    mon: 5 daemons, quorum cn1,cn2,cn3,cnrgw1,cn4 (age 66s)
    mgr: cn1.vvraig(active, since 99s), standbys: cn2.zphxce
    osd: 8 osds: 8 up (since 26m), 8 in (since 3w)
    rgw: 2 daemons active (2 hosts, 2 zones)
 
  data:
    pools:   13 pools, 289 pgs
    objects: 1.39k objects, 29 KiB
    usage:   8.8 GiB used, 351 GiB / 360 GiB avail
    pgs:     289 active+clean
 
  io:
    client:   511 B/s rd, 0 op/s rd, 0 op/s wr
 
  progress:
    Upgrade to 16.2.0 (16s)
      [===.........................] (remaining: 112s)

# le cluster est toujours oppérationnel

[ceph: root@cn1 /]# ceph -W cephadm
  cluster:
    id:     be024bba-5f15-11eb-ad27-5254001065ff
    health: HEALTH_OK
 
  services:
    mon: 5 daemons, quorum cn1,cn2,cn3,cnrgw1,cn4 (age 5m)
    mgr: cn1.vvraig(active, since 11m), standbys: cn2.zphxce
    osd: 8 osds: 8 up (since 13s), 8 in (since 3w)
    rgw: 1 daemon active (1 hosts, 1 zones)
 
  data:
    pools:   13 pools, 289 pgs
    objects: 1.39k objects, 29 KiB
    usage:   1.2 GiB used, 359 GiB / 360 GiB avail
    pgs:     289 active+clean
 
  io:
    client:   33 KiB/s rd, 0 B/s wr, 33 op/s rd, 14 op/s wr
 
  progress:
    Upgrade to 16.2.0 (10m)
      [===================.........] (remaining: 4m)
 

2021-04-04T01:26:35.817499+0000 mgr.cn1.vvraig [INF] Upgrade: Setting container_image for all rgw
2021-04-04T01:26:36.132912+0000 mgr.cn1.vvraig [INF] Upgrade: Setting container_image for all rbd-mirror
2021-04-04T01:26:36.201017+0000 mgr.cn1.vvraig [INF] Upgrade: Setting container_image for all iscsi
2021-04-04T01:26:36.280148+0000 mgr.cn1.vvraig [INF] Upgrade: Setting container_image for all nfs
2021-04-04T01:26:38.776443+0000 mgr.cn1.vvraig [INF] Upgrade: Updating node-exporter.cn1
2021-04-04T01:26:38.777344+0000 mgr.cn1.vvraig [INF] Deploying daemon node-exporter.cn1 on cn1
2021-04-04T01:26:47.806717+0000 mgr.cn1.vvraig [INF] Upgrade: Updating node-exporter.cn2
2021-04-04T01:26:47.806961+0000 mgr.cn1.vvraig [INF] Deploying daemon node-exporter.cn2 on cn2
2021-04-04T01:26:54.899946+0000 mgr.cn1.vvraig [INF] Upgrade: Updating node-exporter.cn3
2021-04-04T01:26:54.900154+0000 mgr.cn1.vvraig [INF] Deploying daemon node-exporter.cn3 on cn3
2021-04-04T01:27:01.640376+0000 mgr.cn1.vvraig [INF] Upgrade: Updating node-exporter.cn4
2021-04-04T01:27:01.640554+0000 mgr.cn1.vvraig [INF] Deploying daemon node-exporter.cn4 on cn4
2021-04-04T01:27:06.941037+0000 mgr.cn1.vvraig [INF] Upgrade: Updating node-exporter.cnrgw1
2021-04-04T01:27:06.941244+0000 mgr.cn1.vvraig [INF] Deploying daemon node-exporter.cnrgw1 on cnrgw1
2021-04-04T01:27:11.666549+0000 mgr.cn1.vvraig [INF] Upgrade: Updating node-exporter.cnrgw2
2021-04-04T01:27:11.666826+0000 mgr.cn1.vvraig [INF] Deploying daemon node-exporter.cnrgw2 on cnrgw2
2021-04-04T01:27:16.813336+0000 mgr.cn1.vvraig [INF] Upgrade: Updating prometheus.cn1
2021-04-04T01:27:16.886056+0000 mgr.cn1.vvraig [INF] Deploying daemon prometheus.cn1 on cn1
2021-04-04T01:27:27.235493+0000 mgr.cn1.vvraig [INF] Upgrade: Updating alertmanager.cn1
2021-04-04T01:27:27.248582+0000 mgr.cn1.vvraig [INF] Deploying daemon alertmanager.cn1 on cn1
2021-04-04T01:27:40.359685+0000 mgr.cn1.vvraig [INF] Upgrade: Updating grafana.cn1
2021-04-04T01:27:40.783075+0000 mgr.cn1.vvraig [INF] Deploying daemon grafana.cn1 on cn1
2021-04-04T01:28:50.176234+0000 mgr.cn1.vvraig [INF] Upgrade: Finalizing container_image settings
2021-04-04T01:28:50.739604+0000 mgr.cn1.vvraig [INF] Upgrade: Complete!

# remarque: c'est plus simple a suivre

[ceph: root@cn1 /]# ceph versions
{
    "mon": {
        "ceph version 16.2.0 (0c2054e95bcd9b30fdd908a79ac1d8bbc3394442) pacific (stable)": 5
    },
    "mgr": {
        "ceph version 16.2.0 (0c2054e95bcd9b30fdd908a79ac1d8bbc3394442) pacific (stable)": 2
    },
    "osd": {
        "ceph version 16.2.0 (0c2054e95bcd9b30fdd908a79ac1d8bbc3394442) pacific (stable)": 8
    },
    "mds": {},
    "rgw": {
        "ceph version 16.2.0 (0c2054e95bcd9b30fdd908a79ac1d8bbc3394442) pacific (stable)": 2
    },
    "overall": {
        "ceph version 16.2.0 (0c2054e95bcd9b30fdd908a79ac1d8bbc3394442) pacific (stable)": 17
    }
}

# c'est migré en 16.2.0

[ceph: root@cn1 /]# ceph -s
  cluster:
    id:     be024bba-5f15-11eb-ad27-5254001065ff
    health: HEALTH_OK
 
  services:
    mon: 5 daemons, quorum cn1,cn2,cn3,cnrgw1,cn4 (age 8m)
    mgr: cn1.vvraig(active, since 14m), standbys: cn2.zphxce
    osd: 8 osds: 8 up (since 3m), 8 in (since 3w)
    rgw: 2 daemons active (2 hosts, 2 zones)
 
  data:
    pools:   13 pools, 289 pgs
    objects: 1.39k objects, 29 KiB
    usage:   1.2 GiB used, 359 GiB / 360 GiB avail
    pgs:     289 active+clean
 
  io:
    client:   27 KiB/s rd, 0 B/s wr, 27 op/s rd, 15 op/s wr

# ok tout fonctionne
[ceph: root@cn1 /]# 
```
