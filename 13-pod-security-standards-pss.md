# Pod Security Standards (PSS)

## Ce que sont les Pod Security Standards

Les Pod Security Standards (PSS) définissent le niveau de privilèges système qu'un pod a le droit de demander au noyau Linux du nœud Kubernetes sur lequel il s'exécute. Le contrôleur natif de Kubernetes chargé de faire respecter ces règles à la création de chaque pod s'appelle Pod Security Admission (PSA), qui a remplacé l'ancien mécanisme PodSecurityPolicy (supprimé depuis Kubernetes 1.25).

Sans PSS, n'importe quel utilisateur autorisé à créer un Pod dans un namespace pourrait lancer un conteneur en `root`, activer le mode `privileged: true`, monter le disque du serveur hôte via `hostPath` et prendre le contrôle complet de la machine virtuelle sous-jacente.

## Comment fonctionnent les profils et modes PSS

PSS s'appuie sur trois profils cumulatifs et trois modes d'application configurés directement par des labels sur chaque namespace Kubernetes :

### Les trois profils de sécurité

| Profil | Ce qu'il autorise ou bloque | Usage typique |
|---|---|---|
| `privileged` | Aucune restriction (accès complet aux privilèges hôtes et noyau). | Agents système d'infrastructure (`cilium`, `longhorn`, `fluent-bit`). |
| `baseline` | Bloque les escalades connues (`privileged`, `hostNetwork`, `hostPID`, `hostIPC`, `hostPath`, ajout de capacités Linux dangereuses). | Workloads standards tolérant encore un utilisateur `root` dans le conteneur. |
| `restricted` | Applique `baseline` et impose en plus le durcissement maximal du conteneur. | Workloads multi-tenants non fiables (conteneurs clients). |

### Ce qu'impose concrètement le profil `restricted`

Lorsqu'un namespace porte le profil `restricted`, l'API Kubernetes refuse tout pod dont la spécification ne respecte pas ces cinq exigences :

- Le conteneur ne peut pas tourner en `root` (`runAsNonRoot: true` exigé et `runAsUser` différent de `0`).
- Aucune élévation de privilèges n'est possible (`allowPrivilegeEscalation: false` obligatoire sur tous les conteneurs et initContainers).
- Toutes les capacités Linux doivent être retirées via `capabilities.drop: ["ALL"]` (seule l'ajout de `NET_BIND_SERVICE` est toléré si nécessaire).
- Le profil Seccomp `seccompProfile.type` doit valoir `RuntimeDefault` ou `Localhost` pour filtrer les appels système dangereux vers le noyau.
- Seuls les types de volumes sûrs (`emptyDir`, `configMap`, `secret`, `persistentVolumeClaim`, `downwardAPI`, `projected`) sont admis ; `hostPath` est bloqué.

### Les trois modes d'application

Pour chaque niveau (`enforce`, `audit`, `warn`), un label est placé sur le namespace :

- `enforce` : rejette purement et simplement la création du pod non conforme.
- `audit` : laisse passer mais inscrit une annotation d'alerte dans les journaux d'audit de l'API Kubernetes.
- `warn` : affiche un avertissement à l'utilisateur ou à l'outil lors de la soumission du manifest.

## Articulation dans le projet CaaScad

Sur le Cluster Client de CaaScad, les conteneurs de différents clients cohabitent sur les quatre mêmes nœuds HCI (`cli-hci1` à `cli-hci4`) et partagent donc le noyau Linux de ces VMs. PSS forme la première barrière d'isolation à l'intérieur du namespace de chaque client (`tenant-<client>`).

### Configuration appliquée dans le template client

Chaque namespace généré par Site Manager et déployé par ArgoCD reçoit immédiatement les trois modes au niveau `restricted` :

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-exemple
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

En complément immédiat de PSS dans le même gabarit de provisionnement :

- Le `ServiceAccount` `default` du namespace porte `automountServiceAccountToken: false`. Même si un attaquant prend le contrôle du processus applicatif à l'intérieur d'un pod client, il ne trouve aucun jeton monté dans `/var/run/secrets/kubernetes.io/serviceaccount` pour tenter d'interroger l'API Kubernetes.
- Les profils **Seccomp `RuntimeDefault`** et **AppArmor** sont actifs au niveau des nœuds Ubuntu 24.04 LTS.
- `ResourceQuota` et `LimitRange` (Starter, Standard ou Pro) bornent CPU, RAM, stockage et nombre maximal de pods afin qu'un client ne puisse pas épuiser les ressources d'un nœud HCI par déni de service.

### Limite connue et défense en profondeur

PSS empêche les erreurs de configuration classiques et les escalades triviales, mais il ne supprime pas le partage du noyau Linux entre pods d'un même nœud HCI. Comme les hyperviseurs KVM de la salle U3-316 n'ont pas la virtualisation imbriquée activée, CaaScad ne peut pas isoler chaque pod client dans une micro-VM avec Kata Containers.

C'est pour couvrir cette limite résiduelle que l'isolation client empile plusieurs couches indépendantes (voir la fiche sur `CiliumNetworkPolicy` et le schéma multi-tenant) : cloisonnement réseau eBPF (`CiliumNetworkPolicy`), verrouillage de l'OOB sur l'hôte (`nftables` et `CiliumClusterwideHostPolicy`), séparation physique des VMs entre Cluster Client et Cluster Interne, et blocage périmétrique `F17` sur OPNsense. L'ajout d'un runtime en espace utilisateur comme `gVisor (runsc)` est identifié dans le HLD et le LLD comme renfort futur possible (actuellement laissé hors déploiement pour préserver les performances sur les appels système et les E/S).
