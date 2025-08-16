**Deploy Matomo with Ansible**

This is a simple Ansible playbook to deploy **Matomo** and **phpMyAdmin** on a server running Ubuntu, without using Docker.

A typical use case is where you got yourself a virtual server, and you want to run Matomo and phpMyAdmin on it, 
with some basic security measures in place. 

It is intended for lightweight cases where you do not need much large-scale automation, like server farms or what not - 
just a simple "install once" routine where you do not want to manually install and configure many services to run together.

This is an install from scratch, resulting with an empty Matomo - which you then need to configure via the web interface.

If you want to migrate an existing Matomo installation, you can run this playbook and then the [Replicate MariaDB playbook](replicate_db/replicate_db.md) on top.

Prerequisites:
 - Ansible installed on the machine where this will run from (tested with version 2.16.3.)
 - A server running Ubuntu (tested on version 24.04)
 - SSH access to the server with a user that has sudo privileges
 - MaxMind account and licence key to download the GeoIP2 database (free account is sufficient)

You need to have Ansible installed either on your server (if you are going to run the playbook from it as "local") 
or on your own machine if you will run the playbook from it.

For the cryptography needed to install SSL, Ansible also needs a "collection" for it:
```bash
apt install -y ansible
ansible-galaxy collection install community.crypto
apt install -y git
git clone https://github.com/pavka14/deploy-matomo.git
```
What the playbook will do (in order of appearance):
 - install required packages (prerequisites)
 - install some extras that are not strictly speaking needed, but are nice to have (Fail2ban)
 - install plain nginx server
 - install certbot which will then acquire and configure an SSL certificate (this requires Nginx to be in place already)
 - rebuild nginx from source, this time with the GeoIP2 module (which does not come as a package)
 - install PHP and required PHP extensions (needed to run both Matomo and phpMyAdmin)
 - pull the GeoIP2 database from MaxMind, configure it to update regularly (you need a MaxMind account and licence key for this; can be a free account)
 - install and configure MariaDB (with some "hardening")
 - install PHP
 - download and install Matomo, install phpMyAdmin (which comes as a package)
 - save configurations, which includes:
   - add username and password to be used for basic authentication to the website (this is NOT a Linux user)
   - allow the nginx user to read the phpMyAdmin code
   - create nginx server configuration from Jinja templates

What the end result should be:
 - a working Matomo installation, accessible via the domain or subdomain you set in the .env file, with configured SSL
 - a working phpMyAdmin installation, accessible via the above domain or subdomain plus a "/pma/" path
 - basic authentication for both Matomo and phpMyAdmin (excluding the javascript tracking code for Matomo and the public tracking endpoint)
 - GeoIP2 database installed and configured to update regularly, and usable by nginx
 - fail2ban configured to protect the server from (some) brute-force attacks

For deployment:
```bash
cp .env.example .env
nano .env
```
Add your settings, Ctrl+X to exit, confirm "Save changes" with "Y".

# Then export the variables into your shell:
```bash
export $(grep -v '^#' .env | xargs)
```

Edit the inventory file to match your server's (or servers') IP address or addresses, and define your user.

Then run the playbook on the target server(s) as defined in the inventory file:
```bash
ansible-playbook -i inventory.ini site.yml -e target_group=matomo_servers
```

Or, if you want to run the playbook on the local machine (i.e. the target is localhost):
```bash
ansible-playbook -i inventory.ini site.yml -e target_group=local
```

There is no default value for the `target_group` variable and this is deliberate! Errors in both directions can be costly.

You can run the whole playbook as above, or each of the sub-books individually (change site.yml in the command above to the name of the sub-book you want to run).

They should be independent; of course, you need to have run prerequisites.yml first.

The exceptions are nginx_reinstall.yml and ssl.yml, both of which need nginx to be present already (ssl.yml can work with both the initial vanilla Nginx and the re-installed one).
