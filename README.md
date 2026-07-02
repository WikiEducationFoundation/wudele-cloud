wudele-cloud
============

Ansible provisioning and deployment for [Wudele](https://github.com/WikiEducationFoundation/wudele)
(a fork of [Pollaris](https://framagit.org/pollaris/pollaris) with timezone-aware
slots), running on a Wikimedia Cloud VPS instance at
[wudele.wmcloud.org](https://wudele.wmcloud.org).

This replaces the Framadate-based [wudele-toolforge](https://github.com/JeanFred/wudele-toolforge).

## Architecture

One Debian 13 instance (`wudele.globaleducation.eqiad1.wikimedia.cloud`) runs the
whole stack:

- **nginx** serving `/srv/wudele/app/public`, behind the shared Cloud VPS web
  proxy (which terminates TLS for wudele.wmcloud.org)
- **PHP 8.4 FPM** with a dedicated `wudele` pool running as the `wudele` user
- **PostgreSQL 17** (local), database and role `wudele`
- **wudele-worker** systemd service running the Symfony Messenger worker
  (async jobs and scheduled tasks such as poll expiration)
- Email features disabled entirely (`APP_EMAILS_ENABLED=false` and a null
  mailer transport), like the original wudele.toolforge.org
- Nightly `pg_dump` backups to `/srv/backups/wudele` (14 days retention)

Secrets (Symfony `APP_SECRET`, database password) are generated on the host on
first provisioning and stored under `/srv/wudele/secrets/`; nothing secret
lives in this repository.

Production assets are pre-built and committed in the application repository
(upstream Pollaris convention), so the instance does not need Node.js. When
changing JS/CSS, rebuild and commit before deploying:

```console
$ docker compose -f docker/development/docker-compose.yml run --rm bundler npm run build
```

## One-time Horizon setup (not automated)

1. Create the instance (Debian 13, g4.cores1.ram2.disk20 or larger) — done.
2. Create a web proxy `wudele.wmcloud.org` → the instance, port 80.
3. Make sure the project security groups allow HTTP (port 80) from the web
   proxy to the instance (the default security group usually does).

## Usage

Requirements: `ansible-core`, plus SSH access to the instance through the
Cloud VPS bastion (see `hosts`; set `bastion_user` to your own shell
username).

Full provisioning (idempotent):

```console
$ ansible-playbook main.yml
```

Deploy the latest code from the `wudele` branch:

```console
$ ansible-playbook main.yml --tags application
```

Other tags: `system`, `database`, `webserver`, `worker`, `backups`.
