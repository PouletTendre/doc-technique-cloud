# Hyperconvergence (HCI) et modèle de nœuds mutualisés

## Ce qu'est le modèle HCI

L'hyperconvergence (HCI, Hyper-Converged Infrastructure) réunit sur les mêmes machines la puissance de calcul (exécution des applications), le stockage et le réseau distribué, là où les architectures traditionnelles séparent les serveurs applicatifs des baies de stockage SAN ou NAS dédiées.

Dans un cluster Kubernetes, un modèle non hyperconvergé dédie des VMs au calcul (nœuds workers applicatifs) et d'autres VMs uniquement au stockage. À l'inverse, un modèle HCI fait tourner les pods applicatifs et les services de stockage distribué sur les mêmes nœuds workers.

## Comment fonctionne le HCI avec Longhorn V2

Sur un nœud HCI, deux mondes cohabitent sur la même machine virtuelle :

1. Le plan applicatif exécute les pods utilisateurs ou internes via `kubelet` et `containerd`.
2. Le plan de stockage fait tourner le moteur Longhorn V2 (`longhorn-instance-manager` et le réacteur SPDK `spdk_tgt`) qui gère un disque brut dédié (`/dev/vdb`) et réplique les blocs en NVMe-oF TCP vers les autres nœuds HCI sur le lien 10 Gbps.

Le gain principal vient de la « Data Locality » : lorsqu'un pod a besoin de lire un volume dont l'un des trois réplicas est stocké sur le même nœud que lui, la lecture passe directement par le bus local PCIe/mémoire sans traverser le réseau. Lors du PoC Stockage, cette proximité a permis d'atteindre 301,66 Mo/s en lecture séquentielle 128K en mode HCI, contre 56,13 Mo/s en mode stockage dédié sur réseau.

En contrepartie, la cohabitation expose les nœuds à un risque de contention processeur. Le réacteur SPDK de Longhorn V2 travaille en scrutation active (`busy-polling`) à 100 % d'un cœur CPU. Si des conteneurs applicatifs consomment tout le temps processeur de la VM, SPDK subit une famine CPU et la latence disque grimpe aussitôt (lors des tests de concurrence massive à 50 clients sur le PoC, la latence du HCI sans isolation CPU montait à 43,96 ms contre 1,95 ms avec un cœur dédié).

## Articulation dans le projet CaaScad

CaaScad a retenu le modèle HCI pour ses deux clusters afin de limiter le nombre total de VMs à maintenir et de profiter de la capacité des serveurs physiques en salle U3-316 (96 à 128 CPU, 384 Go de RAM, liens 10 Gbps). Le passage au HCI a permis de ramener le parc Kubernetes à 13 nœuds au total (3 masters + 3 HCI en Interne, 3 masters + 4 HCI en Client), au lieu des 20 VMs envisagées initialement avec des nœuds stockage et runners séparés.

### Dimensionnement et isolation CPU sur les 7 nœuds HCI

Chaque nœud HCI (`int-hci1` à `int-hci3` et `cli-hci1` à `cli-hci4`) dispose de 8 vCPU, 16 Go de RAM, d'un disque système `/dev/vda` de 60 Go et d'un disque brut `/dev/vdb` de 200 Go dédié à SPDK. Pour éviter la contention CPU observée lors du PoC, la configuration RKE2 prévue par le LLD applique trois garde-fous (marqués à vérifier au déploiement sur les VMs finales) :

- Allocation de 2 Go de mémoire en HugePages 2M (`/etc/sysctl.d/99-longhorn-v2.conf` avec `vm.nr_hugepages = 1024`), indispensables au fonctionnement de SPDK en espace utilisateur.
- Activation de `cpu-manager-policy=static` sur le kubelet pour épingler les processus critiques sur des cœurs fixes.
- Réservation système `system-reserved=cpu=1,memory=2Gi` afin de sanctuariser 1 vCPU entier pour le réacteur `spdk_tgt` et de laisser 7 vCPU disponibles aux pods.

### Capacité utile par cluster

Avec un facteur de réplication synchrone de 3 réplicas et une anti-affinité stricte (`replicaSoftAntiAffinity: false`, garantissant que les trois copies d'un volume résident sur trois nœuds HCI différents), la capacité se répartit ainsi :

| Cluster | Nœuds HCI | Disques bruts (`/dev/vdb`) | Capacité utile (x3) | Capacité commercialisable (Client) |
|---|---|---|---|---|
| Interne | 3 (`bob`, `oscar`, `eve`) | 3 x 200 Go (600 Go) | ~200 Go | Usage interne uniquement |
| Client | 4 (`alice`, `bob`, `oscar`, `eve`) | 4 x 200 Go (800 Go) | ~266 Go | ~240 Go PVC, 28 vCPU, 52 Go RAM |

Sur le cluster interne, la perte d'un nœud HCI laisse les deux copies survivantes actives et permet de continuer à servir les volumes (même si la reconstruction d'un troisième réplica doit attendre le retour d'un troisième nœud). Sur le cluster client, la présence de quatre nœuds HCI sur quatre serveurs physiques différents permet même de reconstruire automatiquement le troisième réplica sur le quatrième nœud survivant en cas de panne d'un hyperviseur. Un plafond de bande passante (`replica-rebuild-bandwidth-limit`) limite ce trafic de reconstruction pour préserver les flux applicatifs des autres clients sur le lien 10 Gbps.
