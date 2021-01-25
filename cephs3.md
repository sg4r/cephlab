# Stockage S3 / Swift
Ceph Rados Gateway est une interface de stockage d'objets construite sur la librados pour fournir aux applications une passerelle RESTful vers les clusters de stockage Ceph.
Ceph Rados Gateway prend en charge deux interfaces:
 - S3: fournit un stockage compatible avec l'API RESTful Amazon S3.
 - Swift: fournit un stockage compatible avec l'API OpenStack Swift.

![cephrgw.png](cephrgw.png)
## Topologie réseau
La passerelle est accessible depuis l’internet, donc installer RadosGW sur un serveur dédié dans votre DMZ. Il est
possible de répartir la charge avec un HAPROXY. Les flux librados seront autorisés vers le réseau publique votre de cluster Ceph.
![cephradosgw](cephradosgw.png)
source https://qiita.com/jundo414/items/7ac3bdf3967ec67d7680

## Installation de RadosGW
Le service RadosGW s’installe généralement sur un nœud dédié. Prévoir un CPU à 8 coeurs et entre 32Go à 64Go de
RAM en fonction du nombre de clients. Pour équilibrer la charge, il est conseillé d’utiliser plusieurs RadosGW avec un
proxyHA et comptez une RadosGW pour 50-100 OSDs.

Dans le cadre de ce Labs, il existe une vm ```cmrw``` dédié a l'installation du service RadosGW. Les premières étapes vont d'intégrer ce noeud au cluster Ceph, puis de définir les pools et les comptes utiliser S3 pour l'acces aux données. 
Remarque : les utilisateurs  S3 sont connu uniquement de la passerelle et non du Cluster Ceph.

```
# Préparer le node cnrw
[vagrant@cn1 ~]$ ssh-copy-id -f -i /etc/ceph/ceph.pub root@cnrw
[vagrant@cn1 ~]$ ssh root@cnrw dnf install -y podman
[vagrant@cn1 ~]$ ssh root@cnrw systemctl stop firewalld.service
[vagrant@cn1 ~]$ ssh root@cnrw systemctl disable firewalld.service

# Ajouter au cluster Ceph
[ceph: root@cn1 /]# ceph orch host add cnrw
Added host 'cnrw'
[ceph: root@cn1 /]# ceph orch host ls
HOST  ADDR  LABELS  STATUS  
cn1   cn1                   
cn2   cn2                   
cn3   cn3                   
cn4   cn4                   
cnrw  cnrw                  
[ceph: root@cn1 /]# ceph orch ls
NAME                       RUNNING  REFRESHED  AGE  PLACEMENT  IMAGE NAME                            IMAGE ID      
alertmanager                   1/1  45s ago    9d   count:1    docker.io/prom/alertmanager:v0.20.0   0881eb8f169f  
crash                          5/5  48s ago    9d   *          docker.io/ceph/ceph:v15               5553b0cb212c  
grafana                        1/1  45s ago    9d   count:1    docker.io/ceph/ceph-grafana:6.6.2     a0dce381714a  
mds.moncfs                     2/2  47s ago    6d   count:2    docker.io/ceph/ceph:v15               5553b0cb212c  
mgr                            2/2  47s ago    9d   count:2    docker.io/ceph/ceph:v15               5553b0cb212c  
mon                            5/5  48s ago    9d   count:5    docker.io/ceph/ceph:v15               5553b0cb212c  
node-exporter                  5/5  48s ago    9d   *          docker.io/prom/node-exporter:v0.18.1  e5a616e4b9cf  
osd.all-available-devices      8/8  47s ago    9d   *          docker.io/ceph/ceph:v15               5553b0cb212c  
prometheus                     1/1  45s ago    9d   count:1    docker.io/prom/prometheus:v2.18.1     de242295e225  
 
# Remarque :  Augmentation du nombre de mon de 3/5 à 5/5

[ceph: root@cn1 /]# ceph -s
  cluster:
    id:     2e90db8c-541a-11eb-bb6e-525400ae1f18
    health: HEALTH_OK
 
  services:
    mon: 5 daemons, quorum cn1,cn3,cn2,cnrw,cn4 (age 26s)
    mgr: cn2.rkgnmp(active, since 37m), standbys: cn1.pnzyvw
    mds: moncfs:1 {0=moncfs.cn1.mbgihz=up:active} 1 up:standby
    osd: 8 osds: 8 up (since 37m), 8 in (since 6d)
 
  data:
    pools:   5 pools, 113 pgs
    objects: 65 objects, 16 MiB
    usage:   8.3 GiB used, 352 GiB / 360 GiB avail
    pgs:     113 active+clean

# création des domaines S3
[ceph: root@cn1 /]# radosgw-admin realm create --rgw-realm=default --default
{
    "id": "0b3fa5d4-7836-42bc-8e0f-42cd32f90f25",
    "name": "default",
    "current_period": "f38d211a-8a55-462a-9f07-b9ded0ac40d6",
    "epoch": 1
}
[ceph: root@cn1 /]# radosgw-admin zonegroup create --rgw-zonegroup=default --master --default
{
    "id": "99121b91-66c4-4c4a-a37b-c1dd4a5938b0",
    "name": "default",
    "api_name": "default",
    "is_master": "true",
    "endpoints": [],
    "hostnames": [],
    "hostnames_s3website": [],
    "master_zone": "",
    "zones": [],
    "placement_targets": [],
    "default_placement": "",
    "realm_id": "0b3fa5d4-7836-42bc-8e0f-42cd32f90f25",
    "sync_policy": {
        "groups": []
    }
}
[ceph: root@cn1 /]# radosgw-admin zone create --rgw-zonegroup=default --rgw-zone=us-east-1 --master --default
{
    "id": "aafa8fde-667c-4c24-beb8-0f0a78286026",
    "name": "us-east-1",
    "domain_root": "us-east-1.rgw.meta:root",
    "control_pool": "us-east-1.rgw.control",
    "gc_pool": "us-east-1.rgw.log:gc",
    "lc_pool": "us-east-1.rgw.log:lc",
    "log_pool": "us-east-1.rgw.log",
    "intent_log_pool": "us-east-1.rgw.log:intent",
    "usage_log_pool": "us-east-1.rgw.log:usage",
    "roles_pool": "us-east-1.rgw.meta:roles",
    "reshard_pool": "us-east-1.rgw.log:reshard",
    "user_keys_pool": "us-east-1.rgw.meta:users.keys",
    "user_email_pool": "us-east-1.rgw.meta:users.email",
    "user_swift_pool": "us-east-1.rgw.meta:users.swift",
    "user_uid_pool": "us-east-1.rgw.meta:users.uid",
    "otp_pool": "us-east-1.rgw.otp",
    "system_key": {
        "access_key": "",
        "secret_key": ""
    },
    "placement_pools": [
        {
            "key": "default-placement",
            "val": {
                "index_pool": "us-east-1.rgw.buckets.index",
                "storage_classes": {
                    "STANDARD": {
                        "data_pool": "us-east-1.rgw.buckets.data"
                    }
                },
                "data_extra_pool": "us-east-1.rgw.buckets.non-ec",
                "index_type": 0
            }
        }
    ],
    "realm_id": "0b3fa5d4-7836-42bc-8e0f-42cd32f90f25"
}

# Installation de la RadosGW sur le noeud cnrw
[ceph: root@cn1 /]# ceph orch apply rgw default us-east-1 --placement="1 cnrw"
Scheduled rgw.default.us-east-1 update...
[ceph: root@cn1 /]# 
 
[ceph: root@cn1 /]# ceph -s
  cluster:
    id:     2e90db8c-541a-11eb-bb6e-525400ae1f18
    health: HEALTH_OK
 
  services:
    mon: 5 daemons, quorum cn1,cn3,cn2,cnrw,cn4 (age 33m)
    mgr: cn2.rkgnmp(active, since 70m), standbys: cn1.pnzyvw
    mds: moncfs:1 {0=moncfs.cn1.mbgihz=up:active} 1 up:standby
    osd: 8 osds: 8 up (since 70m), 8 in (since 6d)
    rgw: 1 daemon active (default.us-east-1.cnrw.mnhtaj)
 
  task status:
 
  data:
    pools:   9 pools, 224 pgs
    objects: 261 objects, 16 MiB
    usage:   8.4 GiB used, 352 GiB / 360 GiB avail
    pgs:     0.446% pgs not active
             223 active+clean
             1   peering
 
  progress:
    PG autoscaler decreasing pool 9 PGs from 32 to 8 (60s)
      [===========.................] (remaining: 84s)

# remarque: on constate qu'il y a 4 nouveaux pools avec le passage de 5 a 9 pools.
# remarque: on remarque l'ajout de l'instance rwg dans le cluster
# remarque: l'autoscaler va diminuer le nombre de PGs pour le pool us-east-1.rgw.meta automatiquement
[ceph: root@cn1 /]# ceph df
--- RAW STORAGE ---
CLASS  SIZE     AVAIL    USED     RAW USED  %RAW USED
hdd    360 GiB  352 GiB  455 MiB   8.4 GiB       2.35
TOTAL  360 GiB  352 GiB  455 MiB   8.4 GiB       2.35
 
--- POOLS ---
POOL                   ID  PGS  STORED   OBJECTS  USED     %USED  MAX AVAIL
device_health_metrics   1    1      0 B        0      0 B      0    111 GiB
prbd                    2   16  4.4 MiB       25   16 MiB      0    111 GiB
prbdec                  3   32  7.7 MiB       18   25 MiB      0    166 GiB
cephfs.moncfs.meta      4   32  143 KiB       22  1.9 MiB      0    111 GiB
cephfs.moncfs.data      5   32      0 B        0      0 B      0    111 GiB
.rgw.root               6   32  2.0 KiB       13  2.2 MiB      0    111 GiB
us-east-1.rgw.log       7   32  3.4 KiB      175    6 MiB      0    111 GiB
us-east-1.rgw.control   8   32      0 B        8      0 B      0    111 GiB
us-east-1.rgw.meta      9    8      0 B        0      0 B      0    111 GiB

# remarque : les nouveaux pools sont : .rgw.root, us-east-1.rgw.log, us-east-1.rgw.control, us-east-1.rgw.meta

# Création d'un utilisateur testuser pour l'acces via S3. La gestion est fait avec l'outil radosgw-admin et non ceph
[ceph: root@cn1 /]# radosgw-admin user create --uid="testuser" --display-name="testuser First User" --access-key=testuserkacc --secret-key=testuserkpwd
{
    "user_id": "testuser",
    "display_name": "testuser First User",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "subusers": [],
    "keys": [
        {
            "user": "testuser",
            "access_key": "testuserkacc",
            "secret_key": "testuserkpwd"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "default_storage_class": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
}

# Quittez le contenaire pour tester le service S3
[ceph: root@cn1 /]# exit
exit

[vagrant@cn1 ~]$ 
[vagrant@cn1 ~]$ curl http://cnrw |xmllint --format -
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   214    0   214    0     0   8560      0 --:--:-- --:--:-- --:--:--  8560
<?xml version="1.0" encoding="UTF-8"?>
<ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
  <Owner>
    <ID>anonymous</ID>
    <DisplayName/>
  </Owner>
  <Buckets/>
</ListAllMyBucketsResult>
[vagrant@cn1 ~]$ 

# Remarque : RGW est bien accessible depuis le port 80 de cnrw

[vagrant@cn1 ~]$ exit
logout
Connection to 192.168.121.217 closed.

# connexion sur cephclt qui n'est pas dans le cluster Ceph
sg4r@work:~/dev/ceph-octopus$ vagrant ssh cephclt
Last login: Wed Jan 20 17:07:08 2021 from 192.168.121.1
[vagrant@cephclt ~]$ 
[vagrant@cephclt ~]$ sudo yum install awscli
#Configuration de aws pour l'acces au service RadosGW avec l'utilisateur créé précédament
[vagrant@cephclt ~]$ aws configure
AWS Access Key ID [None]: testuserkacc
AWS Secret Access Key [None]: testuserkpwd
Default region name [None]: us-east-1
Default output format [None]: json

# Création d'un bucket S3. il doit être unique.
[vagrant@cephclt ~]$ aws s3 mb s3://my-new-bucket --endpoint-url http://cnrw
make_bucket: my-new-bucket
# Quelques exemple de manipulation de fichiers
[vagrant@cephclt ~]$ aws s3 cp /etc/hosts s3://my-new-bucket --endpoint-url http://cnrw
upload: ../../etc/hosts to s3://my-new-bucket/hosts                 
[vagrant@cephclt ~]$ aws s3 ls s3://my-new-bucket --endpoint-url http://cnrw
2021-01-21 09:11:06        323 hosts

[vagrant@cephclt ~]$ mkdir /tmp/f2sync
[vagrant@cephclt ~]$ echo fichier1 >>/tmp/f2sync/fichier1.txt
[vagrant@cephclt ~]$ echo fichier2 >>/tmp/f2sync/fichier2.txt
[vagrant@cephclt ~]$ mkdir /tmp/f2sync/f2
[vagrant@cephclt ~]$ echo fichier3 >>/tmp/f2sync/f2/fichier3.txt
[vagrant@cephclt ~]$ aws s3 sync /tmp/f2sync  s3://my-new-bucket/ --endpoint-url http://cnrw --delete
delete: s3://my-new-bucket/hosts
upload: ../../tmp/f2sync/fichier1.txt to s3://my-new-bucket/fichier1.txt
upload: ../../tmp/f2sync/f2/fichier3.txt to s3://my-new-bucket/f2/fichier3.txt
upload: ../../tmp/f2sync/fichier2.txt to s3://my-new-bucket/fichier2.txt

[vagrant@cephclt ~]$ aws s3 ls s3://my-new-bucket --endpoint-url http://cnrw --recursive
2021-01-21 10:05:45          9 f2/fichier3.txt
2021-01-21 10:05:45          9 fichier1.txt
2021-01-21 10:05:45          9 fichier2.txt

# et voila pour une première installation de RGW pour fournir un accès S3 depuis votre cluster Ceph.
```
## RGW DATA CACHING AND CDN
Cette fonctionnalité disponible depuis la version Octopus, ajoute à RGW la possibilité de mettre en cache des objets en toute sécurité et de décharger la charge de travail du cluster, à l'aide de Nginx. Après avoir accédé à un objet la première fois, il sera stocké dans le répertoire de cache Nginx. Lorsque les données sont déjà mises en cache, elles n'ont pas besoin d'être extraites de RGW. Une vérification d'autorisation sera effectuée par rapport à RGW pour s'assurer que l'utilisateur demandeur a accès. Cette fonctionnalité est basée sur des modules Nginx, ngx_http_auth_request_module, https://github.com/kaltura/nginx-aws-auth-module et Openresty.

Pour plus d'information voir https://docs.ceph.com/en/latest/radosgw/rgw-cache/
## Documentation
Pour plus d’information voir https://docs.ceph.com/en/latest/radosgw/

