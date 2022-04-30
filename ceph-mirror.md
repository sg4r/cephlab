# Ceph mirroring
Un cluster Ceph distribue sont stockage sur différents serveurs en mode synchrone limitant ainsi les distances entre chaque serveur. C'est pourquoi la mise en place d'un cluster étendu sur de longues distances n'est pas une bonne idée car cela augmente les latences et dégrade les performances.
Pour garantir une disponibilité des données, il est possible de faire de la réplication asynchrone entre deux clusters Ceph distant de plusieurs centaines de kilomètres.  

**Avantage :**
_Plan de reprise d’activité_ : un site de secours peut reprendre l'activité rapidement. Le site dispose d'une copie des données et il n'est pas nécessaire d'avoir la même infrastructure ou la même version de Ceph au niveau des 2 clusters. Chaque cluster peut évoluer à son rythme.  
_Haute disponibilité_ : Avec 2 clusters Ceph distant, il est possible que chacun réplique les données et ce qui permet des clients par zones.  

Il y a un type de réplication en fonction du type de stockage :  

**rbd-mirror** est disponible en deux modes :
Mode basé sur un _journal_ : Ce mode utilise la fonctionnalité de journalisation de l'image RBD afin de garantir une réplication. Les modifications sont appliquées en continue par relecture du journal sur le site distant. Il est possible de choisir de répliquer tout un pool ou une série d’image rbd d’un pool.  
Mode basé sur _snapschot_ : Ce mode utilise des snapshots d'image RBD planifiés régulièrement pour répliquer des images RBD. En fonction de l’intervalle, les modifications ne sont pas présentées sur le site secondaire s’il y a un désastre sur le site primaire.  

Selon les besoins de réplication, la mise en miroir RBD peut être configurée pour une réplication unidirectionnelle ou bidirectionnelle :  
Réplication unidirectionnelle : Lorsque les données sont mises en miroir uniquement à partir du cluster primaire vers le cluster secondaire  
Réplication bidirectionnelle : Lorsque les données sont mises en miroir à partir des images primaires d’un cluster vers le cluster secondaire et inversement. 
Il n’est possible que de réaliser une réplication par sélection d’une série image RBD depuis un ou plusieurs pools.

**cepfs-mirror** : réplication des données CephFS. Disponible depuis la version Ceph Pacific.   
 Fonctionne seulement vers un site de secours avec la possibilité de faire un plan de reprise d'activité. Il possible de sélectionner différents répertoires avec une définition de différents règles de snapshots.

**radosgw-multisites** : à compléter.
