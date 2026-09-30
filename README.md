# AMPTemplates-FS25 — Farming Simulator 25 pour AMP (Debian/Linux)

> **Deux templates dans ce dépôt**, pour deux usages différents :
>
> | Template | À quoi il sert |
> |---|---|
> | `fs25` | Faire tourner FS25 **sous Wine/Docker**. Écrit en août 2026, quand le serveur dédié était Windows-only. **BETA.** |
> | `farming25teamkit` | Le template **officiel de CubeCoders** (Proton), avec le lien du panneau rabattu sur le reverse proxy HTTPS. C'est celui de la production TeamKit. |

Template **AMP Generic Module** pour héberger un serveur dédié **Farming Simulator 25** sur Linux,
via l'image Docker communautaire [wine-gameservers/arch-fs25server](https://github.com/wine-gameservers/arch-fs25server)
(Wine + VNC, par Toetje585). AMP pilote directement `docker run` : ports, volumes et variables
d'environnement sont gérés depuis le panel.

> ⚠️ **Statut : BETA** — le serveur dédié FS25 est Windows-only, l'exécution sous Wine est
> communautaire et non supportée par GIANTS. Peut casser lors des mises à jour du jeu.

---

# Template `fs25` — Wine/Docker (BETA)

## Prérequis

1. **Licence serveur GIANTS** : le jeu doit être acheté sur le **site GIANTS** (la version Steam
   ne peut pas servir de serveur dédié) — c'est une licence **en plus** de celle du joueur.
2. **Docker** installé sur la machine AMP (`docker --version`), et l'utilisateur `amp` dans le
   groupe docker : `sudo usermod -aG docker amp` puis redémarrer AMP.
3. ~65 Go de disque libre par serveur (jeu + DLC + sauvegardes).

## Installation

### 1. Ajouter le template dans ADS
Configuration → Instance Deployment → Configuration Repositories → ajouter :
```
hydrocut/AMPTemplates-FS25:main
```
puis **Fetch Latest**.

### 2. Créer l'instance
Create Instance → **Farming Simulator 25**. Après création :
- Champ **02/03** : mettre l'UID/GID de l'utilisateur `amp` (commande `id amp` sur l'hôte)
- Champ **01** : nom de conteneur **unique** par instance
- Laisser **04 - Démarrage auto du jeu** sur `false` pour l'instant
- Cliquer **Update** (crée les dossiers + télécharge l'image Docker, ~2 Go)

### 3. Déposer les fichiers du jeu
Depuis le portail GIANTS, télécharger le **zip du jeu** (et les DLC). Via le File Manager
de l'instance (ou SFTP) :
- contenu du zip du jeu → dossier `installer/`
- fichiers DLC → dossier `dlc/`

### 4. Installation graphique via VNC
**Start** sur l'instance, puis ouvrir dans un navigateur : `http://IP-DU-SERVEUR:PORT-VNC-WEB`
(port « VNC (navigateur) » dans la section Ports d'AMP ; mot de passe = champ 32).
Dérouler l'assistant : installation du jeu, **activation de la clé GIANTS**, DLC.
Compter ~20 minutes.

### 5. Passer en mode production
- Champ **04 - Démarrage auto du jeu** → `true`
- **Restart** de l'instance
- Le panel web du serveur FS25 est sur le port « Panel Web FS25 » (identifiants champs 30/31) :
  c'est là que se gèrent la sauvegarde, les mods et le démarrage de la partie.

## Ports (attribués par AMP)

| Rôle | Port conteneur | Réf AMP |
|---|---|---|
| Jeu (TCP+UDP) | 10823 | GamePort |
| Panel web FS25 | 7999 | WebPanelPort |
| noVNC (navigateur) | 6080 | VncWebPort |
| VNC (client) | 5900 | VncPort |

Ouvrir le port de jeu (TCP+UDP) dans le firewall. Les ports VNC ne doivent **pas** être exposés
publiquement une fois l'installation terminée (règle firewall ou accès via VPN/tunnel).

## Notes

- Les données persistantes (jeu installé, config, sauvegardes, DLC) vivent dans le datastore de
  l'instance (`fs25/config`, `fs25/game`, `fs25/dlc`, `fs25/installer`) → les backups AMP les couvrent.
- L'étape d'update « Nettoyage conteneur résiduel » supprime un conteneur resté bloqué après un
  crash : en cas de démarrage impossible avec une erreur « name already in use », lancer Update.
- Crédits : image Docker par [Toetje585 / wine-gameservers](https://github.com/wine-gameservers/arch-fs25server) —
  template AMP par TeamKit.fr.

---

# Template `farming25teamkit` — officiel + proxy HTTPS

Copie **conforme** du template officiel `farming-simulator-25` de CubeCoders,
avec une seule différence de fond : le lien que le panel AMP affiche.

## Le problème

AMP construit le « Connection Link » d'une instance à partir de
`Meta.EndpointURIFormat`. Le template officiel y met :

```
Meta.EndpointURIFormat=http://{ip}:{GenericModule.App.Ports.$WebInterfacePort}
```

Sur notre dédié, ça donne un lien **inutilisable**, pour trois raisons qui
s'additionnent :

1. l'instance est un conteneur **ponté** → `{ip}` vaut son adresse Docker
   interne, `172.17.0.6`, que personne ne peut joindre depuis l'extérieur ;
2. le port 8102 est **fermé au monde** depuis le 29 septembre 2026 → même
   avec l'IP publique, le lien tombe en time-out ;
3. c'est du **HTTP en clair** → les identifiants du panneau circulaient nus.

## Pourquoi un template, et pas une simple correction

On a d'abord corrigé la valeur directement dans le `GenericModule.kvp` de
l'instance. **Elle est revenue au bout d'un jour.**

`Meta.*` est une métadonnée de **template**, pas de l'instance. ADS retire
les dépôts de templates (un `git pull` sur chacun), puis AMP re-remplit le
`GenericModule.kvp` de l'instance depuis le template d'origine — nommé dans
`Meta.OriginalSource`. Tant que cette ligne dit `CubeCoders-AMPTemplates-main`,
toute correction locale est écrite sur du sable.

D'où ce template : la valeur est à la source, donc elle tient.

## Ce qui ne change PAS, et c'est le point important

| Clé | Valeur | Conséquence |
|---|---|---|
| `App.RootDir` / `App.BaseDirectory` | `./farming-simulator-25/` | les dizaines de Go de jeu déjà installés ne bougent pas |
| `Meta.ConfigRoot` | `farming-simulator-25.kvp` | les réglages du serveur saisis dans AMP sont conservés |

Les manifestes de configuration sont des **copies à l'octet** de ceux de
CubeCoders : mêmes réglages, mêmes noms de champs, même ordre. C'est ce qui
rend le template *interchangeable* avec l'officiel.

## Basculer une instance existante

⚠️ **L'instance doit être ARRÊTÉE.** AMP réécrit ses réglages en s'éteignant :
éditer le `.kvp` d'une instance en marche, c'est perdre l'édition au prochain
arrêt. C'est précisément comme ça qu'on a perdu la première correction.

Trois lignes à changer dans
`~amp/.ampdata/instances/<Instance>/GenericModule.kvp` :

```
Meta.ConfigManifest=farming25teamkitconfig.json
Meta.MetaConfigManifest=farming25teamkitmetaconfig.json
Meta.OriginalSource=hydrocut-AMPTemplates-FS25-main
```

Rien d'autre. Ni `ConfigRoot`, ni `RootDir`, ni les ports, ni les réglages du
jeu. Au démarrage suivant, AMP relit les métadonnées depuis **ce** template et
affiche `https://farming25.teamkit.fr`.

## Et le reverse proxy

Le template suppose qu'un nginx fait déjà le pont vers le panneau :
`dedie/farming25.teamkit.fr.conf` dans le dépôt TeamKit. Un piège y est
documenté et vaut d'être connu : **HTTP/2 impose des noms d'en-têtes en
minuscules**, nginx les transmet tels quels, et le serveur web embarqué de
GIANTS compare les noms **en respectant la casse** — il ne reconnaissait donc
pas `content-type`, ne lisait jamais le corps du formulaire, et la connexion
au panneau échouait silencieusement. D'où les `proxy_set_header` qui
remettent les majuscules.

Si tu réutilises ce template ailleurs, change `Meta.EndpointURIFormat` pour
ton propre nom de domaine : c'est la seule ligne qui soit spécifique à TeamKit.

## Crédits

Template officiel par **IceOfWraith** pour CubeCoders
([AMPTemplates](https://github.com/CubeCoders/AMPTemplates)). Cette variante
ne fait que rabattre le lien affiché sur un proxy HTTPS.
