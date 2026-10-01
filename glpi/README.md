# GLPI – déploiement Docker Compose

Stack : image officielle [`glpi/glpi`](https://hub.docker.com/r/glpi/glpi) (11.0) + MariaDB 11.4.

## Démarrage

```bash
cd glpi
cp .env.example .env      # puis modifier les mots de passe
docker compose up -d
docker compose logs -f glpi   # suivre l'auto-installation
```

GLPI est accessible sur `http://<serveur>:8080` (port modifiable via `GLPI_HTTP_PORT`).

Comptes par défaut créés à l'installation (à changer immédiatement) :

| Login     | Mot de passe |
|-----------|--------------|
| glpi      | glpi         |
| tech      | tech         |
| normal    | normal       |
| post-only | postonly     |


## Fuseaux horaires (optionnel)

```bash
docker compose exec db mariadb -uroot -p"$DB_ROOT_PASSWORD" \
  -e "GRANT SELECT ON mysql.time_zone_name TO 'glpi'@'%'; FLUSH PRIVILEGES;"
docker compose exec glpi /var/www/glpi/bin/console database:enable_timezones
```

## Sauvegarde

```bash
docker compose exec db sh -c 'mariadb-dump -uroot -p"$MARIADB_ROOT_PASSWORD" glpi' > glpi-$(date +%F).sql
docker run --rm -v glpi_glpi_data:/data -v "$PWD":/backup alpine tar czf /backup/glpi-data-$(date +%F).tgz -C /data .
```

## Mise à jour

Changer `GLPI_VERSION` dans `.env`, puis :

```bash
docker compose pull && docker compose up -d
```

Le schéma de base est mis à jour automatiquement (sauf si `GLPI_SKIP_AUTOUPDATE=true`).

## Structure

- `docker-compose.yml` – services `glpi` et `db`
- `.env.example` – variables (copier en `.env`, non versionné)
- `php/custom.ini` – surcharge PHP
- `plugins/` – plugins GLPI personnalisés
