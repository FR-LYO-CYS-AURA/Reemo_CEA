# Reemo 2.4.1 — Dossier de déploiement client

> **Statut** : analyse technique terminée, croisée avec la documentation éditeur. Plan de
> déploiement établi, décisions en attente de validation.
> **Source analysée** : `reemo-infra-2.4.1/` (rôle Ansible, livrable éditeur, non modifié)
> **Version applicative** : `PORTAL_GLOBAL_VERSION: 2.4.1` (`defaults/main.yml:735`)
> **Documentation éditeur** : <https://doc-private.reemo.io/2.4.1/> — modèle de référence retenu :
> [Bastion+ with URL](https://doc-private.reemo.io/installs/bastion_websocket_url.html)

---

## Sommaire

0. [Sources et référentiel](#0-sources-et-référentiel)
1. [Ce qu'est ce paquet](#1-ce-quest-ce-paquet)
2. [Architecture de la solution](#2-architecture-de-la-solution)
3. [Les deux mécanismes de déploiement](#3-les-deux-mécanismes-de-déploiement)
4. [Cible retenue pour ce client](#4-cible-retenue-pour-ce-client)
5. [Prérequis](#5-prérequis)
6. [Le sujet critique : installation déconnectée](#6-le-sujet-critique--installation-déconnectée)
7. [Alertes sécurité](#7-alertes-sécurité)
8. [Bugs et pièges du rôle](#8-bugs-et-pièges-du-rôle)
9. [Plan de déploiement](#9-plan-de-déploiement)
10. [Décisions à trancher](#10-décisions-à-trancher)
11. [Annexes](#11-annexes)

---

## 0. Sources et référentiel

### 0.1 Documents de référence

| Source | Usage |
|---|---|
| `reemo-infra-2.4.1/` (code) | Analyse technique, vérification ligne à ligne |
| [Bastion+ with URL](https://doc-private.reemo.io/installs/bastion_websocket_url.html) | **Modèle d'architecture retenu** — topologie, inventaire, playbook, sizing, matrice de flux |
| [INFRA — Registry and Images](https://doc-private.reemo.io/infra/registry.html) | Modes Online / Proxy / Offline pour les images d'infrastructure |
| [PROVISION — User Container Images](https://doc-private.reemo.io/provision/registry.html) | Modes Online / Proxy / Offline pour les **images de session** |
| [INFRA — Prerequisites](https://doc-private.reemo.io/infra/prerequisites.html) | Prérequis système |
| [Licensing](https://doc-private.reemo.io/licensing.html) | À consulter pour la décision D6 |

### 0.2 Corrections apportées par la documentation éditeur

L'analyse initiale du code seul comportait plusieurs approximations, corrigées ici :

| # | Point | Analyse code seule | **Corrigé (doc éditeur)** |
|---|---|---|---|
| C1 | `REGISTRY_ENV` | `"reemo"` | **`"reemoinfra"`** pour l'infra — et **`"reemosb"`** pour les images de session |
| C2 | Exécution multi-groupes | Une passe `--limit` par groupe | **Plusieurs *plays* dans un même playbook** — c'est la solution officielle |
| C3 | Ordre de déploiement | provision → relayws → infra | **infra → relayws → provision** (ordre officiel) |
| C4 | Matrice de flux | Unidirectionnelle infra → zones | **Flux retour PROVISION → INFRA (8443, 8446) et PROVISION → RELAY (443)** |
| C5 | `INIT_PROVISION` / `INIT_RELAYWS` | Clé `ip` seule | Champs `type`, `url`, `relayws` (association provision ↔ relais) |
| C6 | Image de session en offline | Présenté comme un bug | **Étape manuelle documentée** par l'éditeur (Scénario A) — reste une lacune d'automatisation |
| C7 | Registry local | Recommandation personnelle | **Explicitement recommandé par l'éditeur** (Scénario B) |
| C8 | Hôtes registry | `registry.reemo.io` | **`registry.reemo.io` + `registry-auth.reemo.io`** |

---

## 1. Ce qu'est ce paquet

`reemo-infra-2.4.1` est un **rôle Ansible unique** (pas un playbook complet) qui déploie la
plateforme **Reemo** — solution de bureau/navigateur distant sécurisé (VDI / Remote Browser
Isolation / bastion) — sur un ou plusieurs **clusters Docker Swarm**.

### Contenu

| Élément | Volume | Rôle |
|---|---|---|
| `defaults/main.yml` | 1429 lignes | **Le** fichier de configuration — ~900 variables |
| `tasks/` | ~90 fichiers | Un fichier par composant + orchestrateur `main.yml` (807 lignes) |
| `files/ssl/` | 25 certs + 25 clés | Certificats de **démonstration** — voir §7 |
| `files/nginx/` | 1 fichier | Proxy mTLS devant le socket Docker |
| `files/container_json/` | `chromium.json` | Catalogue d'image de session injecté en base |
| `templates/` | 3 fichiers | `vault.hcl`, `openbao.hcl`, `rsyslog_datadog.conf` |
| `library/` | `iptables_raw.py` | Module communautaire de gestion iptables |
| `README.md` (du rôle) | 212 lignes | **Changelog uniquement** |

### Ce qui manque et qu'il faut produire

Le paquet ne contient **aucune documentation d'installation, aucun playbook, aucun inventaire
d'exemple**. Le seul `tests/test.yml` est un squelette trivial référençant un rôle qui n'existe
même pas (`reemo-cata-standalone`).

Tout le contrat de déploiement est **implicite dans les noms de groupes d'inventaire**.

---

## 2. Architecture de la solution

### 2.1 Principe fondamental

**Un seul rôle, appliqué à toutes les machines.** Ce qui s'installe sur un serveur dépend
uniquement de son **appartenance aux groupes d'inventaire** :

```yaml
when: "'infra_manager' in group_names"      # → API, portal, DB, Traefik, TURN...
when: "'relayws_manager' in group_names"    # → Swarm relais + nginx mTLS
when: "'provision' in group_names"          # → Swarm provision + nginx mTLS
```

### 2.2 Groupes d'inventaire reconnus

| Groupe | Rôle | Swarm |
|---|---|---|
| `infra_manager` | **Tout-en-un** : API + Portal + DB + Traefik | Swarm #1 |
| `api_manager` | Mode éclaté : back-end | Swarm #1a |
| `portal_manager` | Mode éclaté : front | Swarm #1b |
| `relayws_manager` | Relais WebSocket | Swarm #2 |
| `provision` + `provision_manager` / `provision_worker` | Ferme de conteneurs de session | Swarm #3 |
| `provisionN_*` (`provision1`, `provision2`…) | N fermes supplémentaires (multi-DC) | Swarm #3+N |

Liste exhaustive des noms reconnus par le code :

```
infra_manager
api_manager
portal_manager
relayws_manager
provision            provision1   provision2   … provisionN
provision_manager    provision1_manager  … provisionN_manager
provision_worker     provision1_worker   … provisionN_worker
<VAULT_HOST_GROUP>   (variable, défaut = infra_manager | api_manager)
```

> **Il n'existe aucun groupe `db*`.** Le placement de la base se fait par contrainte Swarm sur le
> **hostname du nœud** via `MYSQL_NODE_HOSTNAME_DB1/2/3` (`defaults:275-277`).

> **`relayws1_manager`, `relayws2_manager` ne sont pas supportés** par `tasks/main.yml` (seul
> `nginx_redhat/ubuntu` connaît la regex). Le multi-relais passe par `INIT_RELAYWS` et des
> exécutions distinctes.

### 2.3 Règle absolue : `HMACSECRET`

| Mode | Valeur | Contrôle |
|---|---|---|
| All-in-one (`infra_manager`) | **vide** | `tasks/main.yml:76-81` — échec si non vide |
| Éclaté (`api_manager` + `portal_manager`) | **non vide** | `tasks/main.yml:69-74` — échec si vide |

C'est le secret partagé permettant à Portal et API, dans deux Swarms distincts, de s'authentifier.

### 2.4 Composants déployés

| Bloc | Services |
|---|---|
| **Cœur** | `api` (métier, auth, orgas, licences) · `portal` (front web) · `portaladmin` (front admin séparé) · `signal` (signalisation WebRTC) · `db` (job de migration de schéma) |
| **Provisioning** | `proapi` (pilote les Swarms de sessions) · `prorelayapi` (crée les relais WS) · `procloudapi` (cloud) · HAProxy en front |
| **Données** | MariaDB 11.4 (défaut, mTLS) · MySQL 8 (legacy) · NDB Cluster 3 nœuds (HA) · DB externe |
| **Journalisation** | `logapi` + `logapidb` + `logapicron` (base `reemo_log`) |
| **Secrets** | `vault` (OpenBao) + `credentialapi` + `credentialportal` |
| **Passerelles** | `workstation` (postes physiques) · `applianceapi`/`applianceportal` · `wolapi` (Wake-on-LAN) |
| **Réseau** | `traefik` v3 rootless (un par Swarm) · `turn` (coturn) · `exim4` (SMTP) |
| **Cron** | 7 jobs : ghost containers, LDAP sync, purge logs, security logs, workstations, cloud |

Toute la communication interne est en **mTLS + HMAC partagé**.

### 2.5 Réseaux overlay créés

| Créé par | Réseaux |
|---|---|
| `network_portal.yml` (portal/infra) | `internet`, `{{INSTANCE_NAME}}_internal`, `0net` (si `INSTANCE_NAME != reemo`), `reemo_turn` (si TURN), `reemo_appliance` (si appliance) |
| `network_api.yml` (api/infra) | `reemo_internal`, `reemo_mysql`, `reemo_proapi`, `reemo_procloudapi`, `reemo_vault`, `reemo_credential`, `reemo_smtp` |
| `network_relayws.yml` (relayws) | `external` |
| `proapi.yml` | `reemo_provision` |
| `prorelayapi.yml` | `reemo_relayws` |
| `mysqlcluster.yml` | `reemo_mysqlndb` |

---

## 3. Les deux mécanismes de déploiement

C'est **le point d'architecture le plus important à comprendre**.

### Mécanisme 1 — Ansible en SSH : la *préparation*

Ansible se connecte en SSH sur chaque machine, y compris provision et relayws. Sur ces zones, il
ne fait que **préparer le terrain** :

- installer Docker, initialiser un Swarm **vide et indépendant** ;
- poser un **nginx mTLS sur le port 8443** qui proxifie `/var/run/docker.sock` ;
- déposer les certificats de la PKI, configurer firewalld/iptables ;
- pré-télécharger les images de session (`warmup`).

**Ansible ne déploie aucun service Reemo sur provision ni relayws.**

### Mécanisme 2 — Runtime : proapi/prorelayapi pilotent les zones

Une fois l'infra en place, ce sont **les conteneurs applicatifs** qui déploient à distance, via
l'**API Docker Remote** — pas Ansible.

```
[ Swarm INFRA ]
   proapi (conteneur)
      │  résout "reemo_provision" sur l'overlay reemo_provision
      ▼
   HAProxy "reemo_provision"           ← tasks/proapi.yml:373-428
      │  backends = liste d'IP encodée base64 (HAPROXY_BACKEND_IP)
      │  https://<ip-provision>:8443
      ▼
════════ franchit la frontière réseau ════════
      ▼
[ Zone PROVISION ]
   nginx :8443  ── vérifie ssl_client_s_dn == "CN=reemo_proapi" ──► sinon 403
      │
      ▼
   /var/run/docker.sock  ──► docker crée le conteneur de session (Chromium/RBI)
```

Identique pour relayws : `prorelayapi` → HAProxy `reemo_relayws` → nginx (`CN=reemo_prorelayapi`)
→ conteneurs `ws-relay-server` (auto-destruction après 60 s).

### Points clés du mécanisme 2

| Aspect | Valeur |
|---|---|
| **Fédération Swarm** | **Aucune.** Les Swarms sont étanches, ils ne communiquent que par appels HTTPS à l'API Docker |
| **Flux requis** | `infra → zone : 8443/tcp` uniquement |
| **Authentification** | Certificat client, DN vérifié par nginx (`CN=reemo_proapi` / `CN=reemo_prorelayapi`) |
| **Temporalité** | **Runtime**, à chaque session utilisateur — pas au déploiement |
| **Dépendance réseau** | **Le nœud de provision doit joindre un registry au moment de créer la session** |

### Déclaration des zones à l'application

Via `INIT_PROVISION` / `INIT_RELAYWS`, qui alimentent deux choses :

1. **Les backends HAProxy** (`tasks/proapi.yml:409`) — chaque clé du dictionnaire génère un
   service HAProxy dédié nommé `reemo_<clé>` :

```yaml
INIT_PROVISION:
  provision:  { ip: ["10.0.0.20:8443"] }
  provision2: { ip: ["10.30.4.11:8443", "10.30.4.12:8443"] }
```

2. **La configuration en base** (`tasks/db.yml:187-200`), au moment de `INIT_DB` :

```yaml
url: 'https://reemo_provision:8443/v1.50/'
servername: 'reemo_provision'     # doit matcher le CN du certificat
```

> **Ajout d'une zone a posteriori** : ajouter l'entrée dans `INIT_PROVISION`, jouer
> `--limit provisionN`, puis `--tags proapi` sur infra. Pas besoin de rejouer `INIT_DB`.

---

## 4. Cible retenue pour ce client

**Modèle éditeur : « Bastion+ with URL »** — all-in-one, adapté en **site déconnecté** avec
registry local, TURN activé.

| Machine | Groupe(s) | Rôle | Ports entrants |
|---|---|---|---|
| `infra_manager1` | `infra_manager` | API + Portal + Signal + MariaDB + Traefik + TURN + proapi/prorelayapi + crons | 443, 8443, 8446, 58200 tcp+udp |
| `provision1_manager1` | `provision`, `provision1_manager` | Ferme de conteneurs de session | 8443 |
| `relayws_manager1` | `relayws_manager` | Relais WebSocket | 443, 8443 |
| `registry1` | *(hors rôle Ansible)* | Registry Docker local | 443 |

### 4.1 Sizing officiel (10 sessions simultanées)

Source : documentation éditeur, page *Bastion+ with URL*.

| Machine | CPU | RAM | Disque |
|---|---|---|---|
| `infra_manager1` | 4 vCPU | 8 Go | 100 Go SSD |
| `provision1_manager1` | **10 vCPU** | **10 Go** | 100 Go SSD |
| `relayws_manager1` | 4 vCPU | 4 Go | 50 Go SSD |
| `registry1` *(ajout)* | 2 vCPU | 4 Go | **100 Go** |

> Le nœud de provision est de loin le plus sollicité : c'est lui qui exécute les conteneurs de
> session. Dimensionner à la hausse selon le nombre de sessions simultanées visé.

### 4.2 Configuration retenue

```yaml
HMACSECRET: ""              # OBLIGATOIREMENT vide en all-in-one
DB_DIALECT: "mysql"         # → MariaDB 11.4 avec mTLS
TURN_ENABLED: true
LOGAPI_ENABLED: true        # activé par défaut
CRON_ENABLE: "true"         # activé par défaut
INITCA_ENABLE: "true"       # NON NÉGOCIABLE (voir §7)
```

Modules **non retenus** : Vault/Credential, Workstation, Appliance, ProCloudAPI, WOLAPI,
Maintenance, PortalAdmin séparé, sauvegarde DB, SMTP.

> **Écart avec le modèle éditeur** : l'exemple officiel active `CREDENTIAL_ENABLED`,
> `CREDENTIALPORTAL_ENABLED` et `VAULT_ENABLED`, et déclare un `PORTALADMIN_URL` distinct filtré
> par IP (`PORTALADMIN_URL_RESTRICT_IP`, `PORTALADMIN_TYPE: "instadmin"`,
> `PORTAL_TYPE: "orguser"`). Ces options sont écartées ici — **à confirmer avec le client**
> (décisions D13 et D14).

> **À noter** : sans SMTP (`API_MAIL_ACTIVE: false`), `tasks/db.yml:3-11` désactive
> automatiquement la vérification d'email, la 2FA email et les notifications d'invitation.

### 4.3 Matrice de flux

Reprise de la matrice officielle, **adaptée au registry local**. Les IP sont celles de l'exemple
éditeur, à remplacer par le plan d'adressage client.

| Description | Transport | Source | Port source | Destination | Port dest. |
|---|---|---|---|---|---|
| **INTERNET → INFRA** | TCP | WAN | 1024:65535 | infra | **443** |
| **INTERNET → RELAY** | TCP | WAN | 1024:65535 | relayws | **443** |
| **INFRA → PROVISION** | TCP | infra | 1024:65535 | provision | **8443** |
| **INFRA → RELAY** | TCP | infra | 1024:65535 | relayws | **443, 8443** |
| **PROVISION → INFRA** | TCP | provision | 1024:65535 | infra | **8443, 8446** |
| **PROVISION → RELAY** | TCP | provision | 1024:65535 | relayws | **443** |
| **INFRA → REGISTRY** | TCP | infra | 1024:65535 | registry local | **443** |
| **PROVISION → REGISTRY** | TCP | provision | 1024:65535 | registry local | **443** |
| **RELAY → REGISTRY** | TCP | relayws | 1024:65535 | registry local | **443** |
| Clients → TURN | TCP+UDP | WAN/LAN | — | infra | **58200** |
| Intra-Swarm | TCP/UDP | chaque Swarm | — | lui-même | 2377/tcp, 7946/tcp+udp, 4789/udp |

> ⚠ **Flux retour souvent oubliés** — ils n'apparaissent pas dans l'analyse du rôle seul :
> - **PROVISION → INFRA:8443** — les conteneurs de session joignent le **signal** (WebRTC)
> - **PROVISION → INFRA:8446** — entrypoint **credentialportal** (si Credential activé)
> - **PROVISION → RELAY:443** — les conteneurs de session joignent directement le **relais WebSocket**
>
> Sans ces règles, le déploiement réussit mais **les sessions ne s'établissent pas**.

> En mode connecté, l'éditeur exige l'ouverture vers **`registry.reemo.io` ET
> `registry-auth.reemo.io`** (port 443) depuis les trois nœuds. En mode déconnecté, ces flux sont
> remplacés par l'accès au registry local.

---

## 5. Prérequis

### 5.1 Système

| Élément | Exigence |
|---|---|
| **OS** | Ubuntu, RedHat ou Rocky **uniquement**. Debian pur non supporté |
| Rocky 9 | Contrôle bloquant sur `community.docker >= 3.10.2` (`tasks/main.yml:2-19`) |
| Rocky 10 | Installation forcée du paquet `openssl` (`tasks/main.yml:21-27`) |
| **ansible-core** | `meta` annonce 2.9, mais le code utilise FQCN, `block/rescue`, `community.crypto` → **≥ 2.14 recommandé** |
| Python (cibles) | `python3-docker` **obligatoire** si `INSTALL_DOCKER: false` (échec bloquant) |
| Binaires (cibles) | `openssl`, `jq` (si `FW: true`) |
| Accès | `become: yes` requis |
| RAM | Non vérifié, **sauf NDB Cluster : 8 Go minimum par nœud** (`mysqlcluster.yml:16-21`) |

### 5.2 Collections Ansible

```yaml
collections:
  - name: community.docker
    version: ">=3.10.2"
  - name: community.crypto
  - name: ansible.posix      # ⚠ MANQUANTE dans requirements.yml fourni
```

`ansible.posix` est utilisée par `tasks/mysqlcluster.yml:26` mais absente du `requirements.yml`
d'origine. À ajouter dans le kit.

### 5.3 DNS à créer

| Enregistrement | Variable | Obligatoire |
|---|---|---|
| Portail utilisateur | `PORTAL_URL` (ex. `url.domain.tld`) | ✅ |
| Signal | `{{INSTANCE_NAME}}-signal.{{DOMAIN_NAME}}` | ✅ |
| Relais WS | `INIT_RELAYWS.<clé>.url` (ex. `relayws.domain.tld`) | ✅ |
| Registry local | — | ✅ (déconnecté) |
| Portail admin | `PORTALADMIN_URL` (ex. `admin.domain.local`) | ❌ (D14) |

### 5.4 Certificats SSL publics

Distincts de la PKI interne. **Deux certificats sont nécessaires** — c'est un point souvent
manqué :

```yaml
# infra_manager — couvre le portail ET le sous-domaine signal
TRAEFIK_SSL_CERTS:
  - cert_file: "files/<client>/fullchain.pem"
    key_file:  "files/<client>/privkey.pem"

# relayws_manager — couvre relayws.domain.tld
TRAEFIK_SSL_CERTS:
  - cert_file: "files/<client>/relayws-cert.pem"
    key_file:  "files/<client>/relayws-key.pem"
```

Un wildcard `*.domain.tld` couvre les deux cas.

> L'éditeur signale que pour un environnement de test avec certificats auto-signés, Chrome doit
> être lancé avec `--ignore-certificate-errors`. **À proscrire en production.**

### 5.5 Licence

`API_LICENSE` (`defaults:482`), déclarée dans `all.vars` selon le modèle éditeur.

⚠ Vérifier avec l'éditeur le mode de validation en environnement déconnecté
(`LICENSING_URL: https://licensing.reemo.io` — **injoignable chez ce client**).

> **Point bloquant potentiel non résolu** — voir §10, décision D6.

---

## 6. Le sujet critique : installation déconnectée

### 6.1 Les trois modes officiels

L'éditeur documente trois modes, **à choisir avant l'installation** :

| Mode | Principe | Applicable ici |
|---|---|---|
| **Online** | Les nœuds joignent `registry.reemo.io` directement | ❌ Site déconnecté |
| **Proxy** | Les nœuds passent par un proxy HTTP/HTTPS vers `registry.reemo.io` + `registry-auth.reemo.io` | ❌ Aucun accès sortant |
| **Offline** | Images pré-packagées, deux scénarios : **A — tarballs distribués**, **B — registry privé intermédiaire** | ✅ **Scénario B retenu** |

### 6.2 Convention de nommage des images

`REGISTRY_ENV` est un **préfixe de nom d'image**, pas un segment de chemin. Il n'y a **pas de `/`**
entre `REGISTRY_ENV` et le nom du composant.

> Documentation éditeur : *« REGISTRY_ENV: the prefix allowing the Ansible role to build image
> names in the format `REGISTRY_URL/REGISTRY_ENVapi` »*

**Le préfixe diffère selon la nature de l'image** — point d'attention majeur :

| Nature | `REGISTRY_ENV` | Exemple |
|---|---|---|
| **Infrastructure** (api, portal, mariadb, traefik…) | `reemoinfra` | `registry.reemo.io/reemoinfraapi:3.23.1-…` |
| **Images de session** (chromium, rdp, ssh…) | `reemosb` | `registry.reemo.io/reemosbchromium:latest` |

Dans le rôle `reemo-infra`, le préfixe `reemosb` n'est pas piloté par `REGISTRY_ENV` : il est
**codé en dur** dans `PROAPI_URL_REGISTRY_SB` (`defaults:657`) et `PROVISION_IMAGE_WARMUP`
(`defaults:1378`). Ces deux variables doivent donc être surchargées explicitement.

Correspondance amont → aval pour le miroir :

| Source (registry.reemo.io) | Cible (registry client) |
|---|---|
| `registry.reemo.io/reemoinfraapi:3.23.1-…` | `registry.client.lan/reemoinfraapi:3.23.1-…` |
| `registry.reemo.io/reemoinframariadb:11.4.11-…` | `registry.client.lan/reemoinframariadb:11.4.11-…` |
| `registry.reemo.io/reemosbchromium:latest` | `registry.client.lan/reemosbchromium:latest` |

> En conservant les mêmes noms d'images et en ne changeant que `REGISTRY_URL`, on garde
> `REGISTRY_ENV: "reemoinfra"` — configuration la plus proche du mode nominal, donc la plus
> supportable par l'éditeur.

### 6.3 Pourquoi le scénario A (tarballs) est écarté

L'éditeur propose le scénario A, mais il présente des lacunes rédhibitoires ici :

| # | Lacune | Détail |
|---|---|---|
| 1 | **MariaDB absent du script** | `tarball_generate.yml` et `load_image.yml` ne connaissent que `mysql`, jamais `mariadb`. Or en installation neuve c'est **MariaDB 11.4** qui est déployé → `checkimage` échoue. **Non documenté par l'éditeur** |
| 2 | **Images de session hors périmètre Ansible** | `load_image.yml` ne charge jamais `reemosbchromium`. L'éditeur documente une procédure **manuelle** (`docker save` / `docker load` sur chaque nœud) — automatisation absente |
| 3 | **Warmup désactivé par conception** | `LOAD_IMAGE: true` coupe le préchargement (`main.yml:762,781`). Confirmé par l'éditeur : *« Preloading automatically disables if LOAD_IMAGE: true »* |
| 4 | **Composants non couverts** | `vault`, `credentialapi`, `credentialportal` absents du script de tarball |
| 5 | **Pas de chemin de mise à jour** | Cycle pull → save → transport → load sur chaque nœud à chaque version |
| 6 | **Fragile à l'ajout de nœud** | Chaque nouveau worker de provision impose un `docker load` manuel complet |

**Scénario d'échec typique du mode tarball :**

```
✅ ansible-playbook   → se termine sans erreur
✅ Portail accessible → l'utilisateur se connecte
✅ Catalogue affiché  → "Chromium" apparaît
❌ Clic sur Chromium  → proapi → provision → docker pull …/reemosbchromium
                        → échec, session jamais créée
```

Le déploiement **paraît réussi** mais le produit est inutilisable. Ne se détecte qu'au test
fonctionnel de bout en bout.

### 6.4 Scénario B retenu : registry privé — recommandé par l'éditeur

> Documentation éditeur (INFRA) : *« You can also **upload the images to your own registry**. You
> can then run the installation in **Online mode** against **your registry**. »*
>
> Documentation éditeur (PROVISION), *Scenario B: Intermediate Private Registry* : *« The bridge
> machine pulls from registry.reemo.io then pushes to your private registry. […] You can then run
> the installation in Online mode against your private registry. »*

C'est donc une configuration **officiellement supportée**, et non un contournement.

**Principe** : une machine passerelle disposant d'un accès Internet récupère les images et les
pousse vers le registry client. Les nœuds sont ensuite installés **en mode Online contre le
registry local**.

```
┌─ MACHINE PASSERELLE (accès Internet) ─────────────────┐
│  docker login registry.reemo.io                       │
│  docker pull  registry.reemo.io/reemoinfra*           │
│  docker pull  registry.reemo.io/reemosbchromium       │
│  docker tag   → registry.client.lan/…                 │
│  docker push  → registry.client.lan                   │
│     (ou docker save → transport physique → load)      │
└───────────────────────────┬───────────────────────────┘
                            ▼
┌─ SITE CLIENT (déconnecté) ────────────────────────────┐
│  registry.client.lan:443   ← registry:2 + TLS + volume│
│         ▲          ▲          ▲                        │
│   infra_manager1  provision1  relayws_manager1        │
│   (pull deploy)   (pull       (pull                   │
│                    RUNTIME)    RUNTIME)               │
└────────────────────────────────────────────────────────┘
```

### 6.5 Deux pièges non documentés par l'éditeur

#### ⚠ Piège 1 — `REGISTRY_TLS_VERIFY: false` ne fait pas ce que la doc laisse entendre

La documentation décrit cette variable comme *« allows use of a registry on port 80 without
TLS »*. C'est **incomplet et trompeur**.

Vérification dans le code : la variable n'agit **que sur les contrôles côté Ansible** — elle
remplace `docker manifest inspect` par `skopeo inspect --tls-verify=false`
(`tasks/checkimage.yml:25-33`).

**Elle n'a aucun effet sur le daemon Docker**, qui effectue le vrai `pull` au déploiement Swarm
et à la création des sessions. Avec un registry en HTTP ou TLS auto-signé : Ansible passe les
vérifications, puis les services échouent sur `x509: certificate signed by unknown authority`.

Pour un registry en clair, il faudrait déclarer `insecure-registries` dans le daemon Docker —
ce qui mène au piège 2.

#### ⚠ Piège 2 — le rôle écrase `/etc/docker/daemon.json`

```yaml
# tasks/redhat_docker.yml:69-73 (idem ubuntu_docker.yml:56-60)
- name: copy daemon.json
  copy:
    src: "files/provision/daemon.json"
    dest: "/etc/docker/daemon.json"
```

Contenu du fichier : `{ "seccomp-profile": "/etc/docker/seccomp-chrome.json" }`

Le fichier est **remplacé intégralement** sur les nœuds de provision à chaque exécution avec
`INSTALL_DOCKER: true`. Toute déclaration `insecure-registries` ajoutée manuellement serait
silencieusement effacée → les sessions cesseraient de démarrer, sans lien apparent avec
l'exécution Ansible.

#### → Conclusion : vrai certificat TLS sur le registry

```yaml
REGISTRY_URL: "registry.client.lan"     # port 443
REGISTRY_ENV: "reemoinfra"
REGISTRY_TLS_VERIFY: "true"             # mode nominal
```

Le certificat doit être signé par une CA **présente dans le trust store de l'OS** des trois nœuds
(`/etc/pki/ca-trust/source/anchors/` sur Rocky, `/usr/local/share/ca-certificates/` sur Ubuntu).
Aucun `daemon.json` à modifier, aucun `skopeo`, aucun conflit avec le rôle.

### 6.6 Amorçage (poule et œuf)

Le registry est lui-même un conteneur :

1. Récupérer `registry:2` depuis la zone connectée → `docker save` → transport
2. `docker load` sur la machine hôte du registry
3. Démarrer le registry (certificat TLS + volume persistant)
4. **Puis** alimenter avec les ~31 images d'infrastructure + les images de session

C'est le seul transport physique nécessaire, réalisé une fois.

### 6.7 Variables à surcharger

```yaml
# group_vars/all.yml
REGISTRY_URL: "registry.client.lan"
REGISTRY_ENV: "reemoinfra"
REGISTRY_USERNAME: "<user>"
REGISTRY_PASSWORD: "<pass>"
REGISTRY_TLS_VERIFY: "true"
LOAD_IMAGE: "false"          # mode Online contre le registry local
TARBALL_GENERATE: ""

# group_vars/infra_manager.yml
PROAPI_URL_REGISTRY_SB: "registry.client.lan/reemosbchromium:latest"
PROAPI_REGISTRY_ACTIVE: "active"

# group_vars/provision.yml
PROVISION_IMAGE_WARMUP:
  - "registry.client.lan/reemosbchromium:latest"   # version épinglée recommandée

# group_vars/relayws_manager.yml
RELAYWS_IMAGE_WARMUP:
  - "registry.client.lan/reemoinfraws-relay-server:1.2.0-6aa63f223fe9"
```

> L'éditeur recommande explicitement d'**épingler les versions** dans le warmup pour gérer les
> rollbacks : *« Internal mirror + pinned version »*.

### 6.8 Le catalogue en base pointe encore sur registry.reemo.io

`files/container_json/chromium.json` contient en dur :

```json
{"name":"Chromium", "reference":"registry.reemo.io/reemosbchromium", ...}
```

Ce fichier est injecté comme secret Docker au moment de `INIT_DB` (`tasks/db.yml:16-23`) et
devient l'entrée « Chromium » du catalogue. **Il faut corriger cette référence**, sinon proapi
tentera un pull vers Internet à chaque session.

Deux options :

- fournir un `INIT_IMAGES` personnalisé au moment de `INIT_DB` (`defaults:353`, encodé base64 en
  `db.yml:371`) — **à privilégier**, car reproductible ;
- ou corriger la référence **après installation**, depuis l'interface d'administration
  (*Containers & Workstations → Container Images*).

### 6.9 Comparatif des scénarios

| Point | Scénario A (tarballs) | **Scénario B (registry local)** |
|---|---|---|
| Statut éditeur | Documenté | **Documenté et recommandé** |
| MariaDB | ❌ Contournement manuel | ✅ Résolu |
| Images de session | ⚠ Procédure manuelle par nœud | ✅ Résolu |
| `vault`, `credentialapi`… | ❌ Non couverts | ✅ Résolu |
| Ajout d'un worker provision | `docker load` manuel | ✅ Rien à faire |
| Mise à jour de version | Cycle complet sur chaque nœud | ✅ Push sur le registry + `--tags` |
| `LOAD_IMAGE` | `true` | `false` (mode Online) |
| Warmup | ❌ Désactivé | ✅ Actif |
| Mode d'installation | Offline | **Online contre registry local** |

### 6.10 Dimensionnement du registry

~31 images d'infrastructure + images de session (Chromium 2-4 Go, davantage pour RDP/Ubuntu
Desktop). Prévoir **100 Go** pour tenir plusieurs versions en parallèle (rollbacks).

> **Le registry est un composant de PRODUCTION, pas de build.** Provision et relayws en dépendent
> au runtime, en permanence. Sa disponibilité conditionne la création de toute nouvelle session.

---

## 7. Alertes sécurité

### 7.1 Certificats de démonstration — CRITIQUE

`files/ssl/` contient **25 certificats ET leurs 25 clés privées en clair**.

| # | Problème | Impact |
|---|---|---|
| 1 | **`workstation_ca.key` est présent** — clé privée d'une **autorité de certification**, valide jusqu'en **2035** | Quiconque possède l'archive peut forger des certificats acceptés par cette CA |
| 2 | Toutes les clés feuilles livrées (`reemo_api.key`, `reemo_traefik.key`, `reemo_mariadb.key`…) | mTLS interne totalement contournable |
| 3 | Mécanisme de repli automatique vers ces certs si `INITCA_ENABLE: false` (`secrets_template.yml:102-110`) | Utilisation silencieuse en production |
| 4 | `reemo_saml.crt` = auto-signé `CN=localhost` | Hors chaîne de confiance |
| 5 | `reemo_db.crt` porte `CN=reemo_mariadb` | Incohérence de CN |
| 6 | Certs feuilles expirant **oct. 2027 – août 2028**, sans rotation auto | Panne à terme |

> **`INITCA_ENABLE: "true"` + `LOCAL_PATH` est NON NÉGOCIABLE en production.**

Avec `INITCA_ENABLE: true`, la PKI est générée sur le contrôleur Ansible : CA ECC secp521r1,
20 certificats de service, validité 824 j, renouvellement automatique à J-30
(`tasks/initca.yml`, `initca_createcert.yml`).

**Le répertoire `LOCAL_PATH` contient `ca.key` et doit être sauvegardé et protégé.**

### 7.2 Secrets par défaut triviaux — à surcharger

| Variable | Défaut | Réf. |
|---|---|---|
| `STUN_USERNAME` | `reemo` | `defaults:1271` |
| `STUN_PASSWORD` | `reemo` | `defaults:1272` |
| `API_INTERNAL_HMAC` | `reemo` | `defaults:630` |
| `CREDENTIALAPI_VAULT_unsafe` | `true` (TLS Vault désactivé) | `defaults:1100` |

### 7.3 Secrets auto-générés (aucune action requise)

Générés par `openssl rand -hex 32` si laissés vides : HMAC SIGNAL, PROAPI, PRORELAYAPI,
PROCLOUDAPI, WOLAPI, LOGAPI, APPLIANCEAPI, APPLIANCEPROVISION, CREDENTIALAPI, COOKIE_SECRET,
clé ECC, `MYSQL_ROOT_PASSWORD`, `DB_USER_PASSWORD`, `TURN_PASSWORD`
(`tasks/secrets.yml`, `secrets_all.yml`, `secrets_turn.yml`).

---

## 8. Bugs et pièges du rôle

Le rôle est un **livrable éditeur : il ne sera pas modifié**. Les contournements vont dans le kit
et le runbook.

### 8.1 Bugs bloquants

| # | Bug | Contournement |
|---|---|---|
| 1 | **`NGINX_FW_INTERFACE` non définie** — utilisée dans `nginx_fw.yml:6`, définie nulle part → échec de templating | La définir dans `group_vars/provision.yml` |
| 2 | **`ansible.posix` absente** de `requirements.yml` (requise par `mysqlcluster.yml:26`) | Ajoutée dans le kit |
| 3 | **MariaDB absent du mode offline** | Registry local (scénario B) |
| 4 | **`reemosbchromium` hors périmètre de `load_image.yml`** — procédure manuelle chez l'éditeur | `PROAPI_URL_REGISTRY_SB` + `PROVISION_IMAGE_WARMUP` → registry local |
| 5 | **`DOCKER_VERSION_MYSQLBACKUP` non définie** — référencée par `mysqlbackup.yml:60`, `defaults` ne définit que `DOCKER_VERSION_BACKUP` | Sans objet (backup non retenu) |

### 8.2 Pièges d'exploitation

| # | Piège | Conséquence |
|---|---|---|
| 6 | **`ansible_play_hosts_all \| first`** dans `swarm_init_api/portal/relayws.yml:7,21` | Si plusieurs groupes sont dans **le même play**, tous rejoignent le même Swarm. **Résolu par la structure officielle : un *play* distinct par groupe** dans le playbook (§9.3). `--limit` n'est donc pas nécessaire, mais reste utile pour rejouer un seul environnement |
| 7 | **Impossible d'ajouter un nœud à un Swarm existant** (infra/api/portal/relayws) — le join-token n'est généré qu'au `swarm init` initial | Procédure manuelle `docker swarm join-token` à documenter |
| 8 | **`INIT_DB` non idempotente** — relancer à `true` fait échouer le playbook (logs hors fenêtre `DB_DOCKER_LOGS_SINCE: 5m`) **et** écrase la config admin | Passage par `--extra-vars "INIT_DB=true"` au **premier run uniquement**, conformément à la doc éditeur |
| 9 | **`swarm_firewall_redhat.yml` ne s'exécute jamais si `ansible_play_hosts \| length == 1`** (`main.yml:170`) | En all-in-one mono-nœud, ouvrir les ports Swarm manuellement si besoin |
| 10 | **`relayws_manager` exclu du firewall Swarm automatique** (`main.yml:171`) | Ouverture manuelle |
| 11 | **`swarm_ports` figé sur 4789** alors que `SWARM_DATA_PATH_PORT` est configurable (`defaults:1425-1429`) | Surcharger `swarm_ports` si port modifié |
| 12 | **Le groupe parent `provision` est obligatoire** (`main.yml:202`) — sans lui le Swarm de provision n'est jamais initialisé | Structure `provision: children: provision1_manager:` (§9.2) |
| 13 | **Tâches sans tag ignorées avec `--tags`** (contrôles HMACSECRET, `set_fact provision_prefix`) | Peut produire des variables indéfinies |
| 14 | **`/opt/acme/acme.json`** référencé comme storage ACME (`traefik.yml:162`) mais monté par aucun bind mount | Stockage ACME non persistant (sans objet : certs fournis) |
| 15 | **Flux retour PROVISION → INFRA (8443, 8446) et PROVISION → RELAY (443)** absents de l'analyse du rôle | Déploiement OK mais sessions non fonctionnelles — voir §4.3 |

### 8.3 Bugs sans impact ici

| # | Bug |
|---|---|
| 15 | `initcaworkstation.yml:73` signe le mauvais CSR → `workstation_portal.crt` porte le CN du CA. Fichier non idempotent |
| 16 | `workstation.yml:163-217` inspecte `reemo_portal` au lieu du service workstation, tagué `portal` |
| 17 | `apicronlog.yml:182-187` : `APICRONLOG_NETWORKS` initialisée en liste puis écrasée par une chaîne |
| 18 | `logapi.yml:198-199` : condition tautologique `DB_DIALECT == "mysql" or DB_DIALECT == "mysql"` |
| 19 | `portaladmin.yml:184,197,210` utilise `ipwhitelist` (déprécié en Traefik v3) |
| 20 | `tasks/db copy.yml` — fichier mort de 17 Ko, à ne pas confondre avec `db.yml` |

---

## 9. Plan de déploiement

Structure alignée sur le modèle éditeur *Bastion+ with URL*, adaptée au mode déconnecté.

### 9.1 Arborescence du kit à produire

```
/home/a949196/Code/Reemo/reemo-deploy-<client>/
├── ansible.cfg
├── requirements.yml              # + ansible.posix
├── inventory.yml                 # format YAML (modèle éditeur)
├── playbooks/
│   ├── reemo-infra.yml           # 3 plays : infra → relayws → provision
│   ├── preflight.yml             # checks lecture seule
│   └── secrets/                  # init.json si Vault activé (à chiffrer)
├── group_vars/
│   ├── all.yml
│   ├── infra_manager.yml
│   ├── provision.yml
│   └── relayws_manager.yml
├── host_vars/
├── files/<client>/
│   ├── fullchain.pem             # cert SSL public (infra)
│   ├── privkey.pem
│   ├── relayws-cert.pem          # cert SSL public (relayws)
│   └── relayws-key.pem
├── pki/                          # LOCAL_PATH — contient ca.key, À SAUVEGARDER
├── scripts/
│   ├── mirror-images.sh          # alimente le registry local
│   └── backup-pki.sh
├── PREREQUIS-CLIENT.md
└── RUNBOOK.md
```

### 9.2 Inventaire (format YAML éditeur)

```yaml
all:
  vars:
    API_LICENSE: "< LICENSING_KEY >"
    REGISTRY_URL: "registry.client.lan"
    REGISTRY_ENV: "reemoinfra"
    REGISTRY_USERNAME: "< username >"
    REGISTRY_PASSWORD: "< password >"
    REGISTRY_TLS_VERIFY: "true"
    INITCA_ENABLE: "true"
    LOCAL_PATH: "{{ playbook_dir }}/../pki"

infra_manager:
  vars:
    HMACSECRET: ""                    # OBLIGATOIREMENT vide en all-in-one
    TRAEFIK_SSL_CERTS:
      - cert_file: "files/<client>/fullchain.pem"
        key_file:  "files/<client>/privkey.pem"
    PORTAL_URL: "url.domain.tld"
    PROVISION_SIGNAL_IP:
      - ip: "192.168.10.66"
    INIT_PROVISION:
      provision1:
        type: "SWARM"
        ip:
          - "192.168.10.47"
        relayws: "relayws1"           # association provision → relais
    INIT_RELAYWS:
      relayws1:
        type: "WS_SWARM"
        url: "relayws.domain.tld"
        ip:
          - "192.168.10.59"
    TURN_ENABLED: true
    TURN1_NODE: "infra_manager1"
    TURN1_IP: "< IP joignable par les clients >"
    DB_DIALECT: "mysql"
    PROAPI_URL_REGISTRY_SB: "registry.client.lan/reemosbchromium:latest"
  hosts:
    infra_manager1:
      ansible_host: "192.168.10.66"

provision:
  children:
    provision1_manager:
      vars:
        NGINX_FW_INTERFACE: "eth0"    # ⚠ sinon échec de templating (bug n°1)
        PROVISION_IMAGE_WARMUP:
          - "registry.client.lan/reemosbchromium:latest"
      hosts:
        provision1_manager1:
          ansible_host: "192.168.10.47"

relayws_manager:
  vars:
    TRAEFIK_SSL_CERTS:
      - cert_file: "files/<client>/relayws-cert.pem"
        key_file:  "files/<client>/relayws-key.pem"
    RELAYWS_IMAGE_WARMUP:
      - "registry.client.lan/reemoinfraws-relay-server:1.2.0-6aa63f223fe9"
  hosts:
    relayws_manager1:
      ansible_host: "192.168.10.59"
```

> **Points de structure importants :**
> - Le groupe parent `provision` avec `children: provision1_manager` est **obligatoire** (bug n°12)
> - `relayws` a **son propre certificat SSL public** — deux certificats distincts à prévoir
> - `INIT_PROVISION.provision1.relayws: "relayws1"` **associe** la ferme de provision au relais

### 9.3 Playbook (structure officielle)

**`playbooks/reemo-infra.yml`** — un *play* par groupe, ce qui **résout le piège n°6** :

```yaml
- name: Installation of Reemo Infra Server
  hosts: infra_manager
  gather_facts: "True"
  roles:
    - role: reemo-infra
      become: yes

- name: Install Relayws
  hosts: relayws_manager
  gather_facts: "True"
  roles:
    - role: reemo-infra
      become: yes

- name: Installation of Provision
  hosts: provision
  gather_facts: "True"
  roles:
    - role: reemo-infra
      become: yes
```

> Le rôle doit être accessible sous le nom **`reemo-infra`** (sans le suffixe de version).
> Prévoir un lien symbolique ou un `roles_path` adapté dans `ansible.cfg`.

### 9.4 Phases d'exécution

```bash
# PHASE 0 — Amorçage du registry (une seule fois, manuel)
#   1. docker save registry:2 depuis la zone connectée
#   2. transport physique
#   3. docker load + démarrage du registry avec TLS
#   4. ./scripts/mirror-images.sh   → pousse les ~31 images infra + images de session

# PHASE 1 — Pré-vol (lecture seule)
ansible-playbook -i inventory.yml playbooks/preflight.yml

# PHASE 2 — Déploiement initial (commande officielle, UNE SEULE FOIS)
ansible-playbook -i inventory.yml playbooks/reemo-infra.yml \
    --extra-vars "INIT_DB=true" \
    --extra-vars "INSTALL_DOCKER=true"

# PHASE 3 — Validation fonctionnelle
#   - accès portail
#   - création du premier compte administrateur
#   - création d'une session Chromium ← LE test qui valide le mode déconnecté
#   - test WebRTC / TURN

# PHASE 4 — Exploitation courante (sans INIT_DB ni INSTALL_DOCKER)
ansible-playbook -i inventory.yml playbooks/reemo-infra.yml
```

> **`INIT_DB=true` et `INSTALL_DOCKER=true` ne sont passés qu'au premier run**, en `--extra-vars`.
> Les rejouer casserait le playbook (bug n°8).

> L'ordre officiel est **infra → relayws → provision**. Il fonctionne car `INIT_PROVISION` et
> `INIT_RELAYWS` n'écrivent que des enregistrements en base et des backends HAProxy configurés
> par IP : les cibles n'ont pas besoin d'être actives à ce moment-là.

### 9.5 Sécurisation du fichier `init.json` (si Vault activé)

Non retenu ici, mais à documenter si la décision D13 revient sur ce point. Procédure officielle :

```bash
# Immédiatement après le premier run
ansible-vault encrypt playbooks/secrets/reemo/vault/init.json

# Les exécutions suivantes nécessitent alors :
ansible-playbook -i inventory.yml playbooks/reemo-infra.yml --ask-vault-pass
```

> L'éditeur insiste : sauvegarder **le contenu de `init.json` ET la passphrase ansible-vault**,
> dans un gestionnaire de mots de passe ou un coffre. Ce sont deux secrets distincts.

### 9.6 Opérations jour-2

| Opération | Commande |
|---|---|
| Mise à jour applicative | `--tags portal,api` (+ `FORCE_UPDATE: true` si besoin) |
| Renouvellement cert public | `--tags traefik_ssl` |
| Rotation PKI interne | Automatique à J-30, ou `--tags initca` |
| Préchargement d'images | `--tags provision_image_warmup` |
| Ajout d'une zone provision | `INIT_PROVISION` + `--limit provisionN` + `--tags proapi` |
| Sauvegarde PKI | `./scripts/backup-pki.sh` — **critique** |

69 tags disponibles — voir annexe §11.2.

### 9.7 Contenu du `preflight.yml`

Playbook **lecture seule** validant avant tout déploiement :

- OS ∈ {Ubuntu, RedHat, Rocky} et version majeure
- Sizing conforme au §4.1 (CPU, RAM, disque)
- `python3-docker`, `openssl` présents
- Résolution DNS : `PORTAL_URL`, sous-domaine signal, `relayws.domain.tld`, registry
- **Registry joignable en TLS depuis CHAQUE nœud** (infra, provision, relayws)
- **CA du registry présente dans le trust store** des trois nœuds
- **Présence effective des images** attendues : ~31 images `reemoinfra*` **+ `reemosbchromium`**
- Validité et couverture SAN des **deux** certificats SSL publics
- **Matrice de flux complète**, y compris les flux retour PROVISION → INFRA (8443, 8446) et
  PROVISION → RELAY (443)
- Espace disque `/opt`
- Cohérence `HMACSECRET` vide ↔ groupe `infra_manager`

---

## 10. Décisions à trancher

### Décisions bloquantes (avant production du kit)

| # | Décision | Options | Recommandation |
|---|---|---|---|
| **D1** | **Hébergement du registry** | VM dédiée / colocalisé sur `infra_manager1` | **VM dédiée** — provision et relayws en dépendent au runtime en permanence ; une dépendance circulaire au redémarrage est à proscrire. 2 vCPU / 4 Go / 100 Go |
| **D2** | **Certificat du registry** | PKI interne client / CA à distribuer dans le trust store | **PKI interne** si elle existe. Sinon prévoir la distribution de la CA sur les trois nœuds. **Ne pas partir sur du HTTP ou de l'auto-signé** (pièges §6.5) |
| **D3** | **Authentification du registry** | htpasswd / anonyme en lecture | À définir selon la politique client. `REGISTRY_USERNAME`/`PASSWORD` sont propagés à proapi pour le pull runtime |
| **D4** | **Scénario offline** | A (tarballs) / **B (registry privé)** | **Scénario B** — officiellement recommandé par l'éditeur, et seul viable en exploitation (§6.3) |
| **D5** | **Domaine et `INSTANCE_NAME`** | — | Garder `INSTANCE_NAME: "reemo"` (évite des écueils de nommage de certificats) |
| **D6** | **Validation de licence hors-ligne** | — | ⚠ **À VALIDER AVEC L'ÉDITEUR.** `LICENSING_URL: https://licensing.reemo.io` sera injoignable. Consulter la [page Licensing](https://doc-private.reemo.io/licensing.html) et confirmer l'existence d'un mode offline |

### Décisions de configuration

| # | Décision | Impact |
|---|---|---|
| **D7** | **IP publique du TURN** (`TURN1_IP`) | Les clients doivent joindre cette IP en 58200 tcp+udp |
| **D8** | **Certificats SSL publics** : **deux** sont nécessaires (infra + relayws) | Celui d'infra doit couvrir le portail **et** `reemo-signal.<domaine>` ; celui du relais couvre `relayws.domain.tld` |
| **D9** | **SMTP** : vraiment non requis ? | Sans lui : pas de vérification d'email, pas de 2FA email, pas d'invitations (`db.yml:3-11`) |
| **D10** | **Sauvegarde base de données** | `MYSQL_BACKUP_ENABLED` non retenu. Confirmer que le client a une stratégie alternative (snapshot VM ?) |
| **D11** | **Correction du catalogue Chromium** | Via `INIT_IMAGES` au moment de `INIT_DB` (reproductible, à privilégier), ou manuellement en post-install |
| **D12** | **Stockage et protection de `pki/`** | Contient `ca.key`. Coffre-fort ? Dépôt Git privé via `INITCA_GIT_ENABLE` ? |
| **D13** | **Vault / Credential** — écarté, alors que le modèle éditeur les active | Si retenu : ajouter `VAULT_ENABLED`, `CREDENTIAL_ENABLED`, `CREDENTIALPORTAL_ENABLED`, `VAULT_INIT_FILE`, et la procédure `ansible-vault` sur `init.json` (§9.5). **Ouvre le flux PROVISION → INFRA:8446** |
| **D14** | **Portail admin séparé** — écarté, alors que le modèle éditeur le prévoit | Si retenu : `PORTALADMIN_URL`, `PORTALADMIN_URL_RESTRICT_IP`, `PORTALADMIN_TYPE: "instadmin"`, `PORTAL_TYPE: "orguser"` + un enregistrement DNS et une entrée SAN supplémentaires |
| **D15** | **Sizing** | Valider le nombre de sessions simultanées visé. Le sizing officiel (§4.1) vaut pour **10 sessions** ; `provision1_manager1` est le nœud à ajuster en priorité |

### Points à confirmer avec l'éditeur Reemo

1. **Licence en environnement déconnecté** (D6) — bloquant potentiel
2. **Absence de `mariadb` dans `tarball_generate.yml`** — bug ou choix ? Le scénario A est-il
   réellement supporté pour une installation neuve en 2.4.1 ?
3. **`REGISTRY_TLS_VERIFY` documenté comme permettant « a registry on port 80 without TLS »**
   alors qu'il n'agit pas sur le daemon Docker — clarification demandée
4. **`daemon.json` écrasé sur les nœuds de provision** — comment déclarer un `insecure-registries`
   de façon pérenne ?
5. `DOCKER_VERSION_MYSQLBACKUP` non définie — bug confirmé ?
6. Bug de CN dans `initcaworkstation.yml:73`
7. **Présence de `workstation_ca.key` dans le paquet livré** — signalement de sécurité

---

## 11. Annexes

### 11.1 Feature flags (défauts)

| Variable | Défaut | Retenu ici |
|---|---|---|
| `LOGAPI_ENABLED` | `true` | ✅ |
| `CRON_ENABLE` | `"true"` | ✅ |
| `TRAEFIK_ENABLE` | `"true"` | ✅ |
| `MYSQL_ENABLE` | `"true"` | ✅ |
| `FIREWALL_REDHAT` | `true` | ✅ |
| `TURN_ENABLED` | `false` | ✅ activé |
| `INITCA_ENABLE` | `"false"` | ✅ activé |
| `INSTALL_DOCKER` | `"false"` | ✅ activé |
| `SWARM_INIT` | `{{INSTALL_DOCKER}}` | ✅ activé |
| `CREDENTIAL_ENABLED` | `false` | ❌ |
| `VAULT_ENABLED` | `false` | ❌ |
| `WORKSTATION_ENABLED` | `"false"` | ❌ |
| `APPLIANCE_ENABLED` | `false` | ❌ |
| `PROCLOUDAPI_ENABLE` | `"false"` | ❌ |
| `WOLAPI_ENABLE` | `"false"` | ❌ |
| `MAINTENANCE_ENABLE` | `"false"` | ❌ |
| `PORTALADMIN_ENABLED` | `false` | ❌ |
| `MYSQL_BACKUP_ENABLED` | `false` | ❌ (D10) |
| `API_MAIL_ACTIVE` | `"false"` | ❌ (D9) |
| `DATADOG_ENABLED` | `false` | ❌ |
| `FW` | `"false"` | ❌ (nécessiterait `jq`) |
| `HEALTHCHECK_ENABLE` | `"false"` | à décider |
| `TRAEFIK_PROMETHEUS_ENABLE` | `"false"` | à décider |
| `FORCE_UPDATE` | `false` | ponctuel |
| `INIT_DB` | `"false"` | ⚠ une seule fois |

### 11.2 Tags Ansible (69)

**Infrastructure**
`install_docker` · `swarm_init` · `swarm_init_provision` · `swarm_init_relayws` · `docker_login` ·
`skopeo` · `iptables` · `forwarding` · `network` · `nginx` · `nginx_ssl` · `provision_firewall` ·
`datadog`

**PKI et secrets**
`initca` · `initca_git` · `initcarelayws` · `initcaworkstation` · `secrets` · `secrets_all` ·
`secrets_ca` · `secrets_db` · `secrets_dbca` · `secrets_turn`

**Bases de données**
`db` · `mysql` · `mariadb` · `mysqlbackup` · `mysqlcluster` · `mysqlclustermgmd` ·
`mysqlclustermysqld` · `mysqlclusterndbd` · `mysqlclusterstatus` · `mysqlclusterbackup` · `logapidb`

**Services applicatifs**
`api` · `apicron` · `apicroncloud` · `apicronldap` · `apicronlog` · `apicronsecuritylogs` ·
`apicronworkstation` · `portal` · `portaladmin` · `signal` · `proapi` · `prorelayapi` ·
`procloudapi` · `wolapi` · `logapi` · `logapicron` · `workstation` · `turn` · `exim4` ·
`maintenance` · `applianceapi` · `applianceportal` · `credentialapi` · `credentialportal` ·
`vault` · `vault_init`

**Traefik**
`traefik` · `traefik_ssl` · `traefik_docker` · `traefik_copy_config` · `traefik_default_ssl`

**Images / offline**
`tarball_generate` · `load_image` · `provision_image_warmup` · `relayws_image_warmup` · `relayws_cron`

### 11.3 Options de base de données

```
DB_DIALECT ?
├─ "NDBCLUSTER" ──► NDB Cluster (3 mgmd + 3 ndbd + 3 mysqld)
│                    prérequis : ≥3 nœuds Swarm, ≥8 Go RAM/nœud,
│                    MYSQL_NODE_HOSTNAME_DB1/2/3 renseignés
│                    → clients : DB_HOST=reemo_mysql-mysqld, dialect=mysql
└─ "mysql"
   ├─ DB_HOST ≠ API_MYSQL_HOST ──► Base externe (rien de déployé)
   └─ DB_HOST == API_MYSQL_HOST
      ├─ /opt/reemo/db existe   ──► MySQL 8.0 (legacy, sans TLS)
      └─ /opt/reemo/db absent   ──► MariaDB 11.4 (RETENU, mTLS forcé)
```

Bases créées : `reemo` (applicatif) et `reemo_log` (audit, via `MYSQL_EXTRA_DATABASES`).

### 11.4 Stockage persistant

| Chemin hôte | Service | Owner |
|---|---|---|
| `/opt/reemo/mariadb` | MariaDB | `8000:root`, `0750` |
| `/opt/reemo/vault` | Vault (non retenu) | `1000:1000`, `0700` |
| `/opt/traefik/config` | Traefik (certs + TOML) | `root:docker`, `0750` |
| `/opt/ssl/` | nginx provision/relayws | `root:root` |

**Logs** : aucun volume. Tous les services utilisent `logging.driver: syslog` vers
`unixgram:///dev/log`. La persistance est déléguée au rsyslog de l'hôte.

**Secrets** : Docker secrets Swarm montés dans `/run/secrets/<nom>` en `0400`.

### 11.5 Mécanisme de déploiement et rollback

`checkimage.yml` est un wrapper de pré-vol appelé ~25 fois. Il résout dynamiquement
`{{PROJET|upper}}_DOCKER_IMAGE`, vérifie l'existence de l'image (manifest / skopeo / local), puis
importe `{{PROJET}}.yml`.

Chaque `docker_swarm_service` applique :

```yaml
update_config:
  parallelism: 1
  order: stop-first
  monitor: 30s
  max_failure_ratio: 0.0
  failure_action: rollback     # ← rollback automatique
rollback_config:
  failure_action: pause
```

Puis **triple vérification post-déploiement** : attente d'état terminal, échec explicite si
l'état final n'est pas `completed`, et **contrôle de la version applicative** par `docker exec`
lisant `/opt/reemo/version.txt` — ce qui détecte les rollbacks silencieux.

Pour `db`/`logapidb` (jobs one-shot), la validation se fait par grep dans les logs
(`INIT DB OK`, `UPDATE DB OK`, `UPDATE LOGAPIDB OK`).

---

## Historique

| Date | Événement |
|---|---|
| 2026-08-20 | Analyse initiale du rôle `reemo-infra-2.4.1`, rédaction de ce document |
| 2026-08-20 | Croisement avec la documentation éditeur (*Bastion+ with URL*, *Registry and Images* INFRA et PROVISION). Corrections C1 à C8 (§0.2) : `REGISTRY_ENV`, structure du playbook, ordre de déploiement, matrice de flux, `INIT_PROVISION`/`INIT_RELAYWS`, scénarios offline. Ajout du sizing officiel et des décisions D13 à D15 |

---

## Prochaine étape

Trancher les décisions **D1 à D6** (§10) — en priorité **D6 (licence hors-ligne)**, qui
conditionne la faisabilité même du projet — puis production du kit de déploiement
`reemo-deploy-<client>/` décrit en §9.1.
