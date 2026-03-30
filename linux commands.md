PHASE 1 — Enable WSL2 on Windows (One-Time)
STEP 1: Open PowerShell as Administrator
Press Windows key
Type PowerShell
Right-click → Run as Administrator

STEP 2: Enable WSL
Run this command:

wsl --install

PHASE 2 — Install PostgreSQL (Basic Setup)

PHASE 3 — Create Encrypted Storage (THIS IS TDE)
This is the core part.

STEP 9 — Create a virtual disk file (simulating real disk)
In Ubuntu:
dd if=/dev/zero of=pgdisk.img bs=1M count=2048
what this means:
Create a 2GB disk file
This acts like a real hard disk

STEP 10 — Attach the disk file as a block device
sudo losetup -fP pgdisk.img
Find the device name:
losetup -a
You’ll see something like:
/dev/loop0
(Remember this name)

STEP 11 — Encrypt the disk using LUKS
This is where dm-crypt + LUKS come into play.
sudo cryptsetup luksFormat /dev/loop0
Type YES
Set a password (IMPORTANT)
What just happened:
Disk is now encrypted
Only unlockable with password

STEP 12 — Unlock the encrypted disk
sudo cryptsetup open /dev/loop0 pgcrypt
Now Linux creates:
/dev/mapper/pgcrypt
This is the decrypted view (in memory only)

STEP 13 — Create filesystem on encrypted disk
sudo mkfs.ext4 /dev/mapper/pgcrypt

STEP 14 — Mount encrypted disk
sudo mkdir /mnt/pg_encrypted
sudo mount /dev/mapper/pgcrypt /mnt/pg_encrypted
Everything inside /mnt/pg_encrypted is now encrypted at rest

PHASE 4 — Move PostgreSQL Data to Encrypted Disk
STEP 15 — Change ownership
sudo chown -R postgres:postgres /mnt/pg_encrypted

STEP 16 — Initialize PostgreSQL data directory on encrypted disk
sudo -u postgres initdb -D /mnt/pg_encrypted

STEP 17 — Tell PostgreSQL to use this directory
Edit config:
sudo nano /etc/postgresql/*/main/postgresql.conf
Find:
data_directory =
Change to:
data_directory = '/mnt/pg_encrypted'
Save:
CTRL + O
ENTER
CTRL + X

STEP 18 — Start PostgreSQL
sudo systemctl start postgresql

VERIFY TDE IS WORKING
Enter Postgres:
sudo -u postgres psql
Create test:
CREATE TABLE test(secret text);
INSERT INTO test VALUES ('ENCRYPTED DATA');
\q

Now check raw disk:
sudo strings pgdisk.img

You should NOT see:
Table name
Data text
That means encryption is working.


HERE ENDS THE ENCRYPTION PART NOW WE HAVE TDE-LIKE SETUP--


PHASE 2 — Auto Unlock at Boot (Oracle-like Behavior)

Oracle loads wallet automatically.

We simulate that.
STEP 1 — Create Secure Key File
sudo dd if=/dev/urandom of=/root/pg.key bs=4096 count=1
sudo chmod 0400 /root/pg.key

STEP 2 — Add Key to LUKS
sudo cryptsetup luksAddKey /dev/loop14 /root/pg.key
Enter your existing LUKS password.
Now disk can unlock via key file.

STEP 3 — Configure Auto Unlock
Edit crypttab:
sudo nano /etc/crypttab
Add this line:
pgcrypt /dev/loop14 /root/pg.key luks
Save.
Edit fstab:
sudo nano /etc/fstab
Add:
/dev/mapper/pgcrypt /mnt/pg_secure ext4 defaults 0 2
Save.
Now test:
sudo reboot
After reboot:
mount | grep pgcrypt
If mounted automatically → success 🎉

🟢 PHASE 3 — Separate Key Storage Using Vault

Now we move toward Oracle Wallet style.
STEP 1 — Install HashiCorp Vault
sudo apt install unzip -y
wget https://releases.hashicorp.com/vault/1.15.2/vault_1.15.2_linux_amd64.zip
unzip vault_1.15.2_linux_amd64.zip
sudo mv vault /usr/local/bin/
Check:
vault --version

STEP 2 — Start Dev Vault (Learning Mode)
vault server -dev
It will print:
Root Token: XXXXX
Copy this token.
Open new terminal:
export VAULT_ADDR='http://127.0.0.1:8200'
vault login <paste-root-token>

STEP 3 — Store Disk Key in Vault
Store your key file:
vault kv put secret/pgkey key=@/root/pg.key
Now key is inside Vault.
Enterprise style achieved.
Later you can:
Fetch key dynamically
Rotate centrally
Audit access

🟢 PHASE 4 — Encrypt Backups (Oracle RMAN Equivalent)
Logical Backup Encryption
sudo -u postgres pg_dump postgres | gpg -c > backup.sql.gpg
It will ask for password.
That file is encrypted.
Physical Backup
Since your disk is encrypted:
Base backups are encrypted automatically.
Extra layer:
tar czf - /mnt/pg_secure | gpg -c > basebackup.tar.gpg

🟢 PHASE 5 — Tablespace-Level Encryption
Create new encrypted mount (optional advanced).
For now use existing encrypted mount.
Inside psql:
sudo -u postgres psql
CREATE TABLESPACE secure_ts LOCATION '/mnt/pg_secure/secure_ts';
Now create table:
CREATE TABLE sensitive_data (
  id serial,
  secret text
) TABLESPACE secure_ts;
Now only sensitive tables stored in encrypted tablespace.
Oracle-like behavior.

🟢 PHASE 6 — Enable SSL (Encryption in Transit)

STEP 1 — Generate Self-Signed Certificate
sudo mkdir /etc/postgresql/16/main/ssl
cd /etc/postgresql/16/main/ssl
sudo openssl req -new -x509 -days 365 -nodes -text \
-out server.crt \
-keyout server.key
Secure permissions:
sudo chmod 600 server.key
sudo chown postgres:postgres server.*

STEP 2 — Enable SSL in Config
Edit:
sudo nano /etc/postgresql/16/main/postgresql.conf
Set:
ssl = on
ssl_cert_file = 'ssl/server.crt'
ssl_key_file = 'ssl/server.key'
Restart:
sudo systemctl restart postgresql
Now traffic encrypted.

🟢 PHASE 7 — Key Rotation (Oracle Master Key Rotation Equivalent)
Add new key:
sudo cryptsetup luksAddKey /dev/loop14
Remove old key:
sudo cryptsetup luksRemoveKey /dev/loop14
This rotates encryption keys without re-encrypting disk.
Enterprise-level feature.

🔥 What You we Have


PostgreSQL
   ↓
Encrypted Tablespaces
   ↓
dm-crypt (AES-256)
   ↓
LUKS
   ↓
Vault-managed key
   ↓
Encrypted backups
   ↓
SSL network encryption

This is Oracle TDE equivalent architecture.
