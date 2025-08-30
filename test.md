Below is the exact text you can copy‑and‑paste into a file named backup.md.
If you paste it into any plain‑text editor (VS Code, Sublime, Notepad++, etc.) and then save the file, the formatting will be preserved exactly as shown.

backup.md

# Mail-in-a-Box Remote Backup Guide

A step‑by‑step recipe for creating a backup of a Mail‑in‑a‑Box server and pushing it to a remote location (another VPS, cloud storage, or SFTP/rsync target).

---

## 1. Overview of the Built‑in Backup

Mail‑in‑a‑Box already creates a compressed tarball of everything you need:
/home/user-data/backup/ backup-YYYY-MM-DD-HHMMSS.tar.gz


The archive includes:

- All user mailboxes (`/home/user-data/mail`)
- DNS zone files
- MySQL databases (mail, roundcube, …)
- Configuration files (`/etc`, `/var/lib/tor`, etc.)
- SSL certificates, keys, and other secrets

Running the built‑in script (`/usr/local/bin/mailinabox-backup`) is the safest way to capture a consistent snapshot.

---

## 2. Prerequisites on the Mail‑in‑a‑Box Host

1. **Root / sudo access** – the backup script needs to read system files.  
2. **Network connectivity** to the remote destination (port 22 for SSH/SFTP, or HTTPS for cloud APIs).  
3. **Authentication method** – preferably SSH key‑based auth for automated runs.

> **Tip:** Create a dedicated “backup” user on the remote host with a restricted home directory and only the permissions needed to receive the archive.

```bash
# On the remote host
adduser --disabled-password --gecos "" backup_user
mkdir -p /srv/mailinabox_backups
chown backup_user:backup_user /srv/mailinabox_backups
Copy the public key from the Mail‑in‑a‑Box server to ~backup_user/.ssh/authorized_keys.

3. Install the Remote‑Push Helper (Optional but Recommended)
While the native script stores the backup locally, we’ll add a thin wrapper that:

Runs the native backup.
Transfers the newest tarball to the remote host via rsync or scp.
Optionally removes old local backups to save space.
Create a script, e.g. /usr/local/bin/miab-backup-and-sync.sh:

#!/bin/bash
set -euo pipefail

# 1️⃣ Run the built‑in backup
/usr/local/bin/mailinabox-backup

# 2️⃣ Locate the most recent backup file
BACKUP_DIR="/home/user-data/backup"
LATEST=$(ls -t "${BACKUP_DIR}"/backup-*.tar.gz | head -n1)

# 3️⃣ Remote destination (adjust as needed)
REMOTE_USER="backup_user"
REMOTE_HOST="remote.example.com"
REMOTE_PATH="/srv/mailinabox_backups/"

# 4️⃣ Transfer via rsync (preserves timestamps, compresses on‑the‑fly)
rsync -avz --progress "${LATEST}" "${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_PATH}"

# 5️⃣ Optional: prune local backups older than N days (e.g., 7)
find "${BACKUP_DIR}" -type f -name 'backup-*.tar.gz' -mtime +7 -delete

echo "✅ Backup ${LATEST} sent to ${REMOTE_HOST}"
Make it executable:

chmod +x /usr/local/bin/miab-backup-and-sync.sh
4. Automate with Cron
Schedule the wrapper to run at a convenient interval (daily at 02:00 am is common):

# Edit root's crontab
sudo crontab -e
Add the line:

0 2 * * * /usr/local/bin/miab-backup-and-sync.sh >> /var/log/miab-backup.log 2>&1
The log file helps you verify success/failure.

5. Alternative Remote Targets
a) Cloud Storage (AWS S3, Google Cloud Storage, Backblaze B2)
Install the appropriate CLI (awscli, gsutil, or b2), then replace the rsync line with a single upload command, e.g.:

# AWS S3 example
aws s3 cp "${LATEST}" "s3://my-mailinabox-backups/${HOSTNAME}/"
Make sure the IAM credentials have write permission only to the designated bucket/prefix.

b) SFTP (different port or chroot)
If the remote service runs on a non‑standard port or inside a chroot, adjust the rsync/scp command:

rsync -avz -e 'ssh -p 2222' "${LATEST}" "${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_PATH}"
6. Verifying the Backup
After the first run:

List remote files – ssh backup_user@remote.example.com "ls -lh /srv/mailinabox_backups"
Test restore on a test box – download the tarball, extract it, and point Mail‑in‑a‑Box services to the restored data (or run the official restore script: /usr/local/bin/mailinabox-restore <archive>).
7. Security Considerations
Concern	Mitigation
Transport encryption	Use SSH/rsync over SSH or TLS‑protected cloud APIs.
Key compromise	Store the private key with chmod 600; consider a passphrase‑protected key and ssh-agent.
Retention	Keep only a limited number of remote snapshots (e.g., 30 days) and rotate them with a lifecycle policy.
Access control	The remote “backup_user” should have no sudo rights and a locked shell (/usr/sbin/nologin).
Integrity verification	Compare SHA‑256 hashes locally and remotely after transfer.
8. Quick Checklist
 Create a dedicated remote user with a locked shell.
 Copy the public SSH key from the Mail‑in‑a‑Box host.
 Deploy the wrapper script (miab-backup-and-sync.sh).
 Add a cron job (e.g., daily at 02:00).
 Verify the first remote copy and test a restore.
 Set up retention policies on both ends.
TL;DR Command Summary
# 1. Run native backup
/usr/local/bin/mailinabox-backup

# 2. Find newest archive
LATEST=$(ls -t /home/user-data/backup/backup-*.tar.gz | head -n1)

# 3. Push to remote (replace values)
rsync -avz "$LATEST" backup_user@remote.example.com:/srv/mailinabox_backups/

# 4. (Optional) Delete local backups >7 days
find /home/user-data/backup -name 'backup-*.tar.gz' -mtime +7 -delete
Place these lines into a script, schedule it, and you’ll have reliable, off‑site backups of your Mail‑in‑a‑Box installation.

Feel free to open an issue or PR if you spot anything that could be improved!


---  

**How to get the file onto your computer**

1. Open any text editor (e.g., VS Code, Notepad++, Sublime Text, or even the built‑in Notepad).  
2. Paste the entire block above (starting with `# Mail-in-a-Box Remote Backup Guide`).  
3. Save the file as **`backup.md`** (make sure the extension is `.md`).  

You now have a perfectly formatted Markdown file ready to add to your GitHub repository. Enjoy!