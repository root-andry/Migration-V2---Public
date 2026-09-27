# Migration de sites (serveur à serveur)

Application Windows pour migrer les fichiers de sites web d'un serveur **source** vers un serveur **destination**, directement de serveur à serveur, **sans accès root**.

Elle fonctionne avec tout serveur Linux accessible en SSH : Plesk, ISPConfig, cPanel, DirectAdmin, HestiaCP, CyberPanel, Virtualmin ou serveur sans panneau. L'export de la base de données (MySQL/MariaDB ou PostgreSQL) est proposé en option.

L'outil se présente sous la forme d'un fichier unique, **`Migration.exe`** : une interface graphique qui ne demande aucune installation (Python n'est pas nécessaire).

---

## Sommaire

- [Principe de fonctionnement](#principe-de-fonctionnement)
- [Prérequis](#prérequis)
- [Installation](#installation)
- [Démarrage rapide](#démarrage-rapide)
- [Référence de l'interface](#référence-de-linterface)
- [Base de données](#base-de-données)
- [Sécurité](#sécurité)
- [Dépannage](#dépannage)
- [Auteur](#auteur)

---

## Principe de fonctionnement

```
            SSH (orchestration uniquement)
   PC  ─────────────────────┬──────────────────────┐
                            ▼                      ▼
                    Serveur SOURCE  ◄──────►  Serveur DESTINATION
                                   rsync ou tar
                                   via SSH (direct)
```

1. Le PC se connecte aux deux serveurs en SSH. Il se contente d'orchestrer : **aucun fichier de site ne transite par le PC**.
2. Pour chaque site, une **paire de clés SSH temporaire** (Ed25519) est générée :
   - la clé privée est déposée chez l'utilisateur SSH d'un des serveurs, appelé le « lanceur » ;
   - la clé publique est ajoutée à `~/.ssh/authorized_keys` de l'autre serveur.
3. Le lanceur transfère les fichiers directement vers l'autre serveur, ou depuis celui-ci :
   - avec **rsync** s'il est disponible des deux côtés (progression affichée, reprise possible, seules les différences sont recopiées) ;
   - sinon avec **tar + gzip via ssh**, des outils présents même dans les shells chrootés.
4. À la fin, **même en cas d'erreur ou d'arrêt**, les clés temporaires sont supprimées des deux côtés.

Les utilisateurs SSH à renseigner sont les **propriétaires du site** de chaque côté. Les fichiers copiés appartiennent ainsi automatiquement au bon utilisateur.

| Panneau | Utilisateur SSH | Exemple de dossier |
|---|---|---|
| Plesk | Utilisateur système de l'abonnement (accès SSH activé) | `/var/www/vhosts/domaine.com/httpdocs` |
| ISPConfig | « Shell user » du site | `/var/www/clients/client1/web1/web` |
| cPanel | Utilisateur du compte (accès SSH / Terminal) | `/home/utilisateur/public_html` |
| DirectAdmin | Utilisateur DirectAdmin | `/home/utilisateur/domains/domaine.com/public_html` |
| HestiaCP | Utilisateur Hestia/Vesta (shell activé) | `/home/utilisateur/web/domaine.com/public_html` |
| CyberPanel | Utilisateur du site | `/home/domaine.com/public_html` |
| Virtualmin | Utilisateur du serveur virtuel | `/home/utilisateur/public_html` |
| Autre (Linux) | Propriétaire des fichiers (root déconseillé) | `/var/www/html` |

---

## Prérequis

**Sur le PC**

- Windows 10 ou 11 ;
- rien d'autre à installer.

**Sur les serveurs**

- un accès SSH (mot de passe ou clé) pour l'utilisateur propriétaire du site, de chaque côté ;
- `ssh` sur au moins un des deux serveurs (celui qui lance le transfert) ;
- `rsync` des deux côtés (recommandé), ou à défaut `tar` et `gzip` des deux côtés ;
- les deux serveurs doivent pouvoir se joindre directement en SSH ;
- pour l'export de base : `mysqldump`/`mariadb-dump` ou `pg_dump` sur la source.

---

## Installation

1. Téléchargez `Migration.exe` depuis la page [Releases](https://github.com/root-andry/Migration-V2---Public/releases/latest) (section *Assets*).
2. Placez-le dans un dossier de votre choix (par exemple `Documents\Migration`).
3. Double-cliquez dessus pour le lancer.

La configuration est enregistrée dans un fichier `config.json` créé **à côté de `Migration.exe`**. Au lancement, l'application le recharge automatiquement s'il existe ; sinon, elle démarre avec une configuration vide.

---

## Démarrage rapide

1. Lancez `Migration.exe`.
2. Onglet **Paramètres** : choisissez le type des serveurs source et destination, puis renseignez leur adresse et leur port SSH.
3. Onglet **Sites** : ajoutez un site par domaine (**+ Ajouter**). Pour chaque côté, indiquez l'utilisateur SSH, son mot de passe (ou une clé privée) et le dossier du site.
4. Cochez les sites à traiter (clic sur la case ou barre d'espace).
5. Enregistrez la configuration (**Fichier → Enregistrer** ou `Ctrl+S`).
6. Choisissez le **contenu à migrer** : *Fichiers*, *Fichiers + base de données* ou *Base de données seule*.
7. Suivez les étapes conseillées :
   1. **🔍 Vérifier** : contrôle des connexions, des dossiers et des outils, sans rien copier ;
   2. **Simulation** (case cochée) : affiche la liste des fichiers qui seraient copiés, sans rien écrire ;
   3. **Migration réelle** (case décochée) : copie effective.

La migration peut être relancée autant de fois que nécessaire. Avec rsync, seules les différences sont recopiées, ce qui permet une resynchronisation juste avant la bascule DNS.

Le bouton **■ Arrêter** interrompt proprement le transfert en cours ; les clés temporaires sont tout de même nettoyées. Le journal peut être enregistré dans un fichier (**Enregistrer le journal…**).

Un guide rapide est aussi disponible dans l'application : **Aide → Guide rapide** (`F1`).

---

## Référence de l'interface

### Menu Fichier

| Commande | Description |
|---|---|
| Nouvelle configuration | Repart d'une configuration vide |
| Ouvrir… (`Ctrl+O`) | Charge un autre fichier de configuration `.json` |
| Enregistrer (`Ctrl+S`) | Enregistre la configuration courante |
| Enregistrer sous… | Enregistre sous un autre nom (utile pour gérer plusieurs projets de migration) |

### Onglet Paramètres

**Transfert**

| Champ | Valeurs | Description |
|---|---|---|
| Méthode | `auto` · `rsync` · `tar` | `auto` : rsync si disponible des deux côtés, sinon tar |
| Sens | `auto` · `pull` · `push` | Serveur qui lance le transfert (voir ci-dessous) |
| Limite de débit | Ko/s | `0` = illimité (rsync uniquement) |
| Dossier exports SQL | chemin | Dossier des exports SQL sur la destination, relatif au dossier personnel de l'utilisateur SSH (défaut `db_export`) |
| Exclusions globales | une par ligne | Motifs exclus pour tous les sites (ex. `*.log`, `.well-known/acme-challenge/`) |

Sens du transfert :

- `pull` : la destination récupère les fichiers depuis la source ;
- `push` : la source envoie les fichiers vers la destination ;
- `auto` (par défaut) : `pull` si la destination possède un client `ssh`, sinon `push`.

**Serveur source / destination — valeurs par défaut**

Ces valeurs s'appliquent à tous les sites ; chaque site peut les redéfinir. Un champ vide dans un site reprend la valeur par défaut.

| Champ | Description |
|---|---|
| Type de serveur | Plesk, ISPConfig… : sert uniquement à l'affichage et aux exemples |
| Hôte (IP ou nom) | Adresse utilisée par le PC pour se connecter |
| Hôte de transfert | Adresse de ce serveur **vue depuis l'autre serveur** (IP privée, par exemple). Vide = même valeur que l'hôte |
| Port SSH | Port SSH (défaut 22) |

### Onglet Sites

Pour chaque site, les blocs **Source** et **Destination** contiennent :

| Champ | Description |
|---|---|
| Utilisateur SSH | Propriétaire du site sur ce serveur |
| Mot de passe | Mot de passe SSH (laisser vide si une clé privée est utilisée) |
| Clé privée (option) | Fichier de clé privée SSH, à la place du mot de passe (bouton **…** pour parcourir) |
| Dossier du site | Chemin absolu du dossier web |
| Type / Hôte / Port (vide = défaut) | À renseigner seulement si ce site est hébergé sur un autre serveur que celui défini dans **Paramètres** |

S'y ajoutent :

- **Base de données sur la source (optionnel)** : voir [Base de données](#base-de-données) ;
- **Exclusions propres à ce site** : ajoutées aux exclusions globales (ex. `cache/`).

Les boutons **Dupliquer** et **Supprimer** permettent de gérer rapidement la liste des sites. La case **Afficher les mots de passe** rend les champs de mot de passe lisibles.

### Options de lancement

| Option | Description |
|---|---|
| Contenu à migrer | *Fichiers*, *Fichiers + base de données* ou *Base de données seule* |
| Simulation | Affiche ce qui serait copié, sans rien écrire |
| Miroir exact | Supprime sur la destination les fichiers absents de la source (rsync uniquement, ignoré en mode tar) |

Un résumé par site est affiché dans le journal à la fin du traitement.

> **Avertissements tolérés.** rsync `24` et tar `1` indiquent que des fichiers ont été modifiés pendant la copie (site actif). La migration est considérée comme réussie avec un avertissement ; relancez-la pour resynchroniser.

---

## Base de données

Si le champ **Nom de la base** d'un site est renseigné et que le contenu à migrer est *Fichiers + base de données* ou *Base de données seule* :

1. la base est exportée **sur la source** avec `mysqldump`/`mariadb-dump` (`--single-transaction`, routines, triggers, utf8mb4) ou `pg_dump` (`--no-owner --no-privileges`), puis compressée en gzip ;
2. l'intégrité du dump est contrôlée (marqueur de fin de dump) ;
3. le fichier est envoyé directement sur la destination, dans le **Dossier exports SQL** (par défaut `~/db_export`, droits `700`), sous le nom `nom_base_AAAAMMJJ-HHMMSS.sql.gz` ;
4. la taille reçue est comparée à la taille envoyée.

Champs à renseigner : **Type** (MySQL / MariaDB ou PostgreSQL), **Hôte**, **Port**, **Nom de la base**, **Utilisateur**, **Mot de passe**. Nom vide = pas de base pour ce site.

**Aucun import n'est effectué** : il reste manuel. La commande à utiliser est affichée dans le journal, par exemple :

```bash
cd ~/db_export && gunzip < exemple_db_20260927-101500.sql.gz | mysql -u UTILISATEUR -p NOM_BASE
```

Le mot de passe de la base ne passe jamais en ligne de commande : il est écrit dans un fichier d'identifiants temporaire (droits `600`), supprimé à la fin. L'outil refuse aussi de placer l'export dans le dossier web du site, où il serait téléchargeable publiquement.

---

## Sécurité

- **`config.json` contient les mots de passe en clair.** Ne le partagez pas et ne le publiez pas ; préférez les clés SSH (**Clé privée**) aux mots de passe lorsque c'est possible.
- Les clés SSH temporaires sont uniques par site, identifiées par un marqueur `migr-tmp-<id>` et supprimées à la fin, même en cas d'erreur.
- Si le nettoyage échoue (message `ATTENTION` dans le journal), supprimez manuellement la ligne contenant `migr-tmp-` dans `~/.ssh/authorized_keys` du serveur concerné.
- L'option **Miroir exact** supprime des fichiers sur la destination : faites toujours une simulation avant.

---

## Dépannage

| Problème | Piste |
|---|---|
| Windows affiche « Windows a protégé votre ordinateur » | Cliquez sur **Informations complémentaires**, puis **Exécuter quand même** (l'exécutable n'est pas signé numériquement) |
| `Impossible d'écrire ~/.ssh/authorized_keys` | Ajoutez la clé publique affichée via le panneau (Plesk : *SSH Keys* ; cPanel : *Accès SSH* ; ISPConfig : Shell user) |
| Échec du test « connexion directe » | Vérifiez le pare-feu entre les deux serveurs et renseignez **Hôte de transfert** si les serveurs se voient par une autre adresse |
| `Ni rsync ni tar+gzip disponibles` | Installez rsync, ou activez un shell plus complet pour l'utilisateur |
| `Mode 'pull' impossible` | Pas de client ssh sur la destination : choisissez le **Sens** `auto` ou `push` |
| `L'utilisateur … ne peut pas écrire dans …` | Utilisez l'utilisateur propriétaire du site sur la destination |

---

## Auteur

© 2026 **RAKOTOSON Andriniaina Daniel**

- Site web : [andry-rakotoson.pro](https://andry-rakotoson.pro/)
- LinkedIn : [linkedin.com/in/root-andry](https://www.linkedin.com/in/root-andry/)
- GitHub : [github.com/root-andry](https://github.com/root-andry/)
- WhatsApp : [+261 34 27 045 45](https://wa.me/261342704545)
