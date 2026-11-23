# Kubernetes sur RKE2 et stockage Longhorn V2

## Ce que sont Kubernetes, RKE2 et Longhorn V2

Kubernetes est un orchestrateur de conteneurs. Il répartit des applications sur plusieurs machines, surveille leur état de santé, redémarre un conteneur qui plante et expose ces applications sur le réseau sans intervention manuelle sur chaque serveur.

RKE2 (Rancher Kubernetes Engine 2) est une distribution Kubernetes maintenue par SUSE Rancher. Elle livre un binaire unique conforme aux recommandations de sécurité CIS par défaut et isole les composants du plan de contrôle dans des conteneurs statiques gérés par containerd.

Longhorn V2 est un système de stockage bloc distribué conçu pour Kubernetes. Chaque volume créé pour un pod est découpé et répliqué sur les disques locaux de plusieurs nœuds du cluster. La version V2 remplace le moteur historique par SPDK (Storage Performance Development Kit) en espace utilisateur et le protocole NVMe-over-Fabrics (NVMe-oF TCP), ce qui supprime une grande partie des allers-retours avec le noyau Linux lors des écritures disque.

## Comment fonctionnent RKE2 et Longhorn V2

Dans un cluster RKE2, deux types de nœuds se partagent le travail :

- Les nœuds masters (`server`) hébergent l'API Kubernetes (`kube-apiserver`), le planificateur (`kube-scheduler`), les contrôleurs (`kube-controller-manager`) et la base d'état distribuée `etcd`. Ils portent la taint `node-role.kubernetes.io/control-plane:NoSchedule` pour qu'aucun conteneur applicatif ne vienne perturber le plan de contrôle.
- Les nœuds workers (`agent`) font tourner `kubelet`, `containerd` et les pods applicatifs.

Lorsque RKE2 démarre, les composants réseau historiques peuvent être désactivés dès le fichier `/etc/rancher/rke2/config.yaml`. C'est utile quand on souhaite utiliser une pile réseau plus moderne comme Cilium à la place de Canal et `kube-proxy`.

Côté stockage, Longhorn V2 fonctionne via un processus dédié appelé `spdk_tgt` qui tourne sur chaque nœud participant au stockage. Ce réacteur travaille en scrutation active (`busy-polling`) : un cœur processeur vérifie en continu les files d'entrées-sorties NVMe au lieu d'attendre des interruptions matérielles du noyau. Lorsqu'un pod écrit une donnée sur son PersistentVolumeClaim (PVC), Longhorn écrit simultanément cette donnée sur trois réplicas répartis sur des nœuds distincts via le réseau TCP/IP à 10 Gbps. Si un nœud tombe, le volume bascule sur les deux réplicas restants en deux à cinq secondes.

## Articulation dans le projet CaaScad

CaaScad exploite deux clusters RKE2 v1.32.x indépendants, installés sur des machines virtuelles Ubuntu Server 24.04 LTS (noyau 6.8+, swap désactivé, modules `overlay`, `br_netfilter` et `nvme_tcp` chargés au démarrage).

| Cluster | VLAN nœuds | Masters (etcd) | Nœuds HCI | Service CIDR (ULA) | VIP API (OOB / Cluster) |
|---|---|---|---|---|---|
| Interne | 770 (`bb20::/64`) | `int-m1` à `int-m3` | `int-hci1` à `int-hci3` | `fd07:63fe:c5dd:101::/112` | `1:1::10` / `bb20::10` |
| Client | 769 (`bb23::/64`) | `cli-m1` à `cli-m3` | `cli-hci1` à `cli-hci4` | `fd07:63fe:c5dd:102::/112` | `1:2::10` / `bb23::10` |

Séparer les deux clusters évite qu'une charge client ou une opération de maintenance sur le cluster client ne déstabilise les outils d'administration internes (OpenLDAP, OpenBao, ArgoCD, VictoriaMetrics). Sur chaque cluster, les trois masters sont placés sur trois serveurs physiques différents, et `kube-vip` annonce par NDP deux adresses flottantes sur le port 6443 : une VIP sur le réseau du cluster pour les nœuds, et une VIP sur le VLAN 790 OOB pour l'administration depuis le bastion, `runner-infra` et ArgoCD.

Le choix de RKE2 sur Ubuntu 24.04 plutôt que Talos Linux vient de l'expérience de l'équipe avec les playbooks Ansible existants et du besoin de garder la main sur la configuration réseau Netplan et le pare-feu local `nftables`. Dans la configuration RKE2, Canal, `kube-proxy` et `rke2-ingress-nginx` sont désactivés pour laisser toute la place à Cilium 1.19.

Sur le stockage, le PoC réalisé en octobre a tranché en faveur de Longhorn V2 (3 réplicas) face à Ceph Squid. Les écarts mesurés sur un matériel équivalent ont motivé ce retrait de Ceph :

| Profil de test (PoC Stockage) | Ceph Squid v19 | Longhorn V2 HCI | Observation |
|---|---|---|---|
| Microservices CRUD 4K (60/40) | 1 669,2 IOPS (4,86 ms) | 4 646,7 IOPS (1,19 ms) | Latence divisée par 4 |
| Base de données WAL 8K (`fsync=1`) | 41,7 TPS (8,38 ms) | 230,7 TPS (1,57 ms) | 5,5 fois plus de transactions/s |
| Scan séquentiel 128K | 92,56 Mo/s (1,35 ms) | 301,66 Mo/s (0,41 ms) | Lecture locale sur le bus PCIe |

Chaque nœud HCI reçoit un disque système `/dev/vda` de 60 Go en `ext4` et un disque brut non partitionné `/dev/vdb` de 200 Go attribué au moteur SPDK. Pour éviter que le `busy-polling` de `spdk_tgt` ne soit ralenti par les conteneurs applicatifs, les nœuds HCI réservent `vm.nr_hugepages = 1024` (2 Go de HugePages 2M), activent `cpu-manager-policy=static` sur le kubelet et réservent 1 vCPU et 2 Go de RAM via `system-reserved` (ces paramètres sont marqués à vérifier lors de la mise en service sur les machines cibles dans le LLD).

La capacité utile en réplication par trois atteint environ 200 Go sur le cluster interne (3 nœuds HCI) et environ 266 Go bruts côté client (4 nœuds HCI, soit 240 Go commercialisables après marge système). Deux StorageClasses sont exposées aux workloads : `longhorn-v2` par défaut en mode `ReadWriteOnce` (`WaitForFirstConsumer`, `dataLocality: best-effort`), et `longhorn-v2-rwx` pour les besoins ponctuels de volumes partagés en écriture via le composant NFSv4.1 `share-manager`.
