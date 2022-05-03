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

Dans le cadre de ce Labs, il existe deux vms ```cmrgw1 cmrgw2``` dédiée à l'installation du service RadosGW. Les premières étapes vont d'intégrer ces noeuds au cluster Ceph, puis de définir les pools et les comptes utiliser S3 pour l'acces aux données. 
Remarque : les utilisateurs S3 sont connu uniquement de la passerelle et non du Cluster Ceph.

```
# Préparer le node cnrgw1 et cnrgw2
[vagrant@cn1 ~]$ ssh-copy-id -f -i /etc/ceph/ceph.pub root@cnrgw1
[vagrant@cn1 ~]$ ssh root@cnrgw1 dnf install -y podman
[vagrant@cn1 ~]$ ssh root@cnrgw1 systemctl stop firewalld.service
[vagrant@cn1 ~]$ ssh root@cnrgw1 systemctl disable firewalld.service
[vagrant@cn1 ~]$ ssh-copy-id -f -i /etc/ceph/ceph.pub root@cnrgw2
[vagrant@cn1 ~]$ ssh root@cnrgw2 dnf install -y podman
[vagrant@cn1 ~]$ ssh root@cnrgw2 systemctl stop firewalld.service
[vagrant@cn1 ~]$ ssh root@cnrgw2 systemctl disable firewalld.service


# Passer dans le contenaire
[vagrant@cn1 ~]$ sudo ./cephadm shell

# Ajouter au cluster Ceph
[ceph: root@cn1 /]# ceph orch host add cnrgw1
Added host 'cnrgw1'
[ceph: root@cn1 /]# ceph orch host add cnrgw2
Added host 'cnrgw2'
[ceph: root@cn1 /]# ceph orch host ls
HOST  ADDR  LABELS  STATUS  
cn1   cn1                   
cn2   cn2                   
cn3   cn3                   
cn4   cn4                   
cnrgw1  cnrgw1
cnrgw2  cnrgw2
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
    mon: 5 daemons, quorum cn1,cn3,cn2,cnrgw1,cn4 (age 26s)
    mgr: cn2.rkgnmp(active, since 37m), standbys: cn1.pnzyvw
    mds: moncfs:1 {0=moncfs.cn1.mbgihz=up:active} 1 up:standby
    osd: 8 osds: 8 up (since 37m), 8 in (since 6d)
 
  data:
    pools:   5 pools, 113 pgs
    objects: 65 objects, 16 MiB
    usage:   8.3 GiB used, 352 GiB / 360 GiB avail
    pgs:     113 active+clean

# création des domaines S3

[ceph: root@cn1 /]# RGW_REALM=demodom
[ceph: root@cn1 /]# RGW_MASTERZONEGRP=fr
[ceph: root@cn1 /]# RGW_MASTERZONE=fr-est-1
[ceph: root@cn1 /]# SYSTEM_ACCESS_KEY=DemoZoneSyncKeyAcc
[ceph: root@cn1 /]# SYSTEM_SECRET_KEY=DemoZoneSyncKeyMegaSecret
[ceph: root@cn1 /]# 
[ceph: root@cn1 /]# # hostname de la premiere rgw fr-est-1
[ceph: root@cn1 /]# RADOSGW=cnrgw1
[ceph: root@cn1 /]# radosgw-admin realm create --rgw-realm=$RGW_REALM --default
{
    "id": "6136dc56-ee0b-4c00-be95-feefee061d1d",
    "name": "demodom",
    "current_period": "b5ea9223-2ae5-4fc0-bfd8-318001256176",
    "epoch": 1
}
[ceph: root@cn1 /]# radosgw-admin zonegroup delete --rgw-zonegroup=default
failed to init zonegroup: (2) No such file or directory

# pas de configuration initial
[ceph: root@cn1 /]# radosgw-admin zonegroup create --rgw-zonegroup=$RGW_MASTERZONEGRP  --endpoints=http://$RADOSGW:80 --master --default
{
    "id": "4c959d20-3520-4c90-b026-d707c9f9332d",
    "name": "fr",
    "api_name": "fr",
    "is_master": "true",
    "endpoints": [
        "http://cnrgw1:80"
    ],
    "hostnames": [],
    "hostnames_s3website": [],
    "master_zone": "",
    "zones": [],
    "placement_targets": [],
    "default_placement": "",
    "realm_id": "6136dc56-ee0b-4c00-be95-feefee061d1d",
    "sync_policy": {
        "groups": []
    }
}

[ceph: root@cn1 /]# radosgw-admin zone create --rgw-zonegroup=$RGW_MASTERZONEGRP --rgw-zone=$RGW_MASTERZONE \
>  --endpoints=http://$RADOSGW:80 --access-key=$SYSTEM_ACCESS_KEY \
>  --secret=$SYSTEM_SECRET_KEY --default --master
{
    "id": "0238b991-54b6-4dee-8757-5579990fef9d",
    "name": "fr-est-1",
    "domain_root": "fr-est-1.rgw.meta:root",
    "control_pool": "fr-est-1.rgw.control",
    "gc_pool": "fr-est-1.rgw.log:gc",
    "lc_pool": "fr-est-1.rgw.log:lc",
    "log_pool": "fr-est-1.rgw.log",
    "intent_log_pool": "fr-est-1.rgw.log:intent",
    "usage_log_pool": "fr-est-1.rgw.log:usage",
    "roles_pool": "fr-est-1.rgw.meta:roles",
    "reshard_pool": "fr-est-1.rgw.log:reshard",
    "user_keys_pool": "fr-est-1.rgw.meta:users.keys",
    "user_email_pool": "fr-est-1.rgw.meta:users.email",
    "user_swift_pool": "fr-est-1.rgw.meta:users.swift",
    "user_uid_pool": "fr-est-1.rgw.meta:users.uid",
    "otp_pool": "fr-est-1.rgw.otp",
    "system_key": {
        "access_key": "DemoZoneSyncKeyAcc",
        "secret_key": "DemoZoneSyncKeyMegaSecret"
    },
    "placement_pools": [
        {
            "key": "default-placement",
            "val": {
                "index_pool": "fr-est-1.rgw.buckets.index",
                "storage_classes": {
                    "STANDARD": {
                        "data_pool": "fr-est-1.rgw.buckets.data"
                    }
                },
                "data_extra_pool": "fr-est-1.rgw.buckets.non-ec",
                "index_type": 0
            }
        }
    ],
    "realm_id": "6136dc56-ee0b-4c00-be95-feefee061d1d"
}
[ceph: root@cn1 /]# # remarque le nom de la zone est $RGW_MASTERZONE

[ceph: root@cn1 /]# radosgw-admin user create --uid="${RGW_MASTERZONEGRP}_zone.user" --display-name="${RGW_MASTERZONEGRP}_ZoneUser" \
> --access-key=$SYSTEM_ACCESS_KEY --secret=$SYSTEM_SECRET_KEY --system
{
    "user_id": "fr_zone.user",
    "display_name": "fr_ZoneUser",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "subusers": [],
    "keys": [
        {
            "user": "fr_zone.user",
            "access_key": "DemoZoneSyncKeyAcc",
            "secret_key": "DemoZoneSyncKeyMegaSecret"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "system": "true",
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

[ceph: root@cn1 /]# 
[ceph: root@cn1 /]# # valider les modifications et incrémanter la PERIODE
[ceph: root@cn1 /]# radosgw-admin period update --commit
{
    "id": "8f0acd3b-6b23-4807-8ae4-543b0957c5ab",
    "epoch": 1,
    "predecessor_uuid": "b5ea9223-2ae5-4fc0-bfd8-318001256176",
    "sync_status": [],
    "period_map": {
        "id": "8f0acd3b-6b23-4807-8ae4-543b0957c5ab",
        "zonegroups": [
            {
                "id": "4c959d20-3520-4c90-b026-d707c9f9332d",
                "name": "fr",
                "api_name": "fr",
                "is_master": "true",
                "endpoints": [
                    "http://cnrgw1:80"
                ],
                "hostnames": [],
                "hostnames_s3website": [],
                "master_zone": "0238b991-54b6-4dee-8757-5579990fef9d",
                "zones": [
                    {
                        "id": "0238b991-54b6-4dee-8757-5579990fef9d",
                        "name": "fr-est-1",
                        "endpoints": [
                            "http://cnrgw1:80"
                        ],
                        "log_meta": "false",
                        "log_data": "false",
                        "bucket_index_max_shards": 11,
                        "read_only": "false",
                        "tier_type": "",
                        "sync_from_all": "true",
                        "sync_from": [],
                        "redirect_zone": ""
                    }
                ],
                "placement_targets": [
                    {
                        "name": "default-placement",
                        "tags": [],
                        "storage_classes": [
                            "STANDARD"
                        ]
                    }
                ],
                "default_placement": "default-placement",
                "realm_id": "6136dc56-ee0b-4c00-be95-feefee061d1d",
                "sync_policy": {
                    "groups": []
                }
            }
        ],
        "short_zone_ids": [
            {
                "key": "0238b991-54b6-4dee-8757-5579990fef9d",
                "val": 2434533641
            }
        ]
    },
    "master_zonegroup": "4c959d20-3520-4c90-b026-d707c9f9332d",
    "master_zone": "0238b991-54b6-4dee-8757-5579990fef9d",
    "period_config": {
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
        }
    },
    "realm_id": "6136dc56-ee0b-4c00-be95-feefee061d1d",
    "realm_name": "demodom",
    "realm_epoch": 2
}
[ceph: root@cn1 /]# 

[ceph: root@cn1 /]# # déployer la rgw
[ceph: root@cn1 /]# ceph orch apply rgw $RGW_REALM $RGW_MASTERZONE --placement="1 $RADOSGW"
Scheduled rgw.demodom.fr-est-1 update...
[ceph: root@cn1 /]# 
[ceph: root@cn1 /]# 
[ceph: root@cn1 /]# ceph orch ls
NAME                       RUNNING  REFRESHED  AGE  PLACEMENT       IMAGE NAME                            IMAGE ID      
alertmanager                   1/1  5s ago     3w   count:1         docker.io/prom/alertmanager:v0.20.0   0881eb8f169f  
crash                          6/6  6m ago     3w   *               docker.io/ceph/ceph:v15               5553b0cb212c  
grafana                        1/1  5s ago     3w   count:1         docker.io/ceph/ceph-grafana:6.6.2     a0dce381714a  
mgr                            2/2  5m ago     3w   count:2         docker.io/ceph/ceph:v15               5553b0cb212c  
mon                            5/5  6m ago     3w   count:5         docker.io/ceph/ceph:v15               5553b0cb212c  
node-exporter                  6/6  6m ago     3w   *               docker.io/prom/node-exporter:v0.18.1  e5a616e4b9cf  
osd.all-available-devices      8/8  6m ago     3w   *               docker.io/ceph/ceph:v15               5553b0cb212c  
prometheus                     1/1  5s ago     3w   count:1         docker.io/prom/prometheus:v2.18.1     de242295e225  
rgw.demodom.fr-est-1           0/1  -          -    cnrgw1;count:1  <unknown>                             <unknown>     

# Remarque : initialisation de l'installation de rgw.demodom.fr-est-1

[ceph: root@cn1 /]# ceph orch ls
NAME                       RUNNING  REFRESHED  AGE  PLACEMENT       IMAGE NAME                            IMAGE ID      
alertmanager                   1/1  28s ago    3w   count:1         docker.io/prom/alertmanager:v0.20.0   0881eb8f169f  
crash                          6/6  6m ago     3w   *               docker.io/ceph/ceph:v15               5553b0cb212c  
grafana                        1/1  28s ago    3w   count:1         docker.io/ceph/ceph-grafana:6.6.2     a0dce381714a  
mgr                            2/2  6m ago     3w   count:2         docker.io/ceph/ceph:v15               5553b0cb212c  
mon                            5/5  6m ago     3w   count:5         docker.io/ceph/ceph:v15               5553b0cb212c  
node-exporter                  6/6  6m ago     3w   *               docker.io/prom/node-exporter:v0.18.1  e5a616e4b9cf  
osd.all-available-devices      8/8  6m ago     3w   *               docker.io/ceph/ceph:v15               5553b0cb212c  
prometheus                     1/1  28s ago    3w   count:1         docker.io/prom/prometheus:v2.18.1     de242295e225  
rgw.demodom.fr-est-1           1/1  15s ago    21s  cnrgw1;count:1  docker.io/ceph/ceph:v15               5553b0cb212c  

# Remarque : rgw.demodom.fr-est-1 est en fonctionnement

[ceph: root@cn1 /]# curl http://cnrgw1 |xmllint --format -
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   214    0   214    0     0  30571      0 --:--:-- --:--:-- --:--:-- 30571
<?xml version="1.0" encoding="UTF-8"?>
<ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
  <Owner>
    <ID>anonymous</ID>
    <DisplayName/>
  </Owner>
  <Buckets/>
</ListAllMyBucketsResult>

# Remarque: il y a bien un service http sur le port 80 de cnrgw1

[ceph: root@cn1 /]# ## Ajouter une deuxieme rgw pour fr-est => fr-est-2
[ceph: root@cn1 /]# RGW_MASTERZONE=fr-est-2
[ceph: root@cn1 /]# # hostname pour la deuxieme rgw fr-est-2
[ceph: root@cn1 /]# RADOSGW=cnrgw2
[ceph: root@cn1 /]# radosgw-admin zone create --rgw-zonegroup=$RGW_MASTERZONEGRP --rgw-zone=$RGW_MASTERZONE \
>  --endpoints=http://$RADOSGW:80 --access-key=$SYSTEM_ACCESS_KEY \
>  --secret=$SYSTEM_SECRET_KEY
{
    "id": "040b9be7-0df6-4961-9eeb-d919bdfeaa12",
    "name": "fr-est-2",
    "domain_root": "fr-est-2.rgw.meta:root",
    "control_pool": "fr-est-2.rgw.control",
    "gc_pool": "fr-est-2.rgw.log:gc",
    "lc_pool": "fr-est-2.rgw.log:lc",
    "log_pool": "fr-est-2.rgw.log",
    "intent_log_pool": "fr-est-2.rgw.log:intent",
    "usage_log_pool": "fr-est-2.rgw.log:usage",
    "roles_pool": "fr-est-2.rgw.meta:roles",
    "reshard_pool": "fr-est-2.rgw.log:reshard",
    "user_keys_pool": "fr-est-2.rgw.meta:users.keys",
    "user_email_pool": "fr-est-2.rgw.meta:users.email",
    "user_swift_pool": "fr-est-2.rgw.meta:users.swift",
    "user_uid_pool": "fr-est-2.rgw.meta:users.uid",
    "otp_pool": "fr-est-2.rgw.otp",
    "system_key": {
        "access_key": "DemoZoneSyncKeyAcc",
        "secret_key": "DemoZoneSyncKeyMegaSecret"
    },
    "placement_pools": [
        {
            "key": "default-placement",
            "val": {
                "index_pool": "fr-est-2.rgw.buckets.index",
                "storage_classes": {
                    "STANDARD": {
                        "data_pool": "fr-est-2.rgw.buckets.data"
                    }
                },
                "data_extra_pool": "fr-est-2.rgw.buckets.non-ec",
                "index_type": 0
            }
        }
    ],
    "realm_id": "6136dc56-ee0b-4c00-be95-feefee061d1d"
}
 cluster CEPH.1 /]# # remarque : il n'y a pas de répliquation, car les objets sont stoqué dans le même pool sur le même 
[ceph: root@cn1 /]# 

[ceph: root@cn1 /]# radosgw-admin period update --commit
{
    "id": "8f0acd3b-6b23-4807-8ae4-543b0957c5ab",
    "epoch": 2,
    "predecessor_uuid": "b5ea9223-2ae5-4fc0-bfd8-318001256176",
    "sync_status": [],
    "period_map": {
        "id": "8f0acd3b-6b23-4807-8ae4-543b0957c5ab",
        "zonegroups": [
            {
                "id": "4c959d20-3520-4c90-b026-d707c9f9332d",
                "name": "fr",
                "api_name": "fr",
                "is_master": "true",
                "endpoints": [
                    "http://cnrgw1:80"
                ],
                "hostnames": [],
                "hostnames_s3website": [],
                "master_zone": "0238b991-54b6-4dee-8757-5579990fef9d",
                "zones": [
                    {
                        "id": "0238b991-54b6-4dee-8757-5579990fef9d",
                        "name": "fr-est-1",
                        "endpoints": [
                            "http://cnrgw1:80"
                        ],
                        "log_meta": "false",
                        "log_data": "true",
                        "bucket_index_max_shards": 11,
                        "read_only": "false",
                        "tier_type": "",
                        "sync_from_all": "true",
                        "sync_from": [],
                        "redirect_zone": ""
                    },
                    {
                        "id": "040b9be7-0df6-4961-9eeb-d919bdfeaa12",
                        "name": "fr-est-2",
                        "endpoints": [
                            "http://cnrgw2:80"
                        ],
                        "log_meta": "false",
                        "log_data": "true",
                        "bucket_index_max_shards": 11,
                        "read_only": "false",
                        "tier_type": "",
                        "sync_from_all": "true",
                        "sync_from": [],
                        "redirect_zone": ""
                    }
                ],
                "placement_targets": [
                    {
                        "name": "default-placement",
                        "tags": [],
                        "storage_classes": [
                            "STANDARD"
                        ]
                    }
                ],
                "default_placement": "default-placement",
                "realm_id": "6136dc56-ee0b-4c00-be95-feefee061d1d",
                "sync_policy": {
                    "groups": []
                }
            }
        ],
        "short_zone_ids": [
            {
                "key": "0238b991-54b6-4dee-8757-5579990fef9d",
                "val": 2434533641
            },
            {
                "key": "040b9be7-0df6-4961-9eeb-d919bdfeaa12",
                "val": 3410344401
            }
        ]
    },
    "master_zonegroup": "4c959d20-3520-4c90-b026-d707c9f9332d",
    "master_zone": "0238b991-54b6-4dee-8757-5579990fef9d",
    "period_config": {
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
        }
    },
    "realm_id": "6136dc56-ee0b-4c00-be95-feefee061d1d",
    "realm_name": "demodom",
    "realm_epoch": 2
}
[ceph: root@cn1 /]# 


[ceph: root@cn1 /]# # déployer la rgw
[ceph: root@cn1 /]# ceph orch apply rgw $RGW_REALM $RGW_MASTERZONE --placement="1 $RADOSGW"
Scheduled rgw.demodom.fr-est-2 update...
[ceph: root@cn1 /]# # vérification
[ceph: root@cn1 /]# ceph orch ls
NAME                       RUNNING  REFRESHED  AGE  PLACEMENT       IMAGE NAME                            IMAGE ID      
alertmanager                   1/1  7m ago     3w   count:1         docker.io/prom/alertmanager:v0.20.0   0881eb8f169f  
crash                          6/6  7m ago     3w   *               docker.io/ceph/ceph:v15               5553b0cb212c  
grafana                        1/1  7m ago     3w   count:1         docker.io/ceph/ceph-grafana:6.6.2     a0dce381714a  
mgr                            2/2  7m ago     3w   count:2         docker.io/ceph/ceph:v15               5553b0cb212c  
mon                            5/5  7m ago     3w   count:5         docker.io/ceph/ceph:v15               5553b0cb212c  
node-exporter                  6/6  7m ago     3w   *               docker.io/prom/node-exporter:v0.18.1  e5a616e4b9cf  
osd.all-available-devices      8/8  7m ago     3w   *               docker.io/ceph/ceph:v15               5553b0cb212c  
prometheus                     1/1  7m ago     3w   count:1         docker.io/prom/prometheus:v2.18.1     de242295e225  
rgw.demodom.fr-est-1           1/1  7m ago     7m   cnrgw1;count:1  docker.io/ceph/ceph:v15               5553b0cb212c  
rgw.demodom.fr-est-2           0/1  -          -    cnrgw2;count:1  <unknown>                             <unknown>     

[ceph: root@cn1 /]# ceph orch ls
NAME                       RUNNING  REFRESHED  AGE  PLACEMENT       IMAGE NAME                            IMAGE ID      
alertmanager                   1/1  7m ago     3w   count:1         docker.io/prom/alertmanager:v0.20.0   0881eb8f169f  
crash                          6/6  7m ago     3w   *               docker.io/ceph/ceph:v15               5553b0cb212c  
grafana                        1/1  7m ago     3w   count:1         docker.io/ceph/ceph-grafana:6.6.2     a0dce381714a  
mgr                            2/2  7m ago     3w   count:2         docker.io/ceph/ceph:v15               5553b0cb212c  
mon                            5/5  7m ago     3w   count:5         docker.io/ceph/ceph:v15               5553b0cb212c  
node-exporter                  6/6  7m ago     3w   *               docker.io/prom/node-exporter:v0.18.1  e5a616e4b9cf  
osd.all-available-devices      8/8  7m ago     3w   *               docker.io/ceph/ceph:v15               5553b0cb212c  
prometheus                     1/1  7m ago     3w   count:1         docker.io/prom/prometheus:v2.18.1     de242295e225  
rgw.demodom.fr-est-1           1/1  7m ago     7m   cnrgw1;count:1  docker.io/ceph/ceph:v15               5553b0cb212c  
rgw.demodom.fr-est-2           1/1  10s ago    14s  cnrgw2;count:1  docker.io/ceph/ceph:v15               5553b0cb212c  

[ceph: root@cn1 /]# curl http://cnrgw2 |xmllint --format -
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   214    0   214    0     0  71333      0 --:--:-- --:--:-- --:--:-- 71333
<?xml version="1.0" encoding="UTF-8"?>
<ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
  <Owner>
    <ID>anonymous</ID>
    <DisplayName/>
  </Owner>
  <Buckets/>
</ListAllMyBucketsResult>


[ceph: root@cn1 /]# ceph df
--- RAW STORAGE ---
CLASS  SIZE     AVAIL    USED     RAW USED  %RAW USED
hdd    360 GiB  351 GiB  638 MiB   8.6 GiB       2.40
TOTAL  360 GiB  351 GiB  638 MiB   8.6 GiB       2.40
 
--- POOLS ---
POOL                   ID  PGS  STORED   OBJECTS  USED     %USED  MAX AVAIL
device_health_metrics   1    1  1.3 KiB        0  3.9 KiB      0    111 GiB
.rgw.root               8   32  6.1 KiB       19  3.4 MiB      0    111 GiB
fr-est-1.rgw.log        9   32  8.5 KiB      434   30 MiB      0    111 GiB
fr-est-1.rgw.control   10   32      0 B        8      0 B      0    111 GiB
fr-est-1.rgw.meta      11    8    348 B        2  384 KiB      0    111 GiB
fr-est-2.rgw.log       12   32   11 KiB      547   43 MiB   0.01    111 GiB
fr-est-2.rgw.control   13   32      0 B        8      0 B      0    111 GiB
fr-est-2.rgw.meta      14   13    348 B        2  384 KiB      0    111 GiB
fr-est-1.rgw.otp       15   32      0 B        0      0 B      0    111 GiB

# Quittez le contenaire pour tester le service S3
[ceph: root@cn1 /]# exit
exit

[vagrant@cn1 ~]$ # vérification que cnrgw1 et cnrgw2 sont accéssible 
[vagrant@cn1 ~]$ curl http://cnrgw1 |xmllint --format -
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
[vagrant@cn1 ~]$ curl http://cnrgw2 |xmllint --format -
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


# Remarque : cnrgw1 et cnrgw2 sont bien accessible depuis le port 80


[vagrant@cn1 ~]$ sudo dnf install -y awscli
[vagrant@cn1 ~]$ RGW_REALM=demodom
[vagrant@cn1 ~]$ RGW_MASTERZONEGRP=fr
[vagrant@cn1 ~]$ RGW_MASTERZONE=fr-est-1
[vagrant@cn1 ~]$ SYSTEM_ACCESS_KEY=DemoZoneSyncKeyAcc
[vagrant@cn1 ~]$ SYSTEM_SECRET_KEY=DemoZoneSyncKeyMegaSecret
[vagrant@cn1 ~]$ export AWS_ACCESS_KEY_ID=$SYSTEM_ACCESS_KEY
[vagrant@cn1 ~]$ export AWS_SECRET_ACCESS_KEY=$SYSTEM_SECRET_KEY
[vagrant@cn1 ~]$ export AWS_DEFAULT_REGION=$RGW_MASTERZONEGRP
[vagrant@cn1 ~]$ 
[vagrant@cn1 ~]$ aws s3 mb s3://mybucket  --endpoint-url http://cnrgw1
[vagrant@cn1 ~]$ aws s3 ls  --endpoint-url http://cnrgw1
2021-02-22 08:42:12 mybucket
[vagrant@cn1 ~]$ aws s3 ls  --endpoint-url http://cnrgw2
2021-02-22 08:42:12 mybucket
[vagrant@cn1 ~]$ aws s3 mb s3://mybucket2  --endpoint-url http://cnrgw2
[vagrant@cn1 ~]$ aws s3 ls  --endpoint-url http://cnrgw2
2021-02-22 08:42:12 mybucket
2021-02-22 08:43:47 mybucket2
[vagrant@cn1 ~]$ aws s3 ls  --endpoint-url http://cnrgw1
2021-02-22 08:42:12 mybucket
2021-02-22 08:43:47 mybucket2

# remarque: cnrgw1 et cnrgw2 utilise le même pool du même cluster CEPH, donc c'est logique qu'il y a les même informations

[vagrant@cn1 ~]$ 
[vagrant@cn1 ~]$ aws s3 cp /etc/hosts s3://mybucket2/hostcn1  --endpoint-url http://cnrgw2
upload: ../../etc/hosts to s3://mybucket2/hostcn1                   
[vagrant@cn1 ~]$ aws s3 cp s3://mybucket2/hostcn1 /tmp/hostcn1  --endpoint-url http://cnrgw1
download: s3://mybucket2/hostcn1 to ../../tmp/hostcn1              
[vagrant@cn1 ~]$ head -n 5 /tmp/hostcn1 
127.0.0.1	cn1	cn1
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

127.0.0.1 centos8.localdomain

# remarque: cnrgw1 et cnrgw2 peuvent être utilisé avec un DNS ou une solution en HA

# connexion sur cephclt qui n'est pas dans le cluster Ceph
sg4r@work:~/dev/ceph-octopus$ vagrant ssh cephclt
Last login: Wed Jan 20 17:07:08 2021 from 192.168.121.1
[vagrant@cephclt ~]$ 
[vagrant@cephclt ~]$ sudo yum install awscli
#Configuration de aws pour l'acces au service RadosGW avec l'utilisateur créé précédament
[vagrant@cephclt ~]$ aws configure
AWS Access Key ID [None]: testuserkacc
AWS Secret Access Key [None]: testuserkpwd
Default region name [None]: fr-est-1
Default output format [None]: json

# Création d'un bucket S3. il doit être unique.
[vagrant@cephclt ~]$ aws s3 mb s3://my-new-bucket --endpoint-url http://cnrgw1
make_bucket: my-new-bucket
# Quelques exemple de manipulation de fichiers
[vagrant@cephclt ~]$ aws s3 cp /etc/hosts s3://my-new-bucket --endpoint-url http://cnrgw1
upload: ../../etc/hosts to s3://my-new-bucket/hosts                 
[vagrant@cephclt ~]$ aws s3 ls s3://my-new-bucket --endpoint-url http://cnrgw1
2021-01-21 09:11:06        323 hosts

[vagrant@cephclt ~]$ mkdir /tmp/f2sync
[vagrant@cephclt ~]$ echo fichier1 >>/tmp/f2sync/fichier1.txt
[vagrant@cephclt ~]$ echo fichier2 >>/tmp/f2sync/fichier2.txt
[vagrant@cephclt ~]$ mkdir /tmp/f2sync/f2
[vagrant@cephclt ~]$ echo fichier3 >>/tmp/f2sync/f2/fichier3.txt
[vagrant@cephclt ~]$ aws s3 sync /tmp/f2sync  s3://my-new-bucket/ --endpoint-url http://cnrgw1 --delete
delete: s3://my-new-bucket/hosts
upload: ../../tmp/f2sync/fichier1.txt to s3://my-new-bucket/fichier1.txt
upload: ../../tmp/f2sync/f2/fichier3.txt to s3://my-new-bucket/f2/fichier3.txt
upload: ../../tmp/f2sync/fichier2.txt to s3://my-new-bucket/fichier2.txt

[vagrant@cephclt ~]$ aws s3 ls s3://my-new-bucket --endpoint-url http://cnrgw1 --recursive
2021-01-21 10:05:45          9 f2/fichier3.txt
2021-01-21 10:05:45          9 fichier1.txt
2021-01-21 10:05:45          9 fichier2.txt

# et voila pour une première installation de RGW pour fournir un accès S3 depuis votre cluster Ceph.
```
## RGW DATA CACHING AND CDN
Cette fonctionnalité disponible depuis la version Octopus, ajoute à RGW la possibilité de mettre en cache des objets en toute sécurité et de décharger la charge de travail du cluster, à l'aide de Nginx. Après avoir accédé à un objet la première fois, il sera stocké dans le répertoire de cache Nginx. Lorsque les données sont déjà mises en cache, elles n'ont pas besoin d'être extraites de RGW. Une vérification d'autorisation sera effectuée par rapport à RGW pour s'assurer que l'utilisateur demandeur a accès. Cette fonctionnalité est basée sur des modules Nginx, ngx_http_auth_request_module, https://github.com/kaltura/nginx-aws-auth-module et Openresty.

Pour plus d'information voir https://docs.ceph.com/en/latest/radosgw/rgw-cache/
## RGW HA via Ceph orch
L’orchestrateur de ceph permet de créer un service RGW à haute disponibilité. Ceph Orch déploiera et configurera automatiquement plusieurs conteneurs HAProxy et Keepalived pour assurer la continuité du service RGW tandis que le cluster Ceph aura au moins 1 démon RGW en cours d'exécution. Voir le schéma ci-dessous.
![cephs3-ha.jpg](cephs3-ha.jpg)
Pour plus d’information voir https://docs.ceph.com/en/latest/cephadm/services/rgw/#high-availability-service-for-rgw

Remarque: cela fonctionne a partir de la version ceph octopus
```
[vagrant@cn1 ~]# openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ./rgwha.key -out ./rgwha.crt
Generating a RSA private key
.............+++++
........................................................+++++
writing new private key to './rgwha.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:fr
State or Province Name (full name) []:est
Locality Name (eg, city) [Default City]:sbg
Organization Name (eg, company) [Default Company Ltd]:cephlab.org    
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:cnrgwha
Email Address []:root@cephlab.org

# creer le fichier rgwha.yaml en concaténant les 2 fichiers du certificat et rajouter l'entête pour la configuration du service haproxy
cat rgwha.crt rgwha.key >rgwha.yaml

#voici un exmple de configuration

service_type: ingress
service_id: rgwhademo
placement:
  hosts:
    - cnrgw1
    - cnrgw2
spec:
  backend_service: rgw.demodom
  virtual_ip: 192.168.111.15/24
  frontend_port: 443
  monitor_port: 1967
  ssl_cert: |
    -----BEGIN CERTIFICATE-----
    MIIDxzCCAq+gAwIBAgIUPB57k0KuWd3Tbp4qNo2meyJV7m0wDQYJKoZIhvcNAQEL
    ...
    heSMHZro5QphvSg=
    -----END CERTIFICATE-----
    -----BEGIN PRIVATE KEY-----
    MIIEvAIBADANBgkqhkiG9w0BAQEFAASCBKYwggSiAgEAAoIBAQDJ2p0L16PVqkKt
    ...
    9B48bfj5CwHeeNQKn1SurA==
    -----END PRIVATE KEY-----

# pour accéder au fichier rgwha.yaml depuis le shell de cephadm, il est possible de rajouter un montage au conteneur
sudo cephadm shell --mount /home/vagrant
# appliquer l'installation du haproxy
ceph orch apply -i  /mnt/vagrant/rgwha.yaml
# si la configuration est ok, il devrais avoir 4 nouveaux conteneurs
[ceph: root@cna1 /]# ceph orch ls |grep ingress
ingress.rgwhademo          192.168.111.15:443,1967      4/4  9m ago     30m  cnrgw1;cnrgw2
# vérification
curl --insecure  https://192.168.111.15
<?xml version="1.0" encoding="UTF-8"?><ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/">...

# vérifier depuis le client
[vagrant@cephclt ~]$ export PYTHONWARNINGS="ignore" 

[vagrant@cephclt ~]$ aws --no-verify-ssl --endpoint-url https://192.168.111.15 s3 ls mybucket   
2022-04-01 13:48:47        342 hostcn1
# vérifier que le service ha fonctionne en killant un des service rgwcn1 ou rgwcn2 ;)
```

## Documentation
Pour plus d’information voir https://docs.ceph.com/en/latest/radosgw/

