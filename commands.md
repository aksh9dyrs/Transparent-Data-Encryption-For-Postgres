# PostgreSQL TDE Implementation - Complete Command Reference

---

## PHASE 1: Enable WSL2 on Windows (One-Time Setup)

### Overview
WSL2 provides a Linux kernel on Windows, allowing us to run encrypted Linux volumes and PostgreSQL.

### Manual Steps
1. Press Windows key → Type `PowerShell` → Right-click → Run as Administrator

### Command
```bash
wsl --install
```

**What happens:** Installs Windows Subsystem for Linux 2 with Ubuntu distribution.

---

## PHASE 3: Create Encrypted Storage (Core TDE Implementation)

### Step 1: Create Virtual Disk File

**Information:**
- Simulates a real physical hard disk
- Size: 2GB (2048 MB)
- This acts as the storage that will be encrypted

### Command
```bash
dd if=/dev/zero of=pgdisk.img bs=1M count=2048
```

**What this does:** Creates a 2GB disk file named `pgdisk.img`

---

### Step 2: Attach Disk File as Block Device

**Information:**
- Converts the disk image file into a usable block device
- The system will assign a name like `/dev/loop0`

### Commands
```bash
sudo losetup -fP pgdisk.img
```

**To find the device name (remember this):**
```bash
losetup -a
```

**Expected output:** `/dev/loop0` (or similar)

---

### Step 3: Encrypt the Disk Using LUKS

**Information:**
- LUKS (Linux Unified Key Setup) encrypts the disk
- dm-crypt provides the encryption layer
- Only decryptable with your password

### Command
```bash
sudo cryptsetup luksFormat /dev/loop0
```

**Prompts:**
- Type `YES` (confirm you want to format)
- Enter your LUKS password (IMPORTANT - save this!)

**What happens:** Disk is now encrypted with AES-256

---

### Step 4: Unlock the Encrypted Disk

**Information:**
- Opens the encrypted disk and creates a decrypted mapping
- The mapping exists in memory only (still encrypted on disk)

### Command
```bash
sudo cryptsetup open /dev/loop0 pgcrypt
```

**What happens:** Creates `/dev/mapper/pgcrypt` (decrypted view of the disk)

---

### Step 5: Create Filesystem on Encrypted Disk

**Information:**
- Formats the encrypted disk with ext4 filesystem
- Necessary before PostgreSQL can use it

### Command
```bash
sudo mkfs.ext4 /dev/mapper/pgcrypt
```

**What happens:** Initializes ext4 filesystem on the encrypted volume

---

### Step 6: Mount Encrypted Disk

**Information:**
- Makes the encrypted volume accessible as a directory
- All data written here is encrypted automatically

### Commands
```bash
sudo mkdir /mnt/pg_encrypted
```

```bash
sudo mount /dev/mapper/pgcrypt /mnt/pg_encrypted
```

**What happens:** Mounted encrypted volume is now ready for PostgreSQL

---

## PHASE 4: Move PostgreSQL Data to Encrypted Disk

### Step 1: Change Ownership

**Information:**
- PostgreSQL runs as `postgres` user
- Must own the data directory to write data

### Command
```bash
sudo chown -R postgres:postgres /mnt/pg_encrypted
```

**What happens:** PostgreSQL user can now access the encrypted directory

---

### Step 2: Initialize PostgreSQL Data Directory

**Information:**
- Creates PostgreSQL cluster on the encrypted volume
- This becomes the primary data storage location

### Command
```bash
sudo -u postgres initdb -D /mnt/pg_encrypted
```

**What happens:** PostgreSQL cluster initialized on encrypted disk

---

### Step 3: Configure PostgreSQL to Use Encrypted Path

**Information:**
- PostgreSQL config file must point to encrypted directory
- Ensures all data goes to encrypted storage

### Command to edit config
```bash
sudo nano /etc/postgresql/*/main/postgresql.conf
```

**Find this line:**
```
data_directory = '/var/lib/postgresql/16/main'
```

**Change to:**
```
data_directory = '/mnt/pg_encrypted'
```

**Save and exit:**
- `CTRL + O` → `ENTER` → `CTRL + X`

---

### Step 4: Start PostgreSQL

**Information:**
- Starts the PostgreSQL service using the encrypted data directory

### Command
```bash
sudo systemctl start postgresql
```

**What happens:** PostgreSQL now runs on encrypted storage

---

## VERIFY TDE IS WORKING

### Connect to PostgreSQL
```bash
sudo -u postgres psql
```

### Create Test Data
```sql
CREATE TABLE test(secret text);
INSERT INTO test VALUES ('ENCRYPTED DATA');
\q
```

### Verify Encryption

**Check raw disk (should NOT reveal data):**
```bash
sudo strings pgdisk.img
```

**Expected result:** 
- You should NOT see "ENCRYPTED DATA"
- You should NOT see table names
- Only encrypted binary output visible

**If you don't see the data → Encryption is working! ✓**

---

## PHASE 5: Auto Unlock at Boot (Oracle-like Behavior)

### Overview
Enables automatic decryption at boot, similar to Oracle wallet auto-opening.

### Step 1: Create Secure Key File

**Information:**
- Generates random encryption key from system entropy
- Key file is restricted to root only (0400 permissions)

### Commands
```bash
sudo dd if=/dev/urandom of=/root/pg.key bs=4096 count=1
```

```bash
sudo chmod 0400 /root/pg.key
```

**What happens:** Creates 4KB secure key file

---

### Step 2: Add Key to LUKS

**Information:**
- LUKS supports multiple keys (key slots)
- This adds our key file as an alternative unlock method
- Original password still works

### Command
```bash
sudo cryptsetup luksAddKey /dev/loop0 /root/pg.key
```

**Prompt:** Enter your existing LUKS password
**What happens:** Disk can now unlock via key file OR password

---

### Step 3: Configure Automatic Unlock at Boot

**Information:**
- `/etc/crypttab` tells system how to automatically unlock volumes
- `/etc/fstab` tells system where to mount volumes

### Edit crypttab
```bash
sudo nano /etc/crypttab
```

**Add this line:**
```
pgcrypt /dev/loop0 /root/pg.key luks
```

### Edit fstab
```bash
sudo nano /etc/fstab
```

**Add this line:**
```
/dev/mapper/pgcrypt /mnt/pg_encrypted ext4 defaults 0 2
```

### Test Automatic Mounting
```bash
sudo reboot
```

**After reboot, verify:**
```bash
mount | grep pgcrypt
```

**Expected output:** Mount point should be listed
**If listed → Success! ✓ Auto-unlock is working**

---

## PHASE 6: Separate Key Storage Using HashiCorp Vault

### Overview
Moves encryption keys to centralized Vault instead of local filesystem (Oracle Wallet equivalent).

### Step 1: Install HashiCorp Vault

**Information:**
- Vault is enterprise key management tool
- Community version is free and open-source

### Commands
```bash
sudo apt install unzip -y
```

```bash
wget https://releases.hashicorp.com/vault/1.15.2/vault_1.15.2_linux_amd64.zip
```

```bash
unzip vault_1.15.2_linux_amd64.zip
```

```bash
sudo mv vault /usr/local/bin/
```

**Verify installation:**
```bash
vault --version
```

---

### Step 2: Start Vault in Development Mode

**Information:**
- Dev mode for learning/testing only
- Production uses persistent storage backend

### Command (Terminal 1)
```bash
vault server -dev
```

**Output includes:**
```
Root Token: XXXXX (copy this)
Unseal Key: YYYYY
```

**Keep this terminal running!**

---

### Step 3: Login to Vault (Terminal 2)

**Information:**
- Terminal 2 for Vault CLI commands
- Use root token from Terminal 1

### Commands
```bash
export VAULT_ADDR='http://127.0.0.1:8200'
```

```bash
vault login <paste-root-token>
```

**What happens:** Authenticated to Vault

---

### Step 4: Store Encryption Key in Vault

**Information:**
- Moves key from filesystem to Vault
- Keys now centrally managed and audited

### Command
```bash
vault kv put secret/pgkey key=@/root/pg.key
```

**What happens:** 
- Key now stored in Vault
- Enterprise-style key management achieved
- Can now rotate, audit, and manage centrally

---

## PHASE 7: Encrypt Backups

### Overview
Ensures backups are encrypted, preventing backup files from exposing data.

### Logical Backup Encryption (SQL dumps)

**Information:**
- Uses GPG for symmetric encryption
- Encrypted backup can be transmitted safely

### Command
```bash
sudo -u postgres pg_dump postgres | gpg -c > backup.sql.gpg
```

**Prompt:** Enter GPG encryption password
**Output:** `backup.sql.gpg` (encrypted backup file)

---

### Physical Backup Encryption (Base backups)

**Information:**
- Since disk is already encrypted, backups inherit encryption
- Additional GPG layer for maximum protection

### Command
```bash
tar czf - /mnt/pg_encrypted | gpg -c > basebackup.tar.gpg
```

**Prompt:** Enter GPG encryption password
**Output:** `basebackup.tar.gpg` (encrypted base backup)

---

## PHASE 8: Tablespace-Level Encryption

### Overview
Create separate encrypted tablespaces for different data classification levels.

### Step 1: Create Encrypted Tablespace

**Information:**
- Sensitive data can use encrypted tablespaces
- Non-sensitive data can use standard tablespaces
- Provides granular control

### Command (in PostgreSQL)
```bash
sudo -u postgres psql
```

**In psql, execute:**
```sql
CREATE TABLESPACE secure_ts LOCATION '/mnt/pg_encrypted/secure_ts';
```

**Exit psql:**
```
\q
```

---

### Step 2: Create Table on Encrypted Tablespace

**Information:**
- Only this table and its indexes use encrypted tablespace
- Other tables use default storage

### Command (in PostgreSQL)
```bash
sudo -u postgres psql
```

**Create directory first:**
```bash
sudo mkdir -p /mnt/pg_encrypted/secure_ts
sudo chown postgres:postgres /mnt/pg_encrypted/secure_ts
```

**In psql, create table:**
```sql
CREATE TABLE sensitive_data (
  id serial,
  secret text
) TABLESPACE secure_ts;
```

**Exit:**
```
\q
```

**What happens:** Only this sensitive table is encrypted with explicit tablespace

---

## PHASE 9: Enable SSL/TLS (Encryption in Transit)

### Overview
Encrypts database connections to protect data traveling between clients and server.

### Step 1: Generate Self-Signed Certificate

**Information:**
- Creates certificate valid for 365 days
- Self-signed for development (production uses CA-signed)

### Commands
```bash
sudo mkdir -p /etc/postgresql/16/main/ssl
```

```bash
cd /etc/postgresql/16/main/ssl
```

```bash
sudo openssl req -new -x509 -days 365 -nodes -text \
-out server.crt \
-keyout server.key
```

**Prompts:** Answer certificate questions (all optional for self-signed)

**Secure permissions:**
```bash
sudo chmod 600 server.key
```

```bash
sudo chown postgres:postgres server.*
```

---

### Step 2: Enable SSL in PostgreSQL Config

**Information:**
- Configures PostgreSQL to require SSL connections
- All client-server communication encrypted

### Command
```bash
sudo nano /etc/postgresql/16/main/postgresql.conf
```

**Find and set these lines:**
```
ssl = on
ssl_cert_file = 'ssl/server.crt'
ssl_key_file = 'ssl/server.key'
```

**Save:** `CTRL + O` → `ENTER` → `CTRL + X`

**Restart PostgreSQL:**
```bash
sudo systemctl restart postgresql
```

**What happens:** SSL/TLS encryption now active for all connections

---

## PHASE 10: Key Rotation (Master Key Rotation)

### Overview
Rotate encryption keys without re-encrypting the entire disk.

### Step 1: Add New Key

**Information:**
- Creates new key slot in LUKS
- Old key still works during transition

### Command
```bash
sudo cryptsetup luksAddKey /dev/loop0
```

**Prompts:**
- Enter existing LUKS password
- Enter new LUKS password for new key

**What happens:** New encryption key added to key slots

---

### Step 2: Remove Old Key

**Information:**
- After ensuring new key is working, remove old key
- Old key slot is destroyed

### Command
```bash
sudo cryptsetup luksRemoveKey /dev/loop0
```

**Prompt:** Enter the password of key slot to remove

**What happens:** Old key slot removed, disk encrypted with new key only

---

## Final Architecture

Your complete TDE stack:

```
PostgreSQL Server
        ↓
Encrypted Tablespaces
        ↓
dm-crypt + LUKS (AES-256)
        ↓
Block Device Encryption
        ↓
HashiCorp Vault (Key Management)
        ↓
GPG-Encrypted Backups
        ↓
SSL/TLS Encrypted Connections
        ↓
LUKS Key Rotation Support
```

This is a complete **PostgreSQL TDE equivalent** to Oracle Database encryption.

---

## Summary

| Phase | Component | Command | Purpose |
|-------|-----------|---------|---------|
| 1 | WSL2 | `wsl --install` | Linux environment |
| 3 | Storage | `dd if=/dev/zero...` | Create encrypted disk |
| 3 | Encryption | `cryptsetup luksFormat` | Enable LUKS encryption |
| 3 | Mounting | `mount /dev/mapper...` | Mount encrypted volume |
| 4 | PostgreSQL | `initdb -D /mnt/...` | Initialize on encrypted disk |
| 4 | Config | Edit `postgresql.conf` | Point to encrypted path |
| 5 | Auto-unlock | `crypttab + fstab` | Boot-time decryption |
| 6 | Vault | `vault kv put` | Centralized key storage |
| 7 | Backups | `pg_dump \| gpg` | Encrypted backups |
| 8 | Tablespaces | `CREATE TABLESPACE` | Granular encryption |
| 9 | SSL | `openssl req` + config | Connection encryption |
| 10 | Rotation | `cryptsetup luksAddKey` | Key rotation |

