# Stockage S3 / Swift
Ceph Rados Gateway est une interface de stockage d'objets construite sur la librados pour fournir aux applications une passerelle RESTful vers les clusters de stockage Ceph.
Ceph Rados Gateway prend en charge deux interfaces:
 - S3: fournit un stockage compatible avec l'API RESTful Amazon S3.
 - Swift: fournit un stockage compatible avec l'API OpenStack Swift.
## Topologie réseau
La passerelle est accessible depuis l’internet, donc installer RadosGW sur un serveur dédié dans votre DMZ. Il est
possible de répartir la charge avec un HAPROXY. Les flux librados seront autorisé vers le réseau publique votre cluster Ceph.
![cephradosgw](cephradosgw.png)
source https://qiita.com/jundo414/items/7ac3bdf3967ec67d7680

## Installation de RadosGW
Le service RadosGW s’installe généralement sur un nœud dédié. Prévoir un CPU à 8 coeurs et entre 32Go à 64Go de
RAM en fonction du nombre de clients. Pour équilibrer la charge, il est conseillé d’utiliser plusieurs RadosGW avec un
proxyHA et comptez une RadosGW pour 50-100 OSDs .
```
# a completer
```
## RGW DATA CACHING AND CDN
Cette fonctionnalité disponible depuis la version Octopus, ajoute à RGW la possibilité de mettre en cache des objets en toute sécurité et de décharger la charge de travail du cluster, à l'aide de Nginx. Après avoir accédé à un objet la première fois, il sera stocké dans le répertoire de cache Nginx. Lorsque les données sont déjà mises en cache, elles n'ont pas besoin d'être extraites de RGW. Une vérification d'autorisation sera effectuée par rapport à RGW pour s'assurer que l'utilisateur demandeur a accès. Cette fonctionnalité est basée sur des modules Nginx, ngx_http_auth_request_module, https://github.com/kaltura/nginx-aws-auth-module et Openresty.

Pour plus d'information voir https://docs.ceph.com/en/latest/radosgw/rgw-cache/
## Documentation
Pour plus d’information voir https://docs.ceph.com/en/latest/radosgw/

