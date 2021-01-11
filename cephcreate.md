## Installtion du clusteur Ceph avec cephadm
```
[vagrant@cn1 ~]$ curl --silent --remote-name --location https://raw.githubusercontent.com/ceph/ceph/octopus/src/cephadm/cephadm
[vagrant@cn1 ~]$ chmod +x cephadm
[vagrant@cn1 ~]$ sudo mkdir -p /etc/ceph
[vagrant@cn1 ~]$ sudo ./cephadm bootstrap --mon-ip 192.168.0.11
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
Cluster fsid: c9eacd34-513d-11eb-9233-5254009e5678
Verifying IP 192.168.0.11 port 3300 ...
Verifying IP 192.168.0.11 port 6789 ...
Mon IP 192.168.0.11 is in CIDR network 192.168.0.0/24
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
	Password: byxc7x3leo

You can access the Ceph CLI with:

	sudo ./cephadm shell --fsid 2e90db8c-541a-11eb-bb6e-525400ae1f18 -c /etc/ceph/ceph.conf -k /etc/ceph/ceph.client.admin.keyring

Please consider enabling telemetry to help improve Ceph:

	ceph telemetry on

For more information see:

	https://docs.ceph.com/docs/master/mgr/telemetry/

Bootstrap complete.

# Votre cluster Ceph est créé. Noté bien la valeur du Password par defaut. il serra utilisé plus tard pour accéder au dashboard.
# 

# Ajout ssh ceph.pub key sur les nodes 2 à 4
[vagrant@cn1 ~]$ for i in {2..4}; do ssh-copy-id -f -i /etc/ceph/ceph.pub root@cn$i; done

# Connexion au cluster ceph
[vagrant@cn1 ~]$ sudo ./cephadm shell 
Inferring fsid 2e90db8c-541a-11eb-bb6e-525400ae1f18
Inferring config /var/lib/ceph/2e90db8c-541a-11eb-bb6e-525400ae1f18/mon.cn1/config
Using recent ceph image docker.io/ceph/ceph:v15

# Remarque  : on passe dans le contenaire. Attention au changement de shell en [ceph: root@cn1 /]

[ceph: root@cn1 /]#  ceph -s
  cluster:
    id:     2e90db8c-541a-11eb-bb6e-525400ae1f18
    health: HEALTH_WARN
            OSD count 0 < osd_pool_default_size 3
 
  services:
    mon: 1 daemons, quorum cn1 (age 5m)
    mgr: cn1.pnzyvw(active, since 5m)
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:  
    
# Remarque : Le clusteur est en warring au début, car il n'y a pas encore d'osd ou plusieurs mon, mais on va règler cela rapidement

[ceph: root@cn1 /]# cat /etc/redhat-release 
CentOS Linux release 8.3.2011
# Remarque : l'OS du contenaire est basé sur CentOS Linux release 8.3.2011

# Ajouter les nodes supplémentaire au cluster Ceph
[ceph: root@cn1 /]# ceph orch host add cn2
Added host 'cn2'
[ceph: root@cn1 /]# ceph orch host add cn3
Added host 'cn3'
[ceph: root@cn1 /]# ceph orch host add cn4
Added host 'cn4'

# liste des nodes dans le clusteur. On retrouve bien les 4 nodes
[ceph: root@cn1 /]# ceph orch host ls
HOST  ADDR  LABELS  STATUS  
cn1   cn1                   
cn2   cn2                   
cn3   cn3                   
cn4   cn4                   

# l'ajout des nodes va augmenter automatiquement le nombre de process dans le clusteur.
[ceph: root@cn1 /]# ceph orch ls
NAME           RUNNING  REFRESHED  AGE  PLACEMENT  IMAGE NAME                            IMAGE ID      
alertmanager       1/1  51s ago    7m   count:1    docker.io/prom/alertmanager:v0.20.0   0881eb8f169f  
crash              1/4  51s ago    8m   *          docker.io/ceph/ceph:v15               5553b0cb212c  
grafana            1/1  51s ago    7m   count:1    docker.io/ceph/ceph-grafana:6.6.2     a0dce381714a  
mgr                1/2  51s ago    8m   count:2    docker.io/ceph/ceph:v15               5553b0cb212c  
mon                1/5  51s ago    8m   count:5    docker.io/ceph/ceph:v15               5553b0cb212c  
node-exporter      1/4  51s ago    7m   *          docker.io/prom/node-exporter:v0.18.1  e5a616e4b9cf  
prometheus         1/1  51s ago    7m   count:1    docker.io/prom/prometheus:v2.18.1     de242295e225  

# Surveillons les modifications au niveau du Cluster 
[ceph: root@cn1 /]# ceph -w
  cluster:
    id:     2e90db8c-541a-11eb-bb6e-525400ae1f18
    health: HEALTH_WARN
            failed to probe daemons or devices
            OSD count 0 < osd_pool_default_size 3
 
  services:
    mon: 1 daemons, quorum cn1 (age 10m)
    mgr: cn1.pnzyvw(active, since 10m), standbys: cn2.rkgnmp
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     
 

2021-01-11T14:49:03.762707+0000 mon.cn1 [INF] mon.cn1 calling monitor election
2021-01-11T14:49:05.680296+0000 mon.cn3 [INF] mon.cn3 calling monitor election
2021-01-11T14:49:08.820774+0000 mon.cn1 [INF] mon.cn1 is new leader, mons cn1,cn3 in quorum (ranks 0,1)
2021-01-11T14:49:08.897971+0000 mon.cn1 [WRN] Health detail: HEALTH_WARN failed to probe daemons or devices; OSD count 0 < osd_pool_default_size 3
2021-01-11T14:49:08.898046+0000 mon.cn1 [WRN] [WRN] CEPHADM_REFRESH_FAILED: failed to probe daemons or devices
2021-01-11T14:49:08.898062+0000 mon.cn1 [WRN] [WRN] TOO_FEW_OSDS: OSD count 0 < osd_pool_default_size 3
2021-01-11T14:49:09.118427+0000 mon.cn1 [INF] mon.cn1 calling monitor election
2021-01-11T14:49:09.140783+0000 mon.cn3 [INF] mon.cn3 calling monitor election
2021-01-11T14:49:11.104706+0000 mon.cn2 [INF] mon.cn2 calling monitor election
2021-01-11T14:49:14.161148+0000 mon.cn1 [INF] mon.cn1 is new leader, mons cn1,cn3,cn2 in quorum (ranks 0,1,2)
2021-01-11T14:49:14.234202+0000 mon.cn1 [WRN] Health detail: HEALTH_WARN failed to probe daemons or devices; OSD count 0 < osd_pool_default_size 3
2021-01-11T14:49:14.234224+0000 mon.cn1 [WRN] [WRN] CEPHADM_REFRESH_FAILED: failed to probe daemons or devices
2021-01-11T14:49:14.234229+0000 mon.cn1 [WRN] [WRN] TOO_FEW_OSDS: OSD count 0 < osd_pool_default_size 3
2021-01-11T14:49:30.890447+0000 mon.cn1 [INF] Health check cleared: CEPHADM_REFRESH_FAILED (was: failed to probe daemons or devices)
2021-01-11T14:50:00.004693+0000 mon.cn1 [WRN] Health detail: HEALTH_WARN OSD count 0 < osd_pool_default_size 3
2021-01-11T14:50:00.004792+0000 mon.cn1 [WRN] [WRN] TOO_FEW_OSDS: OSD count 0 < osd_pool_default_size 3


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

# le clusteur Ceph est toujours en warring, mais il a maintenant 3 moniteurs et 2 mgr
[ceph: root@cn1 /]# ceph -s
  cluster:
    id:     2e90db8c-541a-11eb-bb6e-525400ae1f18
    health: HEALTH_WARN
            OSD count 0 < osd_pool_default_size 3
 
  services:
    mon: 3 daemons, quorum cn1,cn3,cn2 (age 2m)
    mgr: cn1.pnzyvw(active, since 14m), standbys: cn2.rkgnmp
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     
         
# ajoutons des disques au clusteur, mais avant cela on peux lister l'ensemble des disques libres éxistant
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
    id:     2e90db8c-541a-11eb-bb6e-525400ae1f18
    health: HEALTH_WARN
            failed to probe daemons or devices
 
  services:
    mon: 3 daemons, quorum cn1,cn3,cn2 (age 6m)
    mgr: cn1.pnzyvw(active, since 18m), standbys: cn2.rkgnmp
    osd: 8 osds: 0 up, 0 in
 
  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     100.000% pgs unknown
             1 unknown
 

2021-01-11T14:55:42.462132+0000 mon.cn1 [INF] osd.1 [v2:192.168.0.13:6800/3357115903,v1:192.168.0.13:6801/3357115903] boot
2021-01-11T14:55:42.462189+0000 mon.cn1 [INF] osd.0 [v2:192.168.0.14:6800/3301640803,v1:192.168.0.14:6801/3301640803] boot
2021-01-11T14:55:43.489327+0000 mon.cn1 [INF] osd.2 [v2:192.168.0.12:6800/1513505846,v1:192.168.0.12:6801/1513505846] boot
2021-01-11T14:55:44.516120+0000 mon.cn1 [INF] osd.3 [v2:192.168.0.11:6802/475127529,v1:192.168.0.11:6803/475127529] boot
2021-01-11T14:55:45.552359+0000 mon.cn1 [INF] osd.4 [v2:192.168.0.14:6808/2752264528,v1:192.168.0.14:6809/2752264528] boot
2021-01-11T14:55:46.573267+0000 mon.cn1 [INF] osd.6 [v2:192.168.0.13:6808/3030442383,v1:192.168.0.13:6809/3030442383] boot
2021-01-11T14:55:47.608346+0000 mon.cn1 [INF] osd.5 [v2:192.168.0.12:6808/3117922813,v1:192.168.0.12:6809/3117922813] boot
2021-01-11T14:55:48.695563+0000 mon.cn1 [INF] osd.7 [v2:192.168.0.11:6810/4205497031,v1:192.168.0.11:6811/4205497031] boot
2021-01-11T14:55:54.989466+0000 mon.cn1 [INF] Health check cleared: CEPHADM_REFRESH_FAILED (was: failed to probe daemons or devices)
2021-01-11T14:55:54.989528+0000 mon.cn1 [INF] Cluster is now healthy


# vérification des disques utilisé par le clusteur
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

# vérifiont le status du cluster. c'est ok, on a bien 8 osds dans le cluster avec 4 nodes.
[ceph: root@cn1 /]# ceph -s
  cluster:
    id:     2e90db8c-541a-11eb-bb6e-525400ae1f18
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum cn1,cn3,cn2 (age 7m)
    mgr: cn1.pnzyvw(active, since 19m), standbys: cn2.rkgnmp
    osd: 8 osds: 8 up (since 71s), 8 in (since 71s)
 
  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   8.0 GiB used, 352 GiB / 360 GiB avail
    pgs:     1 active+clean

# Remarque : Le cluster est maintenant ok, et prets pour la configuration des pools
```
## connexion au dashbord du cluster Ceph
retrouvez vos informations de connexoins fournit lors de l'initialisation du cluster , et utilisez un navigateur pour vous connecter a l'adresse https://192.168.0.11:8443
saisir le compte admin et le mot de passe associer. il faudra changer le mot de passe par defaut a la premiere connexion.
Voici une copie d'ecran du dashboard de ceph octopus
![ceph-dashboard](ceph-dashboard.png)


