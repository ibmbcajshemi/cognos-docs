# 🧭 IBM Cognos Analytics 12.1.0 — GUI Installation on AWS EC2 (PostgreSQL or Aurora PostgreSQL Content Store)

## 1️⃣ Install Required Packages and Configure Firewall
```bash
sudo dnf groupinstall -y "Server with GUI"
sudo dnf install -y tigervnc-server perl xorg-x11-xauth xorg-x11-fonts-Type1 xorg-x11-fonts-misc firewalld libnsl
sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --add-port=5901/tcp
sudo firewall-cmd --permanent --add-port=9300/tcp
sudo firewall-cmd --reload
```

---

## 2️⃣ Create Cognos User and Grant Sudo Privileges
```bash
sudo useradd -m -s /bin/bash cognos
sudo passwd cognos
sudo usermod -aG wheel cognos
```

Verify:
```bash
groups cognos
```
✅ Output should include `wheel`.

---

## 3️⃣ Create VNC Password (Non-Interactive)
```bash
sudo -iu cognos bash -lc 'echo ChangeMeSecure! | vncpasswd -f > ~/.vnc/passwd && chmod 600 ~/.vnc/passwd'
```

---

## 4️⃣ Configure and Start VNC Server
Define the session, map display :1 to the Cognos user, and start the VNC service.

```bash
echo ":1=cognos" | sudo tee /etc/tigervnc/vncserver.users

sudo tee /etc/tigervnc/vncserver-config-defaults >/dev/null <<'EOF'
session=gnome
securitytypes=VncAuth
geometry=1600x900
localhost=0
EOF

sudo tee /etc/systemd/system/vncserver@:1.service >/dev/null <<'EOF'
[Unit]
Description=Remote desktop service (VNC)
After=syslog.target network.target

[Service]
Type=forking
ExecStartPre=+/usr/libexec/vncsession-restore %i
ExecStart=/usr/libexec/vncsession-start %i
PIDFile=/run/vncsession-%i.pid
SELinuxContext=system_u:system_r:vnc_session_t:s0

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable vncserver@:1.service
sudo systemctl start vncserver@:1.service
```

Verify VNC listener:
```bash
ss -tulpn | grep 5901
```

✅ Expected:
```
tcp LISTEN 0 5 0.0.0.0:5901 0.0.0.0:* users:(("Xvnc",pid=XXXX,fd=6))
```

---

## 5️⃣ Open AWS Security Group
| Type | Protocol | Port Range | Source | Description |
|------|-----------|-------------|---------|--------------|
| Custom TCP | TCP | **5901** | **Your IP (e.g., 203.0.113.25/32)** | Allow VNC access |

---

## 6️⃣ Run GUI Installer (as cognos user)
Once connected via VNC (as **cognos**):

### Create and grant permissions for the installation directory
```bash
sudo mkdir -p /opt/ibm/cognos
sudo chown -R cognos:cognos /opt/ibm/cognos
sudo chmod -R 755 /opt/ibm/cognos
```

### Upload and start installer
```bash
scp analytics-installer-4.0.8.bin ca_srv_lnx86_12.1.0.zip ec2-user@<EC2-IP>:/tmp/
sudo chown cognos:cognos /tmp/analytics-installer-4.0.8.bin /tmp/ca_srv_lnx86_12.1.0.zip
sudo chmod +x /tmp/analytics-installer-4.0.8.bin
cd /tmp
./analytics-installer-4.0.8.bin
```

🪶 **When prompted:**
> The directory `/opt/ibm/cognos/analytics` does not exist. Do you want to create it during installation?

✅ Click **Yes** — this is expected.

---

## 7️⃣ Add PostgreSQL JDBC Driver
```bash
wget https://jdbc.postgresql.org/download/postgresql-42.7.3.jar
sudo mv postgresql-42.7.3.jar /opt/ibm/cognos/analytics/drivers/
sudo chown cognos:cognos /opt/ibm/cognos/analytics/drivers/postgresql-42.7.3.jar
```

---

## 8️⃣ Configure Content Store (AWS RDS PostgreSQL)
```bash
sudo dnf install -y postgresql
psql -h database-psgres-test.cxmmkcq2kv5v.us-east-2.rds.amazonaws.com -U postgres -p 5432
```

At the `postgres=#` prompt:
```sql
CREATE DATABASE cognos_content;
CREATE USER cognos_user WITH PASSWORD 'StrongPassword123!';
GRANT CONNECT ON DATABASE cognos_content TO cognos_user;
\c cognos_content
GRANT USAGE, CREATE ON SCHEMA public TO cognos_user;
GRANT ALL PRIVILEGES ON DATABASE cognos_content TO cognos_user;
\dn+ public
```

✅ Verify:
```bash
psql -h database-psgres-test.cxmmkcq2kv5v.us-east-2.rds.amazonaws.com -U cognos_user -d cognos_content -p 5432
```

---

## 🌀 Aurora PostgreSQL Option

If your database is **Amazon Aurora PostgreSQL-Compatible**, Cognos works identically with just minor changes.

### Connection Details in Cognos Configuration
| Parameter | Example |
|------------|----------|
| Database | `cognos_content` |
| Hostname | `database-aurora-postgres-instance-1.cxmmkcq2kv5v.us-east-2.rds.amazonaws.com` |
| Port | `5432` |
| User | `cognos_user` |
| Password | `<your-secret>` |

### SQL Commands (same as PostgreSQL)
```sql
CREATE DATABASE cognos_content;
CREATE USER cognos_user WITH PASSWORD 'StrongPassword123!';
GRANT CONNECT ON DATABASE cognos_content TO cognos_user;
\c cognos_content
GRANT USAGE, CREATE ON SCHEMA public TO cognos_user;
GRANT ALL PRIVILEGES ON DATABASE cognos_content TO cognos_user;
\dn+ public
```

Use the **Aurora cluster writer endpoint** (never the reader endpoint).

✅ Optional JDBC URL with SSL:
```
jdbc:postgresql://database-aurora-postgres-instance-1.cxmmkcq2kv5v.us-east-2.rds.amazonaws.com:5432/cognos_content?sslmode=require
```

You can retrieve credentials from AWS Secrets Manager:
```bash
aws secretsmanager get-secret-value --secret-id <your-aurora-secret-name> --query 'SecretString' --output text
```

---

## 9️⃣ Start Cognos Services
Start via GUI (**Actions → Start**) or CLI:
```bash
/opt/ibm/cognos/analytics/bin64/cogconfig.sh -s
```

Check:
```bash
ps -ef | grep cognos
netstat -tulnp | grep 9300
```

Logs:
```
/opt/ibm/cognos/analytics/logs/
```

Access:
```
http://<EC2-public-IP>:9300/bi
```

---

✅ **Installation complete!**
You can now use the VNC desktop to manage Cognos, verify the PostgreSQL or Aurora content store, and access the Cognos Analytics web interface via port **9300**.

