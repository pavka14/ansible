# Django Deployment Playbooks

This directory contains modular Ansible playbooks for deploying a Django application stack on Ubuntu 24.04. The stack includes:
- PostgreSQL database
- Python (latest) in a virtual environment
- Redis server
- Nginx (with SSL and GeoIP2 integration)
- Uvicorn (for serving Django)
- Automated download and setup of GeoIP2 databases and module
- GitHub repository cloning (using credentials from environment)

## Features
- **Modular design**: Each playbook can be run independently for isolated component deployment or troubleshooting.
- **Environment-driven configuration**: All sensitive and configurable values are loaded from the `.env` file.
- **GeoIP2 support**: Nginx is recompiled with the GeoIP2 module and configured to use MaxMind databases.
- **SSL support**: Automated Let's Encrypt certificate acquisition and renewal.

Note that Nginx gets installed with a config file defined in the templates/nginx.conf.j2 jinja template, which is somewhat simplistic (e.g. has no serving of static files). You will probably want to modify it to suit your particular case.
If you want country and city names in a language other than English, the options to change that are in the geoip2_nginx.conf.j2 template.

## Prerequisites
- Ubuntu 24.04 target server(s) with sudo access
- Ansible 2.18.8 or newer (on host system)
- MaxMind GeoIP2 account and license key
- GitHub access token for private repo cloning (if needed)

### Host System Prerequisites
These steps must be performed on the machine where you run Ansible (the control node):

1. **Install Ansible**
   ```bash
   python3 -m pip install --user ansible
   # Or use your OS package manager (e.g. apt, dnf, brew)
   # Ubuntu example:
   sudo apt update && sudo apt install ansible
   ```

2. **Install the community.postgresql collection**
   ```bash
   ansible-galaxy collection install community.postgresql
   ```
   This is required for database setup tasks in `postgres.yml`.

## Setup Instructions

1. **Copy and edit environment file**
   ```bash
   cp .env.example .env
   # Edit .env with your actual values (domain, passwords, MaxMind credentials, etc.)
   ```

2. **Export environment variables**
   ```bash
   export $(grep -v '^#' .env | xargs)
   ```

3. **Edit inventory**
   - Edit `inventory.ini` to list your target server(s) under the `[django_servers]` group.

## Running the Full Deployment

To run the entire stack in order:
```bash
ansible-playbook -i inventory.ini site.yml
```
This will execute all component playbooks in sequence, deploying the full Django stack.

## Running a Playbook in Isolation

Each playbook is self-contained and can be run independently. For example, to run only the Nginx setup:
```bash
ansible-playbook -i inventory.ini nginx.yml
```
Or to run only the database setup:
```bash
ansible-playbook -i inventory.ini postgres.yml
```

## Validation and Testing

Before running any playbook, always validate syntax and variable resolution:
```bash
export $(grep -v '^#' .env | xargs)
ansible-playbook -i inventory.ini --syntax-check <playbook.yml>
```
For a dry-run (no changes made):
```bash
export $(grep -v '^#' .env | xargs)
ansible-playbook -i inventory.ini --check <playbook.yml>
```

## Playbook List
- `prerequisites.yml` - System packages and fail2ban
- `postgres.yml` - PostgreSQL install and config
- `python.yml` - Python and virtualenv setup
- `redis.yml` - Redis install and config
- `nginx.yml` - Nginx install, SSL, GeoIP2 module compilation, config
- `geoupdate.yml` - GeoIP2 database download, configuration, and scheduled updates
- `django.yml` - Django app deployment, repo clone, migrations, static files, Uvicorn
- `ssl.yml` - Let's Encrypt certificate setup
- `uvicorn.yml` - Uvicorn systemd service setup
- `site.yml` - All of the above in sequence

## Notes
- All configuration values (paths, credentials, repo URLs, etc.) are loaded from `.env` via `variables.yml`.
- GeoIP2 databases are downloaded using your MaxMind credentials.
- Nginx is recompiled with the GeoIP2 module for IP geolocation support.

Playbooks are mostly self-contained, but:
- ssl should run after nginx (confirming domain ownership needs a running web server)
- uvicorn should run after django (the app directory is referenced in the service config, and also a change is made in the django virtual environment)

For uvicorn, you can add its own .env file by adding:
EnvironmentFile={{ django_app_dir }}/.env
to the uvicorn.service.j2 template (and then make sure the file is there).
The current setup does not have any environment variables for uvicorn, and supposes that django loads them after it starts with load_env().
You may need uvicorn variables if you want anything to be set before django starts.

## Troubleshooting
- Ensure all required environment variables are set in `.env`.
- Validate each playbook with `--syntax-check` before running.
- Check Ansible output for missing variables or connection errors.

---

For more details, see the main repository README and the comments in each playbook.
