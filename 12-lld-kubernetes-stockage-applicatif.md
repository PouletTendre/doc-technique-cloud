# Section technique Kubernetes, stockage distribué et couche applicative du LLD

## Périmètre couvert dans le document LLD

Dans le document source `LLD_CaaScad.docx`, les spécifications bas niveau correspondant à cette thématique couvrent la **section 4 (« Spécifications Kubernetes, stockage et multi-tenant »)** et la **section 5 (« Topologie applicative interne et chaîne GitOps »)**. L'objectif de cette fiche est de traduire chaque bloc de configuration technique du LLD en règles opérationnelles lisibles.

## 1. Configuration RKE2 1.32, Cilium 1.19 et kube-vip (LLD 4.1)

Les deux clusters utilisent **RKE2 v1.32.x** avec la même logique : retirer les composants réseau par défaut de Rancher pour confier tout le réseau et le routage à Cilium 1.19. Pour éviter toute ambiguïté lors d'un débogage, les plages virtuelles `Service CIDR` (ClusterIP, en ULA) diffèrent entre les deux clusters : `fd07:63fe:c5dd:101::/112` en Interne et `fd07:63fe:c5dd:102::/112` côté Client.

Voici ce que fixe le fichier `/etc/rancher/rke2/config.yaml` sur un master (exemple de `cli-m1`) :

- `node-ip: "2001:678:3fc:bb23::1"` et `cluster-cidr: "2001:678:3fc:bb10::/60"` : le master s'enregistre avec son IP du VLAN 769 et alloue aux pods le `/60` Client.
- `cni: none`, `disable-kube-proxy: true`, `disable: [rke2-ingress-nginx]` : empêche RKE2 d'installer Canal, `kube-proxy` et Nginx Ingress, qui entreraient en conflit avec Cilium et Envoy.
- `secrets-encryption: true` : active le chiffrement au repos des objets `Secret` dans `etcd`.
- `tls-san` : inscrit dans le certificat de l'API Server le nom DNS (`api.cli.caascad.internal`) et les deux VIP (`2001:678:3fc:bb23::10` pour les nœuds, `fd07:63fe:c5dd:1:2::10` pour l'OOB).
- `node-taint: ["node-role.kubernetes.io/control-plane:NoSchedule"]` : réserve les masters au plan de contrôle.
- `etcd-snapshot-schedule-cron: "0 */6 * * *"` et `etcd-snapshot-retention: 12` : produit un snapshot toutes les six heures en gardant les 12 derniers clichés (3 jours en local).

En parallèle, `kube-vip` tourne en pod statique (`/var/lib/rancher/rke2/agent/pod-manifests/`) sur les 3 masters de chaque cluster et maintient deux VIP NDP : la VIP sur le VLAN des nœuds (consommée par `kubelet` et `k8sServiceHost` dans Cilium) et la VIP sur le VLAN 790 OOB (consommée par `kubectl` depuis le bastion, `runner-infra` et ArgoCD).

Côté Helm Cilium 1.19, les valeurs de déploiement activent l'IPv6 natif (`ipv4.enabled: false`, `routingMode: native`, `ipv6NativeRoutingCIDR: "2001:678:3fc:bb00::/56"`, `autoDirectNodeRoutes: true`), remplacent `kube-proxy` (`kubeProxyReplacement: true`), chiffrent le trafic inter-nœuds par WireGuard (`encryption.type: wireguard`), activent BGP (`bgpControlPlane.enabled: true`), activent Gateway API et Ingress Envoy en mode VIP partagée (`loadbalancerMode: shared`), ainsi que Hubble pour la télémétrie réseau (`dns, drop, tcp, flow, policy`).

## 2. Configuration Longhorn V2 sur les 7 nœuds HCI (LLD 4.2)

Le LLD marque ces paramètres `[à vérifier]` jusqu'à la validation finale sur les machines déployées, car les réglages SPDK modifient directement le comportement du kubelet.

Sur chaque nœud HCI, `/dev/vda` (60 Go ext4) héberge Ubuntu et `containerd`, tandis que `/dev/vdb` (200 Go sans partition) sert de périphérique bloc brut SPDK dans l'objet `Node` Longhorn. La capacité utile en réplication par 3 atteint environ 200 Go en Interne (3 HCI) et environ 266 Go bruts côté Client (4 HCI).

Pour préparer le noyau et isoler les ressources CPU, `/etc/sysctl.d/99-longhorn-v2.conf` réserve `vm.nr_hugepages = 1024` (2 Go en pages de 2 Mo), `/etc/modules-load.d/longhorn.conf` charge `nvme_tcp`, et le kubelet reçoit `cpu-manager-policy=static` avec `system-reserved=cpu=1,memory=2Gi` afin de réserver 1 cœur vCPU dédié au processus `spdk_tgt` et d'éviter qu'il ne manque de CPU lors des pics applicatifs. Longhorn active ensuite `v2DataEngine`, `defaultReplicaCount: 3`, `replicaSoftAntiAffinity: false` (impose 3 nœuds distincts) et `replica-rebuild-bandwidth-limit` pour ne pas saturer le lien 10 Gbps en cas de reconstruction.

Deux `StorageClass` sont déclarées : `longhorn-v2` (par défaut, `dataEngine: v2`, `numberOfReplicas: 3`, `dataLocality: best-effort`, `volumeBindingMode: WaitForFirstConsumer`, `fsType: ext4`) pour les volumes blocs RWO, et `longhorn-v2-rwx` avec `nfsOptions: "vers=4.1,noresvport"` lorsque plusieurs pods doivent monter un même volume en lecture/écriture.

## 3. Gabarits et isolation du Cluster Client (LLD 4.3)

Après réservation des ressources système, SPDK et HugePages (environ 4 vCPU et 12 Go de RAM consommés sur l'ensemble des 4 nœuds HCI Client), il reste environ **28 vCPU, 52 Go de RAM et 240 Go de PVC commercialisables**. Chaque client reçoit un namespace `tenant-<client>` dimensionné selon trois offres :

| Gabarit | CPU (`requests` / `limits`) | RAM (`requests` / `limits`) | Stockage (`requests.storage`) | Max pods | `LimitRange` défaut par conteneur |
|---|---|---|---|---|---|
| Starter | 2 / 2 | 4 Gi / 4 Gi | 20 Gi | 10 | req `100m` / `128Mi`, lim `500m` / `512Mi` |
| Standard | 4 / 4 | 8 Gi / 8 Gi | 50 Gi | 25 | req `200m` / `256Mi`, lim `1000m` / `1Gi` |
| Pro | 8 / 8 | 16 Gi / 16 Gi | 100 Gi | 50 | req `250m` / `256Mi`, lim `2000m` / `2Gi` |

Chaque namespace client inclut automatiquement les labels Pod Security Standards au niveau `restricted`, le `ServiceAccount` `default` avec `automountServiceAccountToken: false` et la `CiliumNetworkPolicy` `default-tenant-isolation`. Tous les pods héritent du profil Seccomp `RuntimeDefault` et d'AppArmor sur l'hôte (le sandboxing `gVisor` reste écarté du déploiement actuel).

## 4. Cartographie des workloads du Cluster Interne et chaîne GitOps (LLD 5.1 et 5.2)

Le Cluster Interne exécute les 13 briques applicatives de l'infrastructure dans des namespaces dédiés :

| Namespace | Workload | Type / Réplicas | Stockage Longhorn V2 | Port et exposition |
|---|---|---|---|---|
| `auth-system` | `openldap` | StatefulSet / 1 | 5 Gi | TCP 636 (LDAPS) sur VIP `bb21::2` (`ldap.caascad.internal`) |
| `openbao-system` | `openbao` | StatefulSet / 3 (Raft) | 3 x 5 Gi | HTTPS 8200 sur `bb21::1` (`bao.caascad.internal`), Raft 8201 |
| `openbao-system` | `cert-manager` + `external-secrets` | Deployments / 1 à 2 | Stateless | Auth K8s interne vers OpenBao |
| `observability` | `victoriametrics` | StatefulSet / 1 | 50 Gi | TCP 8428 via `bb21::1` (rétention 6 mois) |
| `observability` | `victorialogs` | StatefulSet / 1 | 60 Gi | TCP 9428 via `bb21::1` + Multus OOB `1:5::2` (rétention 90 j) |
| `observability` | `vmagent` | Deployment / 1 | 5 Gi buffer | Multus OOB `1:5::1` (scrape toutes les 30 s) |
| `observability` | `grafana` | Deployment / 2 | 5 Gi | TCP 3000 -> HTTPS `bb21::1` (`grafana.caascad.internal`) |
| `observability` | `fluent-bit` | DaemonSet (2 clusters) | `hostPath /var/log` (RO) | Envoi HTTP JSON vers `1:5::2:9428` |
| `gitops` | `argocd` | Deployments / 2 | Stateless | HTTPS `bb21::1` (`argocd.caascad.internal`) + Multus OOB `1:5::3` |
| `portal-apps` | `site-vitrine` + `mariadb` | Deploy 1-2 / STS 1 | 10 Gi + 10 Gi | HTTPS `bb22::1` (tunnel Cloudflare) |
| `portal-apps` | `site-manager` + `mariadb` | Deploy 2 / STS 1 | 10 Gi | HTTPS `bb22::1` (tunnel Cloudflare) |
| `portal-apps` | `netbox` + `postgres` + `valkey` | Deploy 2 / STS 1 / STS 1 | 10 Gi + 5 Gi | HTTPS `bb21::1` (`netbox.caascad.internal`) |
| `print` | `cups` | Deployment / 1 | 2 Gi | TCP 631 (IPP/TLS) sur `bb21::3` (`cups.caascad.internal`) |

La chaîne GitOps décrite en section 5.2 du LLD garantit qu'aucun composant public ne touche au VLAN 790 OOB : Site Manager valide le gabarit et committe les manifests YAML dans `gitlab.inetdoc.net` (via un jeton géré par OpenBao et External Secrets Operator), ArgoCD tire ces manifests et les pousse sur le Cluster Client depuis `1:5::3` vers `1:2::10:6443`, pendant que Site Manager publie l'URL sur Cloudflare (`bb24::1`). Côté runners, `runner-ci` (`bb43::2`, sans OOB) compile les images et passe les tests, tandis que `runner-infra` (`bb43::1` et OOB `1:4::3`, limité à la branche protégée `main`) exécute Ansible et OpenTofu sur l'infrastructure.
