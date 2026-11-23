# Bastion Apache Guacamole

## Ce qu'est Apache Guacamole

Apache Guacamole est une passerelle d'administration sans client lourd (un bastion web). Un administrateur se connecte depuis un simple navigateur HTTPS sur une interface web qui lui ouvre des terminaux SSH, des sessions RDP ou VNC vers les serveurs internes, sans installer de client VPN ni ouvrir directement le port SSH de chaque machine sur Internet.

Sur le plan technique, aucun flux SSH ou RDP ne traverse le poste de l'utilisateur. Le navigateur échange uniquement des trames HTTPS et WebSocket (protocole Guacamole) avec le serveur bastion, qui initie ensuite les vraies connexions d'administration vers les machines cibles sur le réseau interne.

## Comment fonctionne Apache Guacamole

L'architecture d'Apache Guacamole s'appuie sur deux briques principales et des extensions d'authentification :

- Le démon natif `guacd` établit les connexions réseau réelles (SSH, RDP, VNC) vers les serveurs cibles, traduit l'affichage en flux graphique ou terminal optimisé et peut enregistrer l'intégralité de la session (frappes clavier et sorties terminal au format typescript ou vidéo).
- L'application web Java (Tomcat) sert l'interface HTML5, vérifie l'identité de l'utilisateur et consulte la liste des machines auxquelles cet utilisateur a le droit d'accéder.
- L'authentification et les permissions peuvent être séparées entre un annuaire d'entreprise (LDAP) pour vérifier le mot de passe et une base de données (PostgreSQL) qui stocke les définitions de serveurs, les clés SSH privées et les secrets TOTP de double facteur.

## Articulation dans le projet CaaScad

Dans CaaScad, Guacamole est le point de passage obligé pour toute intervention humaine sur les machines virtuelles, les pare-feux ou les clusters Kubernetes. Aucun administrateur ne possède d'accès SSH direct depuis son poste en Classe 308 ou depuis l'extérieur.

### Emplacement réseau et tunnel sortant

La VM `bastion` (2 vCPU, 4 Go de RAM, 40 Go de disque sous Ubuntu 24.04 LTS) est hébergée sur l'hyperviseur `oscar.infra.stri`. Elle porte deux interfaces réseau :

- Une interface de service sur le VLAN 760 (`SRV_BASTION`, `2001:678:3fc:bb40::1/64`), sur laquelle tourne le connecteur `cloudflared`. Comme le pare-feu de l'université bloque les connexions entrantes, `cloudflared` initie un tunnel chiffré sortant permanent (UDP/TCP 7844 et TCP 443) vers le bord Cloudflare et relaie localement le trafic web HTTPS vers Guacamole sur `127.0.0.1:8080`.
- Une interface d'administration sur le VLAN 790 OOB (`fd07:63fe:c5dd:1:4::4/64`), utilisée pour ouvrir les sessions SSH et HTTPS d'administration vers les VMs, les VIP API Kubernetes (`1:1::10:6443`, `1:2::10:6443`) et l'interface web des pare-feux OPNsense (`1:3::1`, `1:3::2`).

### Authentification, clés SSH et journalisation

Guacamole assemble trois extensions : `auth-ldap`, `auth-jdbc-postgresql` et `auth-totp`. Lorsqu'un administrateur se connecte, Guacamole interroge OpenLDAP sur le cluster interne en LDAPS (`ldaps://ldap.caascad.internal:636`, flux `F11` sur la VIP `bb21::2`) et restreint l'accès aux membres du groupe `cn=infra`. Le second facteur TOTP est vérifié localement via PostgreSQL, qui conserve également les profils de connexion.

Sur toutes les VMs Ubuntu (y compris les masters et nœuds HCI), le fichier `/etc/ssh/sshd_config.d/99-hardening.conf` applique un durcissement strict aligné dans le LLD (contrairement à la rédaction initiale du HLD qui évoquait encore des connexions directes en `root`) :

- `ListenAddress` limité à l'adresse OOB locale (`fd07:63fe:c5dd:1:...`).
- `PermitRootLogin no` et `PasswordAuthentication no`.
- `AllowUsers caascad-admin` avec authentification par clé publique uniquement.

Guacamole détient une clé privée Ed25519 dédiée pour ouvrir les sessions sous le compte `caascad-admin`. Une fois connecté, l'administrateur passe en privilégié via `sudo`, qui journalise toutes les commandes et sorties dans `/var/log/sudo.log`. En parallèle, `guacd` enregistre la session SSH côté bastion. Les agents Fluent Bit expédient `/var/log/auth.log`, `/var/log/sudo.log` et les journaux Guacamole vers VictoriaLogs (`1:5::2:9428`).

### Accès d'urgence Break-Glass et tolérance aux pannes

La VM `bastion` reste un point de défaillance unique assumé dans cette version de l'architecture : si l'hyperviseur Oscar tombe, l'accès distant via Guacamole devient indisponible. Pour ne pas perdre le contrôle de l'infrastructure en cas d'incident sur Oscar ou sur OpenLDAP, chaque VM dispose d'un compte local `breakglass`. Ce compte est volontairement exclu de `AllowUsers` SSH : il ne fonctionne que depuis la console physique ou KVM locale en salle U3-316. Son mot de passe est rangé dans le coffre KeePassXC hors ligne de l'équipe infrastructure, et toute ouverture de session PAM sur ce compte déclenche immédiatement une alerte critique dans VictoriaLogs et Grafana. En cas de perte définitive du bastion, la VM est redéployée par Ansible puis restaurée depuis le dump PostgreSQL nocturne sauvegardé par BorgBackup.
