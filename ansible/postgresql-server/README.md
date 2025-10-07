# PostgreSQL Server Playbook

Production-ready PostgreSQL installation for dbname.

## Features

- ✅ PostgreSQL 16 installation
- ✅ Optimized configuration for production
- ✅ Automatic database and user creation
- ✅ Daily backups with retention policy
- ✅ Optional S3 backup upload
- ✅ Security hardening (scram-sha-256)
- ✅ ansible-vault support for secrets

## Quick Start

### 1. Setup inventory

Create `inventory/production.ini` with your server details:

```ini
# PostgreSQL Server Inventory
[postgresql_servers]
db-server ansible_host=192.168.3.100

[postgresql_servers:vars]
ansible_user=ops
ansible_password={{ vault_ansible_password }}
ansible_sudo_pass={{ vault_ansible_sudo_pass }}
```

**Options:**
- `ansible_host`: IP address or hostname of your PostgreSQL server
- `ansible_user`: SSH user for connecting to the server
- `ansible_password`: SSH password (from vault)
- `ansible_sudo_pass`: Sudo password (from vault)

**Alternative: Using SSH keys (recommended for production):**

```ini
[postgresql_servers]
db-server ansible_host=192.168.3.100

[postgresql_servers:vars]
ansible_user=ops
ansible_ssh_private_key_file=~/.ssh/id_rsa
ansible_become=yes
ansible_become_pass={{ vault_ansible_sudo_pass }}
```

### 2. Configure secrets

```bash
# Create directory for group_vars
mkdir -p group_vars/all

# Copy example vault file
cp group_vars/all/vault.yml.example group_vars/all/vault.yml

# Edit vault.yml with your passwords
nano group_vars/all/vault.yml

# Encrypt vault file
ansible-vault encrypt group_vars/all/vault.yml
```

**Example `group_vars/all/vault.yml` before encryption:**

```yaml
---
# Vault file for sensitive credentials
# Encrypt this file with: ansible-vault encrypt group_vars/all/vault.yml

# SSH/Sudo credentials
vault_ansible_password: "YourSecureSSHPassword123!"
vault_ansible_sudo_pass: "YourSecureSudoPassword456!"

# PostgreSQL credentials
vault_postgresql_dbname_password: "dbnameDBPassword789!@#$"

# Optional: S3 backup credentials (if using S3 backups)
# vault_aws_access_key_id: "AKIAIOSFODNN7EXAMPLE"
# vault_aws_secret_access_key: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
```

**Tips for secure passwords:**
- Use strong, unique passwords (minimum 16 characters)
- Include uppercase, lowercase, numbers, and special characters
- Generate passwords with: `openssl rand -base64 24`
- Never commit unencrypted vault files to git

**Verify encryption:**
```bash
# Check if vault file is encrypted
head -n 1 group_vars/all/vault.yml
# Should output: $ANSIBLE_VAULT;1.1;AES256...

# Edit encrypted vault file
ansible-vault edit group_vars/all/vault.yml

# View encrypted vault file
ansible-vault view group_vars/all/vault.yml
```

### 3. Customize configuration (optional)

Edit `roles/postgresql/defaults/main.yml` to adjust:
- PostgreSQL version
- Memory settings
- Backup settings
- S3 configuration

### 4. Run playbook

```bash
# Without vault
ansible-playbook -i inventory/production.ini site.yml

# With vault
ansible-playbook -i inventory/production.ini site.yml --ask-vault-pass
```

## Configuration

### Database Settings

Default configuration creates:
- Database: `dbname`
- User: `dbname`
- Encoding: UTF8

### Backup

**Local backups:**
- Location: `/var/backups/postgresql/`
- Schedule: Daily at 2:00 AM
- Retention: 7 days
- Cleanup: Daily at 3:30 AM

**S3 backups (optional):**

Enable in `roles/postgresql/defaults/main.yml`:

```yaml
postgresql_backup_s3_enabled: true
postgresql_backup_s3_bucket: "your-bucket-name"
postgresql_backup_s3_region: "us-east-1"
```

Ensure AWS credentials are configured on the server:
```bash
aws configure
```

### Memory Tuning

Adjust based on your server RAM in `roles/postgresql/defaults/main.yml`:

```yaml
# For 4GB RAM server
postgresql_shared_buffers: "1GB"
postgresql_effective_cache_size: "3GB"
postgresql_work_mem: "8MB"
postgresql_maintenance_work_mem: "256MB"

# For 8GB RAM server
postgresql_shared_buffers: "2GB"
postgresql_effective_cache_size: "6GB"
postgresql_work_mem: "16MB"
postgresql_maintenance_work_mem: "512MB"
```

## Manual Backup/Restore

### Create manual backup

```bash
sudo -u postgres pg_dump dbname | gzip > dbname_backup.sql.gz
```

### Restore from backup

```bash
gunzip -c dbname_backup.sql.gz | sudo -u postgres psql dbname
```

## Security

- Password encryption: scram-sha-256
- Listen address: localhost only (change if K3s needs access)
- All secrets stored in ansible-vault
- Regular automated backups

## Accessing from K3s

If dbname in K3s needs to access PostgreSQL on the host:

1. Update `pg_hba.conf` template to allow K3s pod network
2. Change `postgresql_listen_addresses` to include host IP
3. Update firewall rules to allow port 5432 from localhost

Example:
```yaml
postgresql_listen_addresses: "localhost,192.168.3.100"
```

## Monitoring

Check backup logs:
```bash
tail -f /var/log/postgresql-backup.log
```

Check PostgreSQL status:
```bash
systemctl status postgresql
sudo -u postgres psql -c "SELECT version();"
```

Check database size:
```bash
sudo -u postgres psql dbname -c "SELECT pg_size_pretty(pg_database_size('dbname'));"
```
