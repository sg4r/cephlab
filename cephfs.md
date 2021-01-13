# CephFS
CephFS, est un système de fichiers compatible POSIX permettant de fournir un stockage de fichiers polyvalent, hautement disponible pour l'utilisation 
de répertoires personnels partagés, d'espace de travail HPC. c'est une alternative au service NFS.

CephFS nécessite au moins deux pools RADOS, un pour les données et un pour les métadonnées.
Le pool de métadonnées est utilisé pour le stockage des informations des répertoires et des fichiers : les inodes, les
proprétaires, les dates de création et de modification et les ACLs. Ce pool doit être de type réplication.
Le pool de données est utilisé par par défaut pour le stockage des données.
Il est possible d’utiliser des pools en erasure code avec CephFS, mais il est préférable d'utiliser un pool en réplication
pour la racine du stockage et de définir par la suite des pools en erasure code pour des répertoires spécifiques.

## création des pools
## création de CephFS
## Documentation
Pour plus d’information voir https://docs.ceph.com/en/latest/cephfs/
