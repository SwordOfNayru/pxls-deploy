# Déploiement — Catboat Place

Stack de production pour Pxls : back end Java, front end Node, PostgreSQL et
terminaison HTTPS. Les deux images sont construites depuis les sources de ce
dépôt (Maven pour le back, gulp pour le front) ; rien n'est tiré d'un registre
public en dehors de `postgres`, `caddy` et des images de base.

```
Internet ──443──▶ caddy ──▶ pxls-web:3000 ──▶ pxls-server:4567 ──▶ postgres:5432
                  TLS        dist/ + proxy      canevas, API, WS      comptes, pixels
```

Seul `caddy` publie des ports. Les trois autres services ne sont joignables que
depuis le réseau Docker interne.

## Mise en route

1. **Prérequis** : Docker avec le plugin Compose, un nom de domaine dont
   l'enregistrement DNS pointe vers la machine, et les ports 80 et 443 ouverts
   (Caddy en a besoin pour obtenir puis renouveler le certificat Let's Encrypt).

2. **Configuration** :

   ```sh
   cd deploy
   cp .env.example .env
   $EDITOR .env          # domaine, mot de passe Postgres, clés OAuth, PUID/PGID
   ```

   `.env` contient les secrets et n'est pas versionné. Le reste des réglages du
   jeu (cooldown, taille du canevas, chat, filtres, factions…) se trouve dans
   `config/pxls.conf`, qui ne contient aucun secret et peut être commité.

3. **Démarrage** :

   ```sh
   docker compose up -d --build
   docker compose logs -f
   ```

   Le premier build compile le jar avec Maven : comptez quelques minutes. Les
   suivants réutilisent le cache et ne recompilent que si les sources changent.

Le serveur crée lui-même ses tables au premier démarrage : il n'y a aucune
migration à lancer.

## Ce que fait le démarrage

`config/` est monté en lecture seule sous `/config` dans le conteneur. À chaque
démarrage, l'entrypoint :

- remplace les `${PXLS_*}` de `config/pxls.conf` par les variables
  d'environnement et écrit le résultat dans `/data/pxls.conf` ;
- copie `roles.conf` et `palette.conf` tels quels ;
- copie `config/default_board.dat` s'il existe et qu'aucun n'est déjà en place ;
- ajuste `data/` à `PUID:PGID` puis abandonne les droits root.

Le front end, lui, génère son `config.json5` depuis les variables `WEB_*`.

## Exploitation

**Console d'administration** (`ban`, `roles`, `alert`, `nuke`… voir le README de
`Pxls/`) :

```sh
docker compose attach pxls-server
```

Se détacher avec `Ctrl-P` `Ctrl-Q`. `Ctrl-C` arrêterait le serveur.

**Changer un réglage de jeu** — éditer `config/pxls.conf` puis :

```sh
docker compose restart pxls-server
```

La commande `reload` de la console recharge la plupart des clés sans
redémarrage, mais elle lit `/data/pxls.conf` : il faut donc quand même que
l'entrypoint l'ait régénéré, c'est-à-dire redémarrer le conteneur.

**Mettre à jour le code** :

```sh
git -C ../Pxls pull && git -C ../pxls-web pull
docker compose up -d --build
```

**Sauvegardes** — `data/` appartient à `PUID:PGID` et se sauvegarde directement
(`board.dat`, `backups/`, `logs/`). Pour la base :

```sh
docker compose exec -T postgres pg_dump -U pxls pxls | gzip > pxls-$(date +%F).sql.gz
```

Restauration :

```sh
gunzip -c pxls-2026-01-01.sql.gz | docker compose exec -T postgres psql -U pxls pxls
```

**Nouvelle toile** — incrémenter `PXLS_CANVAS_CODE` dans `.env`, archiver
`data/board.dat`, le supprimer, puis redémarrer `pxls-server`.

## Reprendre les données de `servtest`

Les fichiers de `servtest/` sont directement réutilisables :

```sh
mkdir -p data
cp ../servtest/board.dat ../servtest/heatmap.dat \
   ../servtest/placemap.dat ../servtest/virginmap.dat data/
cp ../servtest/palette.conf ../servtest/roles.conf config/
```

Ne pas copier `servtest/pxls.conf` : `config/pxls.conf` en est déjà dérivé, avec
les secrets extraits vers `.env`.

## Adresses de redirection OAuth

À déclarer chez chaque fournisseur, en remplaçant le domaine :

| Service | URL de redirection |
| --- | --- |
| Discord | `https://place.example.com/auth/discord` |
| Google  | `https://place.example.com/auth/google` |
| Reddit  | `https://place.example.com/auth/reddit` |

## IP réelle des joueurs

Le back end applique ses limites de débit et ses bannissements par adresse IP.
La chaîne est : Caddy pose `X-Real-IP` avec l'IP réelle du client (en écrasant
toute valeur envoyée par celui-ci), pxls-web relaie l'en-tête, et pxls-server ne
l'accepte que s'il provient de `PXLS_WEB_IP` — d'où l'IP fixe attribuée au
conteneur front.

Si vous changez `PXLS_SUBNET`, `PXLS_WEB_IP` doit rester dans ce sous-réseau.
Sinon le serveur verra tous les joueurs comme une seule adresse : le premier
banni les bannirait tous.

## Dépannage

| Symptôme | Cause probable |
| --- | --- |
| `variable d'environnement obligatoire manquante` | `.env` incomplet ; comparer avec `.env.example`. |
| Caddy boucle sur l'obtention du certificat | DNS qui ne pointe pas ici, ou port 80 filtré. |
| Aucun bouton de connexion | Aucun `PXLS_*_ENABLED` à `true`. |
| Connexion qui renvoie vers `localhost` | `PXLS_DOMAIN` erroné : il sert à construire `callbackBase`. |
| `data/` illisible depuis l'hôte | `PUID`/`PGID` ne correspondent pas à `id -u` / `id -g`. |
