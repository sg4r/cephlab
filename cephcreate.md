# Cephadmin
cephadm est un outil pour déployer et gérer un cluster Ceph par connexion aux hôtes à partir du démon gestionnaire (mgr) via SSH. Il permet d'ajouter, supprimer ou mettre à jour des conteneurs de démons Ceph. Il ne repose pas sur des outils de configuration ou d'orchestration externes comme Ansible, Rook ou Salt.

cephadm est le nouvel outil de gestion d'un cluster Ceph fourni à partir de la version Octopus v15.2.0 et ne prend pas en charge les anciennes versions de Ceph. Il est possible de migrer les anciennes instances vers cephadmin. cephadmin remplace ceph-deploy utilisé dans les anciennes version de Ceph.
## Installation du cluster Ceph avec cephadm
```
[vagrant@cn1 ~]$ sudo yum -y install https://download.ceph.com/rpm-15.2.10/el8/noarch/cephadm-15.2.10-0.el8.noarch.rpm
Failed to set locale, defaulting to C.UTF-8
Last metadata expiration check: 0:00:33 ago on Tue Mar 22 21:04:34 2022.
cephadm-15.2.10-0.el8.noarch.rpm                       48 kB/s |  58 kB     00:01    
Dependencies resolved.
======================================================================================
 Package       Arch     Version                                  Repository      Size
======================================================================================
Installing:
 cephadm       noarch   2:15.2.10-0.el8                          @commandline    58 k
Installing dependencies:
 python3-pip   noarch   9.0.3-20.el8.rocky.0                     appstream       19 k
 python36      x86_64   3.6.8-38.module+el8.5.0+671+195e4563     appstream       18 k
Enabling module streams:
 python36               3.6                                                          

Transaction Summary
======================================================================================
Install  3 Packages

Total size: 95 k
Total download size: 37 k
Installed size: 235 k
Downloading Packages:
(1/2): python3-pip-9.0.3-20.el8.rocky.0.noarch.rpm     99 kB/s |  19 kB     00:00    
(2/2): python36-3.6.8-38.module+el8.5.0+671+195e4563.  91 kB/s |  18 kB     00:00    
--------------------------------------------------------------------------------------
Total                                                  45 kB/s |  37 kB     00:00     
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                              1/1 
  Installing       : python36-3.6.8-38.module+el8.5.0+671+195e4563.x86_64         1/3 
  Running scriptlet: python36-3.6.8-38.module+el8.5.0+671+195e4563.x86_64         1/3 
  Installing       : python3-pip-9.0.3-20.el8.rocky.0.noarch                      2/3 
  Running scriptlet: cephadm-2:15.2.10-0.el8.noarch                               3/3 
  Installing       : cephadm-2:15.2.10-0.el8.noarch                               3/3 
  Running scriptlet: cephadm-2:15.2.10-0.el8.noarch                               3/3 
  Verifying        : python3-pip-9.0.3-20.el8.rocky.0.noarch                      1/3 
  Verifying        : python36-3.6.8-38.module+el8.5.0+671+195e4563.x86_64         2/3 
  Verifying        : cephadm-2:15.2.10-0.el8.noarch                               3/3 

Installed:
  cephadm-2:15.2.10-0.el8.noarch                                                      
  python3-pip-9.0.3-20.el8.rocky.0.noarch                                             
  python36-3.6.8-38.module+el8.5.0+671+195e4563.x86_64                                

Complete!
[vagrant@cn1 ~]$ sudo mkdir -p /etc/ceph
[vagrant@cn1 ~]$ sudo cephadm bootstrap --mon-ip 192.168.111.11
Verifying podman|docker is present...
Verifying lvm2 is present...
Verifying time synchronization is in place...
Unit chronyd.service is enabled and running
Repeating the final host check...
podman|docker (/bin/podman) is present
systemctl is present
lvcreate is present
Unit chronyd.service is enabled and running
Host looks OK
Cluster fsid: d3232dac-aa23-11ec-b513-5254004e3b60
Verifying IP 192.168.111.11 port 3300 ...
Verifying IP 192.168.111.11 port 6789 ...
Mon IP 192.168.111.11 is in CIDR network 192.168.111.0/24
Pulling container image docker.io/ceph/ceph:v15...
Extracting ceph user uid/gid from container image...
Creating initial keys...
Creating initial monmap...
Creating mon...
Waiting for mon to start...
Waiting for mon...
mon is available
Assimilating anything we can from ceph.conf...
Generating new minimal ceph.conf...
Restarting the monitor...
Setting mon public_network...
Creating mgr...
Verifying port 9283 ...
Wrote keyring to /etc/ceph/ceph.client.admin.keyring
Wrote config to /etc/ceph/ceph.conf
Waiting for mgr to start...
Waiting for mgr...
mgr not available, waiting (1/10)...
mgr not available, waiting (2/10)...
mgr not available, waiting (3/10)...
mgr not available, waiting (4/10)...
mgr is available
Enabling cephadm module...
Waiting for the mgr to restart...
Waiting for Mgr epoch 5...
Mgr epoch 5 is available
Setting orchestrator backend to cephadm...
Generating ssh key...
Wrote public SSH key to to /etc/ceph/ceph.pub
Adding key to root@localhost's authorized_keys...
Adding host cn1...
Deploying mon service with default placement...
Deploying mgr service with default placement...
Deploying crash service with default placement...
Enabling mgr prometheus module...
Deploying prometheus service with default placement...
Deploying grafana service with default placement...
Deploying node-exporter service with default placement...
Deploying alertmanager service with default placement...
Enabling the dashboard module...
Waiting for the mgr to restart...
Waiting for Mgr epoch 13...
Mgr epoch 13 is available
Generating a dashboard self-signed certificate...
Creating initial admin user...
Fetching dashboard port number...
Ceph Dashboard is now available at:

	     URL: https://cn1:8443/
	    User: admin
	Password: 1qk7yr3ksx

You can access the Ceph CLI with:

	sudo /sbin/cephadm shell --fsid d3232dac-aa23-11ec-b513-5254004e3b60 -c /etc/ceph/ceph.conf -k /etc/ceph/ceph.client.admin.keyring

Please consider enabling telemetry to help improve Ceph:

	ceph telemetry on

For more information see:

	https://docs.ceph.com/docs/master/mgr/telemetry/

Bootstrap complete.

# Votre cluster Ceph est créé. Notez bien la valeur du Password par défaut. il sera utilisé plus tard pour accéder au dashboard.
# 

# Ajout ssh ceph.pub key sur les nodes 2 à 4
[vagrant@cn1 ~]$ for i in {2..4}; do ssh-copy-id -f -i /etc/ceph/ceph.pub root@cn$i; done
# Ajout cephadm sur tous les nodes 2 à 4
[vagrant@cn1 ~]$ for i in {2..4}; do ssh root@cn$i dnf  -y install https://download.ceph.com/rpm-15.2.10/el8/noarch/cephadm-15.2.10-0.el8.noarch.rpm; done

# Connexion au cluster ceph
[vagrant@cn1 ~]$ sudo cephadm shell 
Inferring fsid d3232dac-aa23-11ec-b513-5254004e3b60
Inferring config /var/lib/ceph/d3232dac-aa23-11ec-b513-5254004e3b60/mon.cn1/config
Using recent ceph image docker.io/ceph/ceph@sha256:056637972a107df4096f10951e4216b21fcd8ae0b9fb4552e628d35df3f61139
[ceph: root@cn1 /]# 

# Remarque  : on passe dans le conteneur. Attention au changement de shell en [ceph: root@cn1 /]

[ceph: root@cn1 /]# ceph -s
  cluster:
    id:     d3232dac-aa23-11ec-b513-5254004e3b60
    health: HEALTH_WARN
            mon is allowing insecure global_id reclaim
            OSD count 0 < osd_pool_default_size 3
 
  services:
    mon: 1 daemons, quorum cn1 (age 4m)
    mgr: cn1.lgzsul(active, since 3m)
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     
 
[ceph: root@cn1 /]# 

# Remarque : Le cluster est en warring au début, car il n'y a pas encore d'osd ou plusieurs mon, mais on va rêgler cela rapidement

[ceph: root@cn1 /]# cat /etc/redhat-release 
CentOS Linux release 8.3.2011
# Remarque : l'OS du conteneur est basé sur CentOS Linux release 8.3.2011

# Ajouter les noeudes supplémentaires au cluster Ceph
[ceph: root@cn1 /]# ceph orch host add cn2
Added host 'cn2'
[ceph: root@cn1 /]# ceph orch host add cn3
Added host 'cn3'
[ceph: root@cn1 /]# ceph orch host add cn4
Added host 'cn4'

# liste des noeudes dans le cluster. On retrouve bien les 4 noeudes
[ceph: root@cn1 /]# ceph orch host ls
HOST  ADDR  LABELS  STATUS  
cn1   cn1                   
cn2   cn2                   
cn3   cn3                   
cn4   cn4                   

# l'ajout des noeudes va augmenter automatiquement le nombre de process dans le cluster.
[ceph: root@cn1 /]# ceph orch ls
NAME           RUNNING  REFRESHED  AGE  PLACEMENT  IMAGE NAME                            IMAGE ID      
alertmanager       1/1  51s ago    7m   count:1    docker.io/prom/alertmanager:v0.20.0   0881eb8f169f  
crash              1/4  51s ago    8m   *          docker.io/ceph/ceph:v15               5553b0cb212c  
grafana            1/1  51s ago    7m   count:1    docker.io/ceph/ceph-grafana:6.6.2     a0dce381714a  
mgr                1/2  51s ago    8m   count:2    docker.io/ceph/ceph:v15               5553b0cb212c  
mon                1/5  51s ago    8m   count:5    docker.io/ceph/ceph:v15               5553b0cb212c  
node-exporter      1/4  51s ago    7m   *          docker.io/prom/node-exporter:v0.18.1  e5a616e4b9cf  
prometheus         1/1  51s ago    7m   count:1    docker.io/prom/prometheus:v2.18.1     de242295e225  

# Surveillons les modifications au niveau du cluster 
[ceph: root@cn1 /]# ceph -w
  cluster:
    id:     d3232dac-aa23-11ec-b513-5254004e3b60
    health: HEALTH_WARN
            mon is allowing insecure global_id reclaim
            OSD count 0 < osd_pool_default_size 3
 
  services:
    mon: 1 daemons, quorum cn1 (age 14m)
    mgr: cn1.lgzsul(active, since 14m)
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     
 

2022-03-22T22:49:11.711857+0000 mon.cn1 [INF] mon.cn1 calling monitor election
2022-03-22T22:49:13.715668+0000 mon.cn2 [INF] mon.cn2 calling monitor election
2022-03-22T22:49:16.777530+0000 mon.cn1 [INF] mon.cn1 is new leader, mons cn1,cn2 in quorum (ranks 0,1)
2022-03-22T22:49:16.858114+0000 mon.cn1 [WRN] Health detail: HEALTH_WARN mon is allowing insecure global_id reclaim; OSD count 0 < osd_pool_default_size 3
2022-03-22T22:49:16.858197+0000 mon.cn1 [WRN] [WRN] AUTH_INSECURE_GLOBAL_ID_RECLAIM_ALLOWED: mon is allowing insecure global_id reclaim
2022-03-22T22:49:16.858213+0000 mon.cn1 [WRN]     mon.cn1 has auth_allow_insecure_global_id_reclaim set to true
2022-03-22T22:49:16.858225+0000 mon.cn1 [WRN] [WRN] TOO_FEW_OSDS: OSD count 0 < osd_pool_default_size 3
2022-03-22T22:49:17.747510+0000 mon.cn1 [INF] mon.cn1 calling monitor election
2022-03-22T22:49:17.806623+0000 mon.cn2 [INF] mon.cn2 calling monitor election
2022-03-22T22:49:19.724439+0000 mon.cn3 [INF] mon.cn3 calling monitor election
2022-03-22T22:49:22.800376+0000 mon.cn1 [INF] mon.cn1 is new leader, mons cn1,cn2,cn3 in quorum (ranks 0,1,2)
2022-03-22T22:49:22.893486+0000 mon.cn1 [WRN] Health detail: HEALTH_WARN mon is allowing insecure global_id reclaim; OSD count 0 < osd_pool_default_size 3
2022-03-22T22:49:22.893639+0000 mon.cn1 [WRN] [WRN] AUTH_INSECURE_GLOBAL_ID_RECLAIM_ALLOWED: mon is allowing insecure global_id reclaim
2022-03-22T22:49:22.893659+0000 mon.cn1 [WRN]     mon.cn1 has auth_allow_insecure_global_id_reclaim set to true
2022-03-22T22:49:22.893673+0000 mon.cn1 [WRN] [WRN] TOO_FEW_OSDS: OSD count 0 < osd_pool_default_size 3
2022-03-22T22:49:26.799057+0000 mon.cn1 [WRN] Health check update: mons are allowing insecure global_id reclaim (AUTH_INSECURE_GLOBAL_ID_RECLAIM_ALLOWED)
2022-03-22T22:50:00.000381+0000 mon.cn1 [WRN] Health detail: HEALTH_WARN mons are allowing insecure global_id reclaim; OSD count 0 < osd_pool_default_size 3
2022-03-22T22:50:00.000460+0000 mon.cn1 [WRN] [WRN] AUTH_INSECURE_GLOBAL_ID_RECLAIM_ALLOWED: mons are allowing insecure global_id reclaim
2022-03-22T22:50:00.000479+0000 mon.cn1 [WRN]     mon.cn1 has auth_allow_insecure_global_id_reclaim set to true
2022-03-22T22:50:00.000495+0000 mon.cn1 [WRN]     mon.cn2 has auth_allow_insecure_global_id_reclaim set to true
2022-03-22T22:50:00.000513+0000 mon.cn1 [WRN]     mon.cn3 has auth_allow_insecure_global_id_reclaim set to true
2022-03-22T22:50:00.000537+0000 mon.cn1 [WRN] [WRN] TOO_FEW_OSDS: OSD count 0 < osd_pool_default_size 3
2022-03-22T23:00:00.000368+0000 mon.cn1 [WRN] overall HEALTH_WARN mons are allowing insecure global_id reclaim; OSD count 0 < osd_pool_default_size 3
2022-03-22T23:10:00.000332+0000 mon.cn1 [WRN] overall HEALTH_WARN mons are allowing insecure global_id reclaim; OSD count 0 < osd_pool_default_size 3
2022-03-22T23:20:00.000324+0000 mon.cn1 [WRN] overall HEALTH_WARN mons are allowing insecure global_id reclaim; OSD count 0 < osd_pool_default_size 3
2022-03-22T23:30:00.000314+0000 mon.cn1 [WRN] overall HEALTH_WARN mons are allowing insecure global_id reclaim; OSD count 0 < osd_pool_default_size 3


# au bout de 3 minutes, on retrouve bien 4/4 service crash, 3/5 mon, 4/4 node-exporter 
[ceph: root@cn1 /]# ceph orch ls
NAME           RUNNING  REFRESHED  AGE  PLACEMENT  IMAGE NAME                            IMAGE ID      
alertmanager       1/1  63s ago    13m  count:1    docker.io/prom/alertmanager:v0.20.0   0881eb8f169f  
crash              4/4  65s ago    13m  *          docker.io/ceph/ceph:v15               5553b0cb212c  
grafana            1/1  63s ago    13m  count:1    docker.io/ceph/ceph-grafana:6.6.2     a0dce381714a  
mgr                2/2  64s ago    13m  count:2    docker.io/ceph/ceph:v15               5553b0cb212c  
mon                3/5  65s ago    13m  count:5    docker.io/ceph/ceph:v15               5553b0cb212c  
node-exporter      4/4  65s ago    13m  *          docker.io/prom/node-exporter:v0.18.1  e5a616e4b9cf  
prometheus         1/1  63s ago    13m  count:1    docker.io/prom/prometheus:v2.18.1     de242295e225  

# le cluster Ceph est toujours en warring, mais il a maintenant 3 moniteurs et 2 mgr
  cluster:
    id:     d3232dac-aa23-11ec-b513-5254004e3b60
    health: HEALTH_WARN
            mons are allowing insecure global_id reclaim
            OSD count 0 < osd_pool_default_size 3
 
  services:
    mon: 3 daemons, quorum cn1,cn2,cn3 (age 7h)
    mgr: cn1.lgzsul(active, since 9h), standbys: cn2.nbvlgl
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     

# ajoutons des disques au cluster, mais avant cela listons l'ensemble des disques libres existants
[ceph: root@cn1 /]# ceph orch device ls
Hostname  Path      Type  Serial  Size   Health   Ident  Fault  Available  
cn1       /dev/vdb  hdd           42.9G  Unknown  N/A    N/A    Yes        
cn1       /dev/vdc  hdd           53.6G  Unknown  N/A    N/A    Yes        
cn2       /dev/vdb  hdd           42.9G  Unknown  N/A    N/A    Yes        
cn2       /dev/vdc  hdd           53.6G  Unknown  N/A    N/A    Yes        
cn3       /dev/vdb  hdd           42.9G  Unknown  N/A    N/A    Yes        
cn3       /dev/vdc  hdd           53.6G  Unknown  N/A    N/A    Yes        
cn4       /dev/vdb  hdd           42.9G  Unknown  N/A    N/A    Yes        
cn4       /dev/vdc  hdd           53.6G  Unknown  N/A    N/A    Yes        

# une commande pour ajouter tous les disques libres, c'est pratique ;)
[ceph: root@cn1 /]# ceph orch apply osd --all-available-devices
Scheduled osd.all-available-devices update...

# surveillons les modifications du Cluster
[ceph: root@cn1 /]# ceph -w
  cluster:
    id:     d3232dac-aa23-11ec-b513-5254004e3b60
    health: HEALTH_WARN
            mons are allowing insecure global_id reclaim
            OSD count 0 < osd_pool_default_size 3
 
  services:
    mon: 3 daemons, quorum cn1,cn2,cn3 (age 7h)
    mgr: cn1.lgzsul(active, since 9h), standbys: cn2.nbvlgl
    osd: 1 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     
 

2022-03-23T06:37:52.279764+0000 mon.cn3 [INF] mon.cn3 calling monitor election
2022-03-23T06:37:52.296307+0000 mon.cn2 [INF] mon.cn2 calling monitor election
2022-03-23T06:37:57.324534+0000 mon.cn2 [INF] mon.cn2 is new leader, mons cn2,cn3 in quorum (ranks 1,2)
2022-03-23T06:37:57.349617+0000 mon.cn2 [WRN] Health check failed: 1/3 mons down, quorum cn2,cn3 (MON_DOWN)
2022-03-23T06:37:57.369295+0000 mon.cn2 [WRN] Health detail: HEALTH_WARN mons are allowing insecure global_id reclaim; 1/3 mons down, quorum cn2,cn3; OSD count 0 < osd_pool_default_size 3
2022-03-23T06:37:57.369354+0000 mon.cn2 [WRN] [WRN] AUTH_INSECURE_GLOBAL_ID_RECLAIM_ALLOWED: mons are allowing insecure global_id reclaim
2022-03-23T06:37:57.369363+0000 mon.cn2 [WRN]     mon.cn2 has auth_allow_insecure_global_id_reclaim set to true
2022-03-23T06:37:57.369406+0000 mon.cn2 [WRN]     mon.cn3 has auth_allow_insecure_global_id_reclaim set to true
2022-03-23T06:37:57.369428+0000 mon.cn2 [WRN] [WRN] MON_DOWN: 1/3 mons down, quorum cn2,cn3
2022-03-23T06:37:57.369435+0000 mon.cn2 [WRN]     mon.cn1 (rank 0) addr [v2:192.168.111.11:3300/0,v1:192.168.111.11:6789/0] is down (out of quorum)
2022-03-23T06:37:57.369441+0000 mon.cn2 [WRN] [WRN] TOO_FEW_OSDS: OSD count 0 < osd_pool_default_size 3
2022-03-23T06:37:57.410892+0000 mon.cn2 [WRN] Health check update: OSD count 1 < osd_pool_default_size 3 (TOO_FEW_OSDS)
2022-03-23T06:37:59.201999+0000 mon.cn2 [INF] Health check cleared: TOO_FEW_OSDS (was: OSD count 1 < osd_pool_default_size 3)
2022-03-23T06:37:55.541791+0000 mon.cn1 [INF] mon.cn1 calling monitor election
2022-03-23T06:38:00.353154+0000 mon.cn3 [INF] mon.cn3 calling monitor election
2022-03-23T06:38:00.356380+0000 mon.cn1 [INF] mon.cn1 calling monitor election
2022-03-23T06:38:00.356967+0000 mon.cn2 [INF] mon.cn2 calling monitor election
2022-03-23T06:38:00.367522+0000 mon.cn1 [INF] mon.cn1 is new leader, mons cn1,cn2,cn3 in quorum (ranks 0,1,2)
2022-03-23T06:38:00.393294+0000 mon.cn1 [INF] Health check cleared: MON_DOWN (was: 1/3 mons down, quorum cn2,cn3)
2022-03-23T06:38:00.408420+0000 mon.cn1 [WRN] Health detail: HEALTH_WARN mons are allowing insecure global_id reclaim
2022-03-23T06:38:00.408441+0000 mon.cn1 [WRN] [WRN] AUTH_INSECURE_GLOBAL_ID_RECLAIM_ALLOWED: mons are allowing insecure global_id reclaim
2022-03-23T06:38:00.408445+0000 mon.cn1 [WRN]     mon.cn1 has auth_allow_insecure_global_id_reclaim set to true
2022-03-23T06:38:00.408449+0000 mon.cn1 [WRN]     mon.cn2 has auth_allow_insecure_global_id_reclaim set to true
2022-03-23T06:38:00.408452+0000 mon.cn1 [WRN]     mon.cn3 has auth_allow_insecure_global_id_reclaim set to true
2022-03-23T06:38:09.529002+0000 mon.cn1 [INF] osd.0 [v2:192.168.111.13:6800/1230560976,v1:192.168.111.13:6801/1230560976] boot
2022-03-23T06:38:11.581373+0000 mon.cn1 [INF] osd.2 [v2:192.168.111.14:6800/2254200292,v1:192.168.111.14:6801/2254200292] boot
2022-03-23T06:38:12.607737+0000 mon.cn1 [INF] osd.4 [v2:192.168.111.13:6808/2412124976,v1:192.168.111.13:6809/2412124976] boot
2022-03-23T06:38:13.616896+0000 mon.cn1 [INF] osd.1 [v2:192.168.111.12:6800/2626008572,v1:192.168.111.12:6801/2626008572] boot
2022-03-23T06:38:14.243023+0000 mon.cn1 [INF] osd.5 [v2:192.168.111.14:6808/2854343695,v1:192.168.111.14:6809/2854343695] boot
2022-03-23T06:38:14.243044+0000 mon.cn1 [INF] osd.3 [v2:192.168.111.11:6802/4039541501,v1:192.168.111.11:6803/4039541501] boot
2022-03-23T06:38:15.288383+0000 mon.cn1 [INF] osd.6 [v2:192.168.111.12:6808/1556197905,v1:192.168.111.12:6809/1556197905] boot
2022-03-23T06:38:17.505732+0000 mon.cn1 [INF] osd.7 [v2:192.168.111.11:6810/262489380,v1:192.168.111.11:6811/262489380] boot
2022-03-23T06:40:00.000380+0000 mon.cn1 [WRN] overall HEALTH_WARN mons are allowing insecure global_id reclaim
2022-03-23T06:50:00.000508+0000 mon.cn1 [WRN] overall HEALTH_WARN mons are allowing insecure global_id reclaim

# les osd sont créé et intrégré dans le cluster.
# il reste un warning. il devrais être possible d'activer cette fonction plutot. 
[ceph: root@cn1 /]# ceph config set mon auth_allow_insecure_global_id_reclaim false

# vérification des disques utilisés par le cluster
[ceph: root@cn1 /]# ceph osd tree
ID  CLASS  WEIGHT   TYPE NAME      STATUS  REWEIGHT  PRI-AFF
-1         0.35156  root default                            
-9         0.08789      host cn1                            
 3    hdd  0.03909          osd.3      up   1.00000  1.00000
 7    hdd  0.04880          osd.7      up   1.00000  1.00000
-7         0.08789      host cn2                            
 2    hdd  0.03909          osd.2      up   1.00000  1.00000
 5    hdd  0.04880          osd.5      up   1.00000  1.00000
-3         0.08789      host cn3                            
 1    hdd  0.03909          osd.1      up   1.00000  1.00000
 6    hdd  0.04880          osd.6      up   1.00000  1.00000
-5         0.08789      host cn4                            
 0    hdd  0.03909          osd.0      up   1.00000  1.00000
 4    hdd  0.04880          osd.4      up   1.00000  1.00000

# vérifions le status du cluster. C'est ok, on a bien 8 osds dans le cluster avec 4 nodes.
[ceph: root@cn1 /]# ceph -w
  cluster:
    id:     d3232dac-aa23-11ec-b513-5254004e3b60
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum cn1,cn2,cn3 (age 22m)
    mgr: cn1.lgzsul(active, since 9h), standbys: cn2.nbvlgl
    osd: 8 osds: 8 up (since 21m), 8 in (since 21m)
 
  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   8.0 GiB used, 352 GiB / 360 GiB avail
    pgs:     1 active+clean
 

# Remarque : Le cluster est maintenant ok, et prêt pour la configuration des pools
```
## connexion au dashboard du cluster Ceph
retrouvez vos informations de connexions fournies lors de l'initialisation du cluster , et utilisez un navigateur pour vous connecter à l'adresse https://192.168.0.11:8443
saisissez le compte admin et le mot de passe associé. Il faudra changer le mot de passe par défaut à la première connexion.
Voici une copie d'écran du dashboard de ceph octopus
![ceph-dashboard](ceph-dashboard.png)


