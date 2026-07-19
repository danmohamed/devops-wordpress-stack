# DevOps WordPress Stack

A Dockerized WordPress deployment with an Nginx reverse proxy, built and debugged during my DevOps internship. This project started as a manual LEMP stack setup in WSL/Ubuntu and was later migrated to a fully containerized architecture.

## Architecture

```
                    ┌─────────────┐
   Browser  ───────▶│    Nginx    │  (reverse proxy, port 80/443)
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  WordPress  │  (PHP-FPM)
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │    MySQL    │  (persistent volume)
                    └─────────────┘
```

- **Nginx** handles incoming requests and reverse-proxies them to the WordPress container over PHP-FPM.
- **WordPress** runs as its own container, decoupled from the web server.
- **MySQL** stores all WordPress data in a named Docker volume so data survives container restarts/rebuilds.

## Tech Stack

- Docker & Docker Compose
- Nginx (reverse proxy + PHP-FPM upstream)
- WordPress (PHP-FPM variant)
- MySQL
- WSL2 / Ubuntu (development environment)
- Ansible (optional — used for provisioning/automation)

## Getting Started

### Prerequisites
- Docker & Docker Compose installed
- WSL2 with Ubuntu (if on Windows)

### Setup

1. Clone the repo:
   ```bash
   git clone https://github.com/danmohamed/devops-wordpress-stack.git
   cd devops-wordpress-stack
   ```

2. Copy the example environment file and fill in your own values:
   ```bash
   cp .env.example .env
   ```

3. Start the stack:
   ```bash
   docker compose up -d
   ```

4. Visit `http://localhost` (or your configured domain) to complete the WordPress install.

### Stopping the stack
```bash
docker compose down
```
Add `-v` to also remove volumes (this will delete your database data).

## Issues Encountered & Fixed

This project involved real troubleshooting during setup — documented here for reference:

- **MySQL authentication errors** — Resolved a mismatch between the MySQL auth plugin and WordPress's connection method by adjusting the container's auth configuration.
- **Nginx symlink issues** — Fixed broken symlinks between `sites-available` and `sites-enabled` that were preventing the reverse proxy config from loading.
- **WordPress URL misconfiguration** — Corrected `siteurl`/`home` values in the WordPress database that were pointing to the wrong host after migrating from a manual LEMP setup to Docker.
- **Credential syntax errors** — Fixed malformed environment variable syntax in `.env` that was silently breaking the MySQL container's startup.

See [`docs/troubleshooting.md`](docs/troubleshooting.md) for full details on each issue and how it was diagnosed.

## Project Structure

```
devops-wordpress-stack/
├── docker-compose.yml
├── .env.example
├── nginx/
│   └── conf.d/
│       └── wordpress.conf
├── ansible/
│   ├── playbook.yml
│   └── inventory.ini
├── docs/
│   └── troubleshooting.md
└── README.md
```

## Roadmap / Future Improvements

- [ ] Add Prometheus + Grafana monitoring for container health
- [ ] Automate provisioning fully with Ansible
- [ ] Add HTTPS via Let's Encrypt/Certbot
- [ ] CI pipeline for config validation

## License

MIT
