# Ansible Matomo Deployment Playbooks

**Always follow these instructions first and only fallback to additional search and context gathering if the information in these instructions is incomplete or found to be in error.**

This repository contains Ansible playbooks for deploying Matomo (web analytics platform) with MariaDB, nginx, SSL certificates, GeoIP2 integration, and phpMyAdmin on Ubuntu servers. It includes both initial deployment and database replication functionality.

## Working Effectively

### Prerequisites and Dependencies
- Ansible is already available (version 2.18.8) 
- **LIMITATION**: Required collections (`community.crypto`, `community.mysql`, `community.general`) may not be accessible due to network restrictions. Some syntax checks will fail.
- Install additional linting tools if needed:
  ```bash
  pip install ansible-lint
  ```
- **Network restriction**: `ansible-galaxy collection install` commands will fail due to sandbox limitations

### Basic Setup and Configuration
- Always work from the `ansible/` directory: `cd /home/runner/work/ansible/ansible/ansible`
- Copy the environment template: `cp .env.example .env`
- Edit `.env` with your specific settings (domain, passwords, MaxMind credentials)
- Load environment variables: `export $(grep -v '^#' .env | xargs)`
- Edit `inventory.ini` to configure target servers

### Validation and Testing Commands
Run these commands before making changes:
- **Syntax validation**: `ansible-playbook -i inventory.ini --syntax-check -e target_group=local site.yml` (< 1 second, may fail on community modules due to network restrictions)
- **YAML linting**: `yamllint .` (< 1 second, will show warnings and errors) 
- **Ansible linting**: `ansible-lint prerequisites.yml` (< 5 seconds, test individual playbooks)
- **Dry run test**: `ansible-playbook -i inventory.ini --check -e target_group=local prerequisites.yml` (< 5 seconds, NEVER CANCEL)

**NOTE**: Full site.yml syntax check and ansible-lint will fail due to missing community collections in sandbox environment.

### Deployment Commands
**WARNING**: Full deployment requires actual Ubuntu servers and takes 45+ minutes. NEVER CANCEL builds.

For local testing (limited functionality in sandbox):
```bash
export $(grep -v '^#' .env | xargs)
ansible-playbook -i inventory.ini -e target_group=local site.yml
```

For remote servers:
```bash
export $(grep -v '^#' .env | xargs) 
ansible-playbook -i inventory.ini -e target_group=matomo_servers site.yml
```

Expected deployment time: **45-60 minutes minimum**. NEVER CANCEL. Set timeout to 90+ minutes.

### Individual Playbooks
You can run individual components separately:
- `prerequisites.yml` - Install system dependencies (< 5 minutes)
- `extras.yml` - Install Fail2ban security (< 2 minutes) 
- `nginx_plain.yml` - Install basic nginx (< 5 minutes)
- `ssl.yml` - Configure SSL certificates (< 10 minutes, requires nginx)
- `nginx_reinstall.yml` - Rebuild nginx with GeoIP2 (< 20 minutes, NEVER CANCEL)
- `maria_db.yml` - Install and configure MariaDB (< 10 minutes)
- `geoupdate.yml` - Install GeoIP2 database updates (< 5 minutes)
- `php.yml` - Install PHP and extensions (< 5 minutes)
- `php_applications.yml` - Install Matomo and phpMyAdmin (< 10 minutes)
- `save_configurations.yml` - Save final configurations (< 2 minutes)

## Database Replication
Located in `replicate_db/` directory - separate functionality for migrating existing Matomo installations:

Setup commands:
```bash
cd replicate_db/
ansible-galaxy collection install community.mysql community.general ansible.posix
```

Main replication playbook (NEVER CANCEL - can take 30+ minutes):
```bash
ansible-playbook -i inventory.ini replicate_db.yml -e target_group=local
```

## Validation Scenarios

**CRITICAL**: Always test these scenarios after making changes to ensure functionality:

### Basic Playbook Validation
1. **Syntax validation**: Run syntax check on all modified playbooks
2. **YAML linting**: Ensure YAML format is correct with `yamllint`
3. **Variable resolution**: Test that environment variables load properly
4. **Inventory parsing**: Verify inventory file is valid

### Configuration Testing  
1. **Environment setup**: Verify `.env` file loading works correctly
2. **Template rendering**: Check that Jinja2 templates in `templates/` compile
3. **Variable substitution**: Ensure all required environment variables are defined

**NOTE**: Full end-to-end deployment testing requires actual Ubuntu servers and is not possible in sandbox environment. Network limitations prevent downloading external packages and configuring services.

## Common Tasks and File Locations

### Repository Structure
```
.
├── README.md                    # Main documentation
├── ansible/
│   ├── .env.example            # Environment template  
│   ├── inventory.ini           # Server inventory
│   ├── site.yml               # Main orchestration playbook
│   ├── variables.yml          # Variable definitions
│   ├── prerequisites.yml      # System dependencies
│   ├── extras.yml             # Security tools (Fail2ban)
│   ├── nginx_plain.yml        # Basic nginx installation
│   ├── ssl.yml                # SSL certificate setup
│   ├── nginx_reinstall.yml    # Nginx rebuild with GeoIP2
│   ├── maria_db.yml           # MariaDB installation
│   ├── geoupdate.yml          # GeoIP2 database setup
│   ├── php.yml                # PHP and extensions
│   ├── php_applications.yml   # Matomo and phpMyAdmin
│   ├── save_configurations.yml # Final configuration
│   ├── templates/             # Jinja2 configuration templates
│   └── replicate_db/          # Database replication playbooks
```

### Key Configuration Files
- **Main playbook**: `ansible/site.yml` 
- **Environment variables**: `ansible/.env.example` (copy to `.env`)
- **Server inventory**: `ansible/inventory.ini`
- **Variable definitions**: `ansible/variables.yml`
- **Nginx templates**: `ansible/templates/`

### Frequently Used Commands
```bash
# Always run from ansible/ directory
cd /home/runner/work/ansible/ansible/ansible

# Basic validation sequence
cp .env.example .env
export $(grep -v '^#' .env | xargs)

# Test individual playbooks that work in sandbox
ansible-playbook -i inventory.ini --syntax-check -e target_group=local prerequisites.yml
ansible-playbook -i inventory.ini --syntax-check -e target_group=local extras.yml  
ansible-playbook -i inventory.ini --syntax-check -e target_group=local nginx_plain.yml
ansible-playbook -i inventory.ini --syntax-check -e target_group=local geoupdate.yml
ansible-playbook -i inventory.ini --syntax-check -e target_group=local php.yml
ansible-playbook -i inventory.ini --syntax-check -e target_group=local php_applications.yml

# YAML linting (works for all files)
yamllint .

# Dry run testing (safe in sandbox for these playbooks)
ansible-playbook -i inventory.ini --check -e target_group=local prerequisites.yml
ansible-playbook -i inventory.ini --check -e target_group=local extras.yml

# Do NOT test these due to missing collections:
# ssl.yml, nginx_reinstall.yml, maria_db.yml, save_configurations.yml, site.yml
```

## Limitations and Warnings

### Sandbox Environment Limitations
- **Network restrictions**: Cannot download external packages (nginx source, MaxMind databases) or access ansible-galaxy
- **Missing collections**: Community collections are not accessible, causing syntax check failures on ssl.yml, maria_db.yml, and save_configurations.yml
- **Service management**: Cannot start/stop system services like nginx, MariaDB
- **SSL certificate generation**: Cannot obtain real SSL certificates from Let's Encrypt
- **Full deployment impossible**: Use dry run mode (`--check`) for testing changes on individual playbooks only

### Build and Timing Expectations  
- **Syntax checks**: < 1 second per playbook
- **Linting**: < 30 seconds for full repository
- **Dry run testing**: < 30 seconds (safe in sandbox)
- **Full deployment**: 45-90 minutes (requires real servers, NEVER CANCEL)
- **Database replication**: 30-60 minutes (NEVER CANCEL)

### Required External Dependencies
- MaxMind GeoIP2 account and license key (free account sufficient)
- Ubuntu 24.04 target servers with sudo access
- SSL certificate email address for Let's Encrypt

Always verify changes with syntax validation and linting before attempting deployment.