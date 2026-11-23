# Orchestration Kubernetes dans le HLD (Cilium, Multus, WireGuard et stockage)

## Vision d'orchestration posée par le HLD

Dans le High-Level Design (HLD), la couche Kubernetes organise toute la plateforme autour d'une règle structurante : séparer physiquement le plan de gestion interne de l'hébergement des conteneurs clients, tout en gardant des briques identiques pour simplifier l'exploitation.

Plutôt que d'empiler tous les services sur un seul grand cluster mutualisé, CaaScad déploie deux clusters RKE2 sur Ubuntu Server 24.04 LTS :

- Le Cluster Interne (6 VMs : 3 masters sur Alice, Bob, Oscar et 3 nœuds HCI sur Bob, Oscar, Eve) fait tourner les services de l'entreprise, les outils d'exploitation et les portails exposés.
- Le Cluster Client (7 VMs : 3 masters sur Alice, Oscar, Eve et 4 nœuds HCI sur les 4 hyperviseurs) est exclusivement réservé aux conteneurs des clients.

Dans le HLD, les fonctions réseau de base (routage inter-VLAN, pare-feu, DHCPv6/RA, DNS Unbound, NAT64) restent volontairement en dehors de Kubernetes, sur la paire OPNsense HA. Si les clusters Kubernetes redémarrent après une coupure, les pare-feux assurent la résolution DNS et le routage avant même le démarrage du premier pod.

## Comment s'assemblent RKE2, Cilium, WireGuard, Multus et Longhorn V2

Chaque brique retenue dans le HLD répond à un besoin précis de la couche d'orchestration :

| Brique | Rôle dans l'orchestration | Pourquoi ce choix (arbitrage HLD) |
|---|---|---|
| RKE2 | Distribution Kubernetes durcie CIS | Maîtrise d'Ubuntu 24.04 et compatibilité directe avec les playbooks Ansible existants (Talos écarté). |
| Cilium | CNI eBPF, routage natif IPv6, `kube-proxy` replacement, LB IPAM, BGP, NetworkPolicies | Remplace à lui seul Canal (VXLAN), `kube-proxy` et MetalLB tout en gérant le peering BGP IPv6 avec OPNsense. |
| WireGuard (via Cilium) | Chiffrement transparent des paquets IPv6 entre nœuds Kubernetes | Protège le trafic inter-nœuds sans configuration manuelle de tunnels IPSec, avec clés dans `cilium-secrets`. |
| Multus CNI | Attachement d'une deuxième interface réseau (`macvlan` VLAN 790 OOB) sur des pods ciblés | Donne accès au réseau d'administration et de collecte uniquement aux rares pods autorisés (`vmagent`, `VictoriaLogs`, `ArgoCD`). |
| Longhorn V2 | Stockage bloc distribué en 3 réplicas sur les nœuds HCI | Retenu après le PoC du 22/10 face à Ceph : intégration directe, 3x réplication fluide sur 10 Gbps. |

À l'intérieur des deux clusters, l'organisation par namespaces traduit directement le découpage fonctionnel :

| Cluster | Namespace | Workloads et politique associée |
|---|---|---|
| Interne | `kube-system` / `cilium-secrets` | Plan de contrôle Cilium, CoreDNS, Multus, clés de chiffrement WireGuard. |
| Interne | `longhorn-system` | Opérateur, moteur SPDK V2 et volumes distribués Longhorn. |
| Interne | `auth-system` | Annuaire interne OpenLDAP (employés CaaScad uniquement). |
| Interne | `openbao-system` / `cert-manager` | PKI intermédiaire OpenBao (3 pods HA Raft, `mlock`), `cert-manager` et External Secrets Operator. |
| Interne | `observability` | `vmagent`, `VictoriaMetrics`, `VictoriaLogs`, `Grafana` et `fluent-bit`. |
| Interne | `gitops` / `portal-apps` / `print` | ArgoCD, Site Manager, Site vitrine, NetBox et serveur d'impression CUPS vers Classe 308. |
| Client | `kube-system` / `longhorn-system` / `monitoring` | Cilium, Multus, Longhorn V2 et agents locaux de collecte (`fluent-bit`, métriques). |
| Client | `tenant-<client>` | Un namespace par client avec `CiliumNetworkPolicy` en refus par défaut, `ResourceQuota`, `LimitRange` et Pod Security Standards `restricted`. |

## Comment ces briques s'articulent de bout en bout

Lorsqu'un flux entre ou sort d'un pod, la pile décrite dans le HLD (et précisée techniquement dans le LLD) suit une chaîne cohérente :

1. Au niveau du cluster, RKE2 délègue tout le plan de données réseau à Cilium en mode **Native Routing IPv6** (sans encapsulation VXLAN) et utilise `kube-vip` sur les masters pour maintenir la haute disponibilité des API Servers (port 6443) sur le réseau du cluster et sur le VLAN 790 OOB.
2. Entre deux nœuds d'un même cluster (par exemple un pod sur `cli-hci1` parlant à un pod sur `cli-hci4`, ou la réplication d'un service client), Cilium chiffre automatiquement les trames dans le noyau grâce à WireGuard intégré.
3. Quand un service interne comme `vmagent` ou `ArgoCD` doit interroger des machines hors du cluster (les VMs Ubuntu, les OPNsense ou l'API du Cluster Client), Multus lui injecte une patte réseau secondaire sur le VLAN 790. Tous les autres pods restent strictement cantonnés au plan réseau Kubernetes grâce aux `CiliumClusterwideHostPolicy` et au pare-feu `nftables` sur chaque VM.
4. Enfin, pour le stockage persistant (bases MariaDB/PostgreSQL, coffre OpenBao, VictoriaMetrics/VictoriaLogs), Longhorn V2 fournit par défaut des volumes blocs RWO répliqués trois fois sur les nœuds HCI, avec option RWX via NFSv4.
