# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

Standalone Docker Compose stack that runs phpMyAdmin over HTTPS to administer the **host's** MariaDB. It is a developer convenience tool living inside the larger `oliver-kroener.de-typo3-14` monorepo (see the root `CLAUDE.md` for the TYPO3/Astro project). This directory is independent of DDEV — it talks to MariaDB on the Docker host, not to the DDEV `db` container.

## Commands

```bash
docker compose up -d        # Start phpMyAdmin
docker compose down         # Stop and remove
docker compose logs -f      # Tail logs (watch cert generation + apache startup)
docker compose up -d --force-recreate   # Apply changes to apache-ssl.conf or command
```

Access: `https://localhost:444` (self-signed cert — expect a browser warning).

## Architecture

Three files, no images built locally:

- **`docker-compose.yml`** — single `phpmyadmin:latest` service. Maps host `444` → container `443`. The `command:` overrides the default CMD to (1) generate the Debian *snakeoil* self-signed cert at the standard paths if absent (`make-ssl-cert generate-default-snakeoil`, falling back to `openssl`), (2) `a2enmod ssl`, then (3) hand back to the image's `/docker-entrypoint.sh apache2-foreground`. The entrypoint still runs, so phpMyAdmin's normal config generation is preserved.
- **`apache-ssl.conf`** — mounted read-only into `/etc/apache2/conf-enabled/`. Adds `Listen 443` and an SSL vhost pointing at the snakeoil cert/key. This is what turns the HTTP-only base image into HTTPS.
- **`.env`** — supplies `MYSQL_ROOT_PASSWORD` (consumed by `PMA_PASSWORD`). Empty by default → login screen prompts for the password.

### Key wiring decisions

- **Host DB reachability**: `PMA_HOST=host.docker.internal` + `extra_hosts: host.docker.internal:host-gateway`. The `host-gateway` mapping is required on Linux/WSL — without it the hostname does not resolve inside the container.
- **Root login**: `PMA_USER=root`, `PMA_ARBITRARY=0` (locks phpMyAdmin to the single configured host; no free-form server field).
- **Snakeoil generation, not host mount**: certs are created *inside* the container because the host has no `ssl-cert` package / snakeoil files. Do not switch to mounting `/etc/ssl/.../ssl-cert-snakeoil.*` from the host unless those files actually exist there.

## Gotchas

- Host MariaDB usually binds `127.0.0.1` and only grants `root@localhost`. For this container to connect, MariaDB must listen on `0.0.0.0` (or the docker bridge) and grant `root@'%'` (or the docker subnet).
- To point at a different DB (e.g. DDEV's `db` container), change `PMA_HOST`/`PMA_PORT` and the network setup — `host.docker.internal:3306` does not reach DDEV's internal network.
- Editing `apache-ssl.conf` or the inline `command` requires `--force-recreate`, not just `restart`.
