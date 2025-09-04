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
- Always work from the `ansible/deploy-matomo/` directory: `cd /home/runner/work/ansible/ansible/ansible/deploy-matomo`
- Copy the environment template: `cp .env.example .env`
- Edit `.env` with your specific settings (domain, passwords, MaxMind credentials)
- Load environment variables: `export $(grep -v '^#' .env | xargs)`
- Edit `inventory.ini` to configure target servers

### Validation and Testing Commands
Run these commands before making changes:
- **Syntax validation**: `ansible-playbook -i inventory.ini --syntax-check <playbook.yml>` (< 1 second, may fail on community modules due to network restrictions)
- **YAML linting**: `yamllint .` (< 1 second, will show warnings and errors) 
- **Ansible linting**: `ansible-lint <playbook.yml>` (< 5 seconds, test individual playbooks)
- **Dry run validation**: `ansible-playbook -i inventory.ini --check <playbook.yml>` (< 5 seconds, validation only - do not execute)

**NOTE**: Syntax checks and ansible-lint will fail on playbooks using community collections due to missing dependencies in sandbox environment.

### Validation Commands
**WARNING**: Use validation commands only - do not execute playbooks in sandbox environment.

For syntax validation:
```bash
export $(grep -v '^#' .env | xargs)
ansible-playbook -i inventory.ini --syntax-check <playbook.yml>
```

For dry-run validation:
```bash
export $(grep -v '^#' .env | xargs) 
ansible-playbook -i inventory.ini --check <playbook.yml>
```

**NOTE**: Replace `<playbook.yml>` with the actual playbook filename. Validation only - never execute playbooks without explicit approval.

### Modular Playbook Design
When adding new playbooks to the repository, follow these principles:
- **Modularity**: Design playbooks to run independently without requiring execution of other playbooks
- **Separation of concerns**: Each playbook should handle a specific component or service
- **Reusability**: Create playbooks that can be used in different deployment scenarios
- **Clear dependencies**: If dependencies exist, document them clearly in the playbook comments

Current playbook structure allows individual execution of components. Each playbook is self-contained and can be validated independently using the commands above.

## Database Replication
Located in `ansible/replicate_mariadb/` directory - separate functionality for migrating existing Matomo installations.

**WARNING**: Use validation commands only - do not execute replication playbooks without explicit approval.

For validation only:
```bash
cd ansible/replicate_mariadb/
ansible-playbook -i inventory.ini --syntax-check replicate_db.yml
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
├── .github/
│   └── copilot-instructions.md  # GitHub Copilot instructions
└── ansible/                    # Main Ansible directory
    ├── deploy-matomo/          # Main deployment playbooks
    │   ├── .env.example        # Environment template  
    │   ├── inventory.ini       # Server inventory
    │   ├── site.yml           # Main orchestration playbook
    │   ├── variables.yml      # Variable definitions
    │   ├── [individual playbooks] # Component-specific playbooks
    │   └── templates/         # Jinja2 configuration templates
    └── replicate_mariadb/     # Database replication playbooks
        ├── replicate_db.yml   # Main replication playbook
        └── [support files]    # Additional replication components
```

### Key Configuration Files
- **Main playbook**: `ansible/deploy-matomo/site.yml` 
- **Environment variables**: `ansible/deploy-matomo/.env.example` (copy to `.env`)
- **Server inventory**: `ansible/deploy-matomo/inventory.ini`
- **Variable definitions**: `ansible/deploy-matomo/variables.yml`
- **Configuration templates**: `ansible/deploy-matomo/templates/`

## Deploy Matomo Playbook Limitations and Requirements

### Sandbox Environment Limitations
- **Network restrictions**: Cannot download external packages (nginx source, MaxMind databases) or access ansible-galaxy
- **Missing collections**: Community collections are not accessible, causing syntax check failures on SSL and database playbooks
- **Service management**: Cannot start/stop system services like nginx, MariaDB
- **SSL certificate generation**: Cannot obtain real SSL certificates from Let's Encrypt
- **Validation only**: Use dry run mode (`--check`) for testing changes on individual playbooks only

### Build and Timing Expectations  
- **Syntax checks**: < 1 second per playbook
- **Linting**: < 30 seconds for full repository
- **Dry run validation**: < 30 seconds (safe in sandbox)
- **Full deployment**: 45-90 minutes (requires real servers, validation only in sandbox)

### Required External Dependencies
- MaxMind GeoIP2 account and license key (free account sufficient)
- Ubuntu 24.04 target servers with sudo access
- SSL certificate email address for Let's Encrypt

## Replicate MariaDB Playbook Limitations and Requirements

### Sandbox Environment Limitations
- **Database connectivity**: Cannot connect to actual MariaDB instances
- **SSH access**: Cannot establish SSH connections for replication setup
- **File transfers**: Cannot transfer database dumps between servers
- **Validation only**: Use syntax check and dry run modes only

### Build and Timing Expectations
- **Syntax checks**: < 1 second per playbook component
- **Linting**: < 10 seconds for replication directory
- **Dry run validation**: < 10 seconds (safe in sandbox)
- **Full replication**: 30-60 minutes (requires real servers, validation only in sandbox)

### Required External Dependencies
- Source MariaDB server with existing Matomo database
- Target MariaDB server for replication setup
- SSH access between servers
- Network connectivity between database servers

Always verify changes with syntax validation and linting before attempting any deployment or replication operations.