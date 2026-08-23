# AMPTemplates-FS25 — Farming Simulator 25 pour AMP (Debian/Linux)

Template **AMP Generic Module** pour héberger un serveur dédié **Farming Simulator 25** sur Linux,
via l'image Docker communautaire [wine-gameservers/arch-fs25server](https://github.com/wine-gameservers/arch-fs25server)
(Wine + VNC, par Toetje585). AMP pilote directement `docker run` : ports, volumes et variables
d'environnement sont gérés depuis le panel.

> ⚠️ **Statut : BETA** — le serveur dédié FS25 est Windows-only, l'exécution sous Wine est
> communautaire et non supportée par GIANTS. Peut casser lors des mises à jour du jeu.

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
