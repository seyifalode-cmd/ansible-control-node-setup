# **Ansible Control Node Setup**

Ansible playbooks executed from an EC2 control node to install and configure Apache (httpd) on web servers and MariaDB on database servers across a multi-node inventory — including database creation, user provisioning, and SQL data import.

---

## Project at a Glance

| | |
|---|---|
| **Tools Used** | Ansible, Apache httpd, MariaDB, Amazon Linux 2 |
| **Platform** | AWS EC2 (managed from Ansible control node) |
| **Languages** | YAML (Ansible), SQL |
| **What It Does** | Configures web servers with httpd and static content, configures database servers with MariaDB and seed data |

---

## The Problem This Project Solves

After an Ansible control node and a fleet of managed EC2 instances are provisioned, the next challenge is configuring those nodes consistently and correctly. Manually SSHing into each server to install packages, start services, create databases, and deploy content is time-consuming, inconsistent, and leaves configuration drift as an ongoing risk. A single missed step — failing to enable a service, misconfiguring database privileges, or deploying the wrong content — can break an application silently.

This project solves that problem by encoding the entire configuration of two distinct server roles — web and database — into Ansible playbooks. The `install_httpd.yaml` playbook installs Apache, copies a zip archive of static content to the remote server, extracts it, places the files in the web root, and starts the service with systemd persistence. The `install_mariadb.yaml` playbook installs MariaDB, starts and enables the service, installs the Python MySQL client library needed by Ansible's `mysql_db` and `mysql_user` modules, creates a database, provisions a user with full privileges, and imports a seed SQL file.

Together these playbooks represent idempotent, version-controlled configuration management for a two-tier application stack. They can be re-run safely and are driven entirely from the control node established in the companion provisioner project.

---

## Architecture

```
Ansible Control Node (EC2)
         |
         |  SSH (port 22, using id_rsa key pair)
         |
         +---------------------------+---------------------------+
         |                           |
         v                           v
+------------------+       +----------------------+
| [webservers]     |       | [databases]          |
| EC2 instances    |       | EC2 instances        |
|                  |       |                      |
| install_httpd.yaml:      | install_mariadb.yaml:|
| - yum: httpd     |       | - yum: mariadb-server|
| - copy: Einstein.zip     | - systemd: mariadb   |
| - unarchive      |       | - yum: python2-PyMySQL
| - copy to webroot|       | - mysql_db: bobdata  |
| - systemd: httpd |       | - mysql_user: Bob    |
+------------------+       | - copy: quotes.sql   |
                           | - shell: mysql import|
                           +----------------------+
```

---

## Repository Structure

```
ansible-control-node-setup/
├── install_httpd.yaml      # Playbook: install Apache and deploy static content
├── install_mariadb.yaml    # Playbook: install MariaDB, create DB/user, import data
├── quotes.sql              # Seed data: SteveJobsQuotes table (20 rows)
├── Einstein.zip            # Static web content (index.html + einstein.jpg)
└── .gitignore
```

---

## How It Works

### Web Server Configuration (`install_httpd.yaml`)

The playbook targets hosts in the `[webservers]` group defined in the control node's inventory file. It runs as `ec2-user` with privilege escalation (`become: true`).

1. **Package installation** — Installs `httpd` via `yum`.
2. **Content transfer** — Copies `Einstein.zip` from the control node to the remote server's home directory.
3. **Archive extraction** — Uses the `unarchive` module with `remote_src: yes` to extract the zip in `/tmp`.
4. **Web root deployment** — Loops over `index.html` and `einstein.jpg`, copying each from `/tmp` to `/var/www/html/` with `ec2-user` as the owner.
5. **Service management** — Starts `httpd` and enables it on boot via the `systemd` module.

### Database Server Configuration (`install_mariadb.yaml`)

The playbook targets the `[databases]` group and also runs with `become: true`.

1. **Package installation** — Installs `mariadb-server` via `yum`.
2. **Service management** — Starts and enables the `mariadb` service via `systemd`.
3. **EPEL + Python client** — Installs the EPEL repository (for access to EPEL-only packages) and then installs `python2-PyMySQL`, the library required for Ansible's `mysql_db` and `mysql_user` modules to communicate with MariaDB.
4. **Database creation** — Creates a database named `bobdata`.
5. **User provisioning** — Creates a MySQL user `Bob` with full privileges across all databases.
6. **Data import** — Copies `quotes.sql` to the remote host and runs it via a shell task (`mysql -u root bobdata < quotes.sql`), creating the `SteveJobsQuotes` table and inserting 20 rows of seed data.

---

## Walkthrough

The following commands are run from the Ansible control node after the managed nodes are provisioned and the inventory file is populated with their public IPs.

```bash
# Verify connectivity to all managed nodes
ansible all -i host.ini -m ping

# Run the web server playbook against webservers group
ansible-playbook -i host.ini install_httpd.yaml

# Run the database playbook against databases group
ansible-playbook -i host.ini install_mariadb.yaml

# Verify httpd is running on a web node
curl http://<webserver-public-ip>

# Verify MariaDB is running on a database node
ssh -i ~/.ssh/id_rsa ec2-user@<db-node-public-ip>
mysql -u Bob -ppa55word bobdata -e "SELECT * FROM SteveJobsQuotes LIMIT 5;"
```

---

## How to Reproduce

**Prerequisites**

- Ansible control node provisioned (see `ansible-control-node-provisioner`)
- Managed nodes running (see `ansible-managed-nodes-provisioner`)
- SSH keys from the control node authorized on all managed nodes
- Inventory file (`host.ini`) populated with managed node IPs under `[webservers]` and `[databases]` groups

```bash
# From the Ansible control node, clone this repository
git clone https://github.com/Seyifunmi0604/ansible-control-node-setup.git
cd ansible-control-node-setup

# Run the web server setup
ansible-playbook -i host.ini install_httpd.yaml

# Run the database setup
ansible-playbook -i host.ini install_mariadb.yaml
```

**Sample inventory (`host.ini`) structure:**

```ini
[webservers]
<webserver-1-public-ip>
<webserver-2-public-ip>

[databases]
<db-server-1-public-ip>
<db-server-2-public-ip>
```

---

## Seed Data

The `quotes.sql` file creates a `SteveJobsQuotes` table and populates it with 20 quotations. This is used to validate end-to-end database provisioning — confirming that the Ansible `mysql_db`, `mysql_user`, and shell import tasks all completed successfully.

---

## Related Projects

- `ansible-sandbox-ec2` — Minimal single-node Ansible sandbox
- `ansible-control-node-provisioner` — Provisions the control node used to run these playbooks
- `ansible-managed-nodes-provisioner` — Provisions the target fleet this project configures
- `ansible-control-node-setup` — **This project** — Configuration playbooks for web and database roles
- `mariadb-ansible-setup` — Self-contained single-node MariaDB provisioning

---

*Oluwaseyi Michael Falode · Cybersecurity & Cloud Security Engineer · Toronto, ON*
