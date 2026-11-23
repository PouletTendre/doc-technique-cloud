# Manifests Kubernetes et base d'état etcd

## Ce que sont les manifests Kubernetes et etcd

Un manifest Kubernetes est un fichier descriptif, écrit au format YAML, qui déclare l'état souhaité d'une ressource du cluster. Au lieu d'enchaîner des scripts impératifs (« lance ce conteneur, puis ouvre tel port »), on décrit le résultat attendu : trois réplicas d'un serveur web, tant de mémoire allouée, tel volume persistant ou telle règle réseau.

`etcd` est la base de données clé-valeur distribuée qui stocke tous ces objets et l'état courant du cluster. Elle contient la définition de chaque Deployment, Pod, Service, Secret, ConfigMap ou règle réseau personnalisée. Si `etcd` est perdue sans sauvegarde, le cluster perd la mémoire de tout ce qui y est déployé, même si les conteneurs continuent temporairement de tourner sur les nœuds.

## Comment fonctionnent les manifests et etcd

### La boucle de réconciliation

Lorsque vous soumettez un manifest à l'API Kubernetes (`kube-apiserver`), l'API valide sa syntaxe, applique les contrôles d'admission (comme les Pod Security Standards), puis écrit l'objet dans `etcd`. À partir de là, les contrôleurs de Kubernetes travaillent en boucle :

- Le `kube-controller-manager` et le `kube-scheduler` constatent que l'état souhaité dans `etcd` diffère de la réalité (par exemple, zéro pod ne tourne alors que le manifest en demande deux) et planifient les pods sur des nœuds disponibles.
- Sur chaque nœud, `kubelet` lit les affectations depuis l'API, demande à `containerd` de démarrer les conteneurs et renvoie l'état réel vers `etcd`.

Les extensions installées dans un cluster ajoutent leurs propres types de manifests, appelés CRD (Custom Resource Definitions). Dans notre architecture, Cilium ajoute des manifests comme `CiliumNetworkPolicy`, `CiliumBGPClusterConfig` ou `CiliumLoadBalancerIPPool`, qui suivent exactement le même cycle de validation et de stockage dans `etcd`.

### Le consensus Raft et le quorum d'etcd

Pour tolérer la panne d'un serveur master sans perdre de données ni créer de divergence (« split-brain »), `etcd` tourne en cluster et utilise l'algorithme de consensus Raft. Un nœud est élu leader pour traiter les écritures ; les autres sont followers et répliquent chaque opération dans leur journal disque (WAL, Write-Ahead Log) avant de valider l'écriture.

La règle du quorum impose qu'une majorité stricte des membres soit joignable (`N/2 + 1`). Avec trois masters, le quorum vaut 2 : le cluster supporte la perte d'un master en continuant à lire et écrire. Avec deux masters seulement, la perte d'un nœud laisse 1 membre sur 2, ce qui ne forme plus de majorité et bloque toute modification du cluster. Au-delà des pannes réseau, `etcd` est très sensible à la latence d'écriture sur disque lors de l'appel `fsync` du WAL : au-dessus de 10 millisecondes au 99e centile, les élections de leader se multiplient et déstabilisent l'API Kubernetes.

## Articulation dans le projet CaaScad

CaaScad maintient une séparation nette entre les deux clusters, chacun disposant de sa propre grappe `etcd` sur trois masters placés sur trois serveurs physiques distincts :

| Cluster | Masters et placement KVM | Tolérance aux pannes | Chiffrement des Secrets |
|---|---|---|---|
| Cluster Interne | `int-m1` (Alice), `int-m2` (Bob), `int-m3` (Oscar) | 1 hyperviseur parmi Alice, Bob, Oscar | `secrets-encryption: true` |
| Cluster Client | `cli-m1` (Alice), `cli-m2` (Oscar), `cli-m3` (Eve) | 1 hyperviseur parmi Alice, Oscar, Eve | `secrets-encryption: true` |

Chaque master est dimensionné avec 4 vCPU, 8 Go de RAM et un disque système de 60 Go. Dans `/etc/rancher/rke2/config.yaml`, l'option `secrets-encryption: true` garantit que les objets `Secret` stockés dans `etcd` (y compris les identifiants synchronisés depuis OpenBao par External Secrets Operator) sont chiffrés au repos sur le disque des masters.

La sauvegarde et la surveillance d'etcd sont automatisées des deux côtés :

- RKE2 exécute un snapshot natif toutes les six heures (`etcd-snapshot-schedule-cron: "0 */6 * * *"`) à 00h, 06h, 12h et 18h, avec une rétention locale de 12 clichés sur les masters.
- La VM `backup` (sur Bob, VLAN 762) récupère ces snapshots dans `/srv/borg/staging/etcd/`, les chiffre côté client avec BorgBackup à 02h00, puis réplique le dépôt vers Hetzner Storage Box à 04h00.
- `vmagent` interroge les métriques `etcd` sur le port TCP 2381 (via l'interface OOB VLAN 790) toutes les 30 secondes. Grafana déclenche une alerte d'avertissement si la latence `fsync` p99 dépasse 10 ms pendant une minute, et une alerte critique en cas de changement de leader ou de perte de quorum.

Enfin, tous les manifests applicatifs (y compris ceux des namespaces clients `tenants/tenant-<client>/` générés par Site Manager) sont versionnés sur `gitlab.inetdoc.net` avant d'être appliqués dans `etcd` par ArgoCD. En cas de reconstruction complète après sinistre, un cluster RKE2 neuf retrouve ainsi l'intégralité de ses manifests en quelques minutes.
