# Déploiement GLPI (Docker + Portainer + Nginx Proxy Manager)

Environnement cible : hôte Docker `debian13-docker`, réseau `Docker_Net` existant, données persistantes sous `/volume2/docker/glpi/`, HTTPS géré en amont par Nginx Proxy Manager (NPM), stack géré via Portainer.

Architecture : 2 services — `glpi-app` (image officielle `glpi/glpi`, Apache + PHP, tâches cron intégrées) et `glpi-db` (MySQL 8.4). Les noms de services sont volontairement uniques : `Docker_Net` est partagé avec d'autres stacks, et un nom générique comme `db` peut y désigner plusieurs conteneurs à la fois. Aucun build : les images sont tirées depuis Docker Hub.

Points clés issus de la documentation officielle (`glpi-project/docker-images`) :

* Le conteneur GLPI tourne avec l'utilisateur non-root `www-data` (UID 33) : le dossier de données de l'hôte doit lui appartenir.
* Un seul volume `/var/glpi` contient `config`, `files`, `logs` et `marketplace` (donc les plugins installés via le Marketplace).
* GLPI s'installe tout seul au premier démarrage si les 5 variables `GLPI_DB_*` sont définies.

Fichiers :

| Fichier | Rôle |
|---------|------|
| [`docker-compose.yml`](docker-compose.yml) | Stack à coller dans Portainer |
| [`stack.env.example`](stack.env.example) | Variables du stack (mots de passe, port, chemins) |

> **Mots de passe** : ce dépôt est public, ils ne sont donc **pas** dans le compose. Ils sont passés en variables d'environnement du stack Portainer. Garde-les dans ton gestionnaire de mots de passe.

---

## 1. Créer les dossiers

```bash
mkdir -p /volume2/docker/glpi/data/{config,files,logs,marketplace}
mkdir -p /volume2/docker/glpi/db
```

## 2. Appliquer les droits

```bash
chown -R 33:33 /volume2/docker/glpi/data
```

Le dossier `db` n'a pas besoin de `chown` : l'image MySQL gère elle-même ses permissions.

## 3. Déployer le stack via Portainer

1. Générer deux mots de passe : `openssl rand -hex 16` (un pour `GLPI_DB_PASSWORD`, un pour `MYSQL_ROOT_PASSWORD`).
2. Portainer → **Stacks → Add stack** :
   * **Web editor** : coller [`docker-compose.yml`](docker-compose.yml) ;
   * ou **Repository** : URL `https://github.com/djoe08/my-glpi`, compose path `glpi/docker-compose.yml` (permet ensuite "Pull and redeploy" depuis Git).
3. **Environment variables** : ajouter `GLPI_DB_PASSWORD` et `MYSQL_ROOT_PASSWORD` (ou "Load variables from .env file" avec une copie remplie de `stack.env.example`).
4. **Deploy the stack**.

Si une variable obligatoire manque, le déploiement échoue avec le message `GLPI_DB_PASSWORD manquant` / `MYSQL_ROOT_PASSWORD manquant`.

## 4. Vérifier le démarrage

```bash
docker logs -f GLPI
```

Au premier démarrage, GLPI crée lui-même le schéma de base de données (auto-install). Attends la fin de l'installation avant de te connecter.

## 5. Configurer Nginx Proxy Manager

NPM et GLPI sont sur le même réseau `Docker_Net` : NPM peut joindre directement `GLPI:80` (port interne du conteneur). GLPI est aussi publié sur le port **9461** de l'hôte (`9461:80`), utile pour un accès direct `http://<ip-hôte>:9461` ou si tu préfères que NPM cible `<ip-hôte>:9461`.

Dans NPM → Proxy Hosts → Add Proxy Host :

| Champ               | Valeur                                                         |
| ------------------- | -------------------------------------------------------------- |
| Domain Names        | `glpi.jo.priv`                                                 |
| Scheme              | `http`                                                         |
| Forward Hostname/IP | `GLPI`                                                         |
| Forward Port        | `80`                                                           |
| SSL                 | Let's Encrypt ou certificat existant, activé sur ce Proxy Host |

## 6. Premier accès

* Interface : `https://glpi.jo.priv`
* Comptes par défaut créés par l'installation : `glpi` (super-admin), `tech`, `normal`, `post-only`. Le mot de passe initial de chacun est le même que le nom d'utilisateur (`glpi`/`glpi`, etc.).
* **À faire immédiatement** : changer les mots de passe de ces 4 comptes (ou désactiver les 3 qui ne servent pas).

## 7. Installer les plugins communautaires

### Méthode : le Marketplace intégré

Dans GLPI : **Configuration → Plugins → onglet Marketplace**. Pour chaque plugin : Télécharger → Installer → Activer.

C'est la bonne méthode ici, pour deux raisons :

* le Marketplace ne propose que des versions compatibles avec ta version de GLPI, donc pas de mauvaise surprise de compatibilité ;
* les plugins sont stockés dans `/var/glpi/marketplace`, donc dans `/volume2/docker/glpi/data/marketplace` : ils survivent aux mises à jour et aux redémarrages du conteneur.

### À ne PAS installer sur une nouvelle installation

| Plugin        | Raison                                                                                                                                    |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| Formcreator   | Remplacé par le système de formulaires natif de GLPI 11. La version 3.x n'est qu'un plugin de migration pour les anciennes installations. |
| GenericObject | Remplacé par les « actifs personnalisés » natifs de GLPI 11. La version 3.0.0 n'est qu'un plugin de migration en fin de vie.              |

### Plugins à essayer via le Marketplace

Le plugin **Fields** (champs personnalisés) a une version compatible GLPI 11 (1.22.0 et suivantes). Des sources secondaires indiquent aussi des versions compatibles pour **Escalade** et **DataInjection**. Vérifie leur disponibilité dans l'onglet Marketplace : s'ils apparaissent, ils sont compatibles avec ta version.

Il n'existe pas d'ensemble fixe de plugins à installer d'un bloc : le catalogue est large, une partie est abandonnée ou pas encore compatible GLPI 11, et certains font doublon avec des fonctions devenues natives. Le Marketplace fait office de filtre fiable.

---

## Mise à jour de GLPI

1. Sauvegarder la base et le dossier de données (voir plus bas).
2. Dans Portainer, éditer le stack et changer `GLPI_VERSION` (ou "Pull and redeploy" si tu restes sur `latest`).
3. La mise à jour du schéma de base est automatique au redémarrage (`GLPI_SKIP_AUTOUPDATE` n'étant pas défini).
4. Après une montée de version majeure, mettre à jour les plugins depuis le Marketplace.

## Sauvegarde

```bash
mkdir -p /mnt/backup_syno4/mes_images/glpi
docker exec GLPI-DB sh -c 'mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" glpi' > /mnt/backup_syno4/mes_images/glpi/glpi-db-$(date +%Y%m%d).sql
tar -czf /mnt/backup_syno4/mes_images/glpi/glpi-data-$(date +%Y%m%d).tar.gz -C /volume2/docker/glpi data
```

Le mot de passe root est lu dans l'environnement du conteneur : il n'apparaît ni dans la commande ni dans l'historique du shell.

---

## Points d'attention

* **Permissions du volume `db`** : si le conteneur `GLPI-DB` échoue avec « Operation not permitted », la cause côté hôte n'a jamais été identifiée. Dans ce cas, récupérer `docker logs GLPI-DB` et le résultat de `docker info | grep -i userns`.
* **Mots de passe** : jamais en clair dans le dépôt (public). Les conserver dans un gestionnaire de mots de passe.
* **Noms de services sur un réseau partagé** : sur `Docker_Net`, un nom de service générique (`db`, `redis`, `postgres`…) peut correspondre à plusieurs conteneurs de stacks différents, et GLPI se connecte alors tantôt à la bonne base, tantôt à une autre. Utiliser des noms préfixés par l'application.
* **Version de MySQL** : `mysql:8.4` est épinglé volontairement. Ne pas utiliser `mysql:latest`, qui peut changer de version majeure au redéploiement et rendre le dossier de données illisible.
* **Fuseaux horaires (optionnel)** : la prise en charge des timezones demande un accès SQL supplémentaire pour l'utilisateur `glpi` puis `bin/console database:enable_timezones` :

  ```bash
  docker exec GLPI-DB sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "GRANT SELECT ON mysql.time_zone_name TO '"'"'glpi'"'"'@'"'"'%'"'"'; FLUSH PRIVILEGES;"'
  docker exec GLPI /var/www/glpi/bin/console database:enable_timezones
  ```
