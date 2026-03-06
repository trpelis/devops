# DevOps-journey

1. Started with RHEL based Linux OS, mostly using Konsol for my operations and learning, virtualisation in Oracle VM VirtualBox
2. Moved onto Bash scripting, some simple scripts which included encryption and creating random/shuffle data.
3. Doing some MySQL work and getting acquainted with RDBMS, combined with Bash, made script to generate random data using Faker tool.
4. Discovering wonderful world of Ansible for automatisation, I've made 2 virtual machines and done some playbooks which involved
NTP, cronjobs,  package installations and user management.
5. What to do after Ansible? Prometheus and Grafana, with node exporter, of course. I've made few hosts and then my node exporter scraped
system metrics from them, sent them to my server into Prometheus to stash them. Fetch them out and serve it on various dashboards in Grafana.


Things which are not included, but made small project to get myself familiar how high availability/load balancing works:
- made python app to listen on custom port and serve response when reached properly (this one is actually in python folder on GH)
- put them on 2 VM 
- configured NGINX as web service
- HAproxy as load-balancer where I used round robin balancing and created self-signed SSL certificate
- Keepalived as failover between HAproxy services


I also got myself little bit into Docker.



LINUX CHEATSHEET RHCSA STYLE



Legend:
- `[SERVERA]` — run on servera
- `[SERVERB]` — run on serverb
- `[WORKSTATION]` — run on workstation

---

## Section 1 — Packages and Directory Structure

### Task 1 — Verify apache is installed `[SERVERB]`
```bash
rpm -q httpd
```

### Task 2 — Install joe and verify `[SERVERB]`
```bash
dnf install -y joe
joe --version
```

### Task 3 — Show who installed the last package `[SERVERB]`
```bash
dnf history
dnf history info <ID>
```

### Task 4 — Create directory tree `[WORKSTATION]`
```bash
for mjesec in Ozu Sij Vel; do
  for smjer in Izlazne Ulazne; do
    for status in Obrada Odradeno Priprema; do
      mkdir -p ~/Documents/Racunovodstvo/$mjesec/Fakture/$smjer/$status
      touch ~/Documents/Racunovodstvo/$mjesec/Fakture/$smjer/$status/popis_faktura.txt
    done
  done
done

tree ~/Documents/Racunovodstvo/
# expected: 30 directories, 18 files
```

---

## Section 2 — Partitions, Filesystems, Users, ACL

### Task 5 — Partition /dev/vdb `[SERVERA]`
```bash
lsblk                  # verify disk name first
fdisk /dev/vdb
```
Inside fdisk:
```
n > p > 1 > Enter > +1G      # vdb1
n > p > 2 > Enter > +500M    # vdb2
n > p > 3 > Enter > +2G      # vdb3
t > 3 > 82                   # set vdb3 as swap
w                             # write and exit
```

```bash
mkfs.ext4 /dev/vdb1
mkfs.xfs  /dev/vdb2
mkswap    /dev/vdb3

mkdir -p /primjer-2/ext4
mkdir -p /primjer-2/xfs
```

### Task 6 — Persistent mount via fstab `[SERVERA]`
```bash
blkid /dev/vdb1
blkid /dev/vdb2
blkid /dev/vdb3

echo "UUID=<uuid1>  /primjer-2/ext4  ext4  defaults  0 2" >> /etc/fstab
echo "UUID=<uuid2>  /primjer-2/xfs   xfs   defaults  0 2" >> /etc/fstab
echo "UUID=<uuid3>  swap           swap  defaults  0 0" >> /etc/fstab

mount -a       # test before reboot — if this errors, fix fstab immediately
swapon -a

df -h /primjer-2/ext4 /primjer-2/xfs
swapon --show
```

### Task 7 — Users, groups, and ACL `[SERVERA]`
```bash
groupadd bosses
groupadd workers

useradd -G bosses boss1
useradd -G bosses boss2
useradd -G bosses boss3

useradd -G workers worker1
useradd -G workers worker2
useradd -G workers worker3

mkdir /bosses
mkdir /workers

chown root:bosses /bosses
chmod 770 /bosses

chown root:workers /workers
chmod 770 /workers

# Professors get read access to /workers via ACL
setfacl -m g:bosses:rx /workers

# Verify
getfacl /bosses
getfacl /workers
```

Test access `[SERVERA]`:
```bash
su - boss1
touch /bosses/test.txt    # must work
ls /workers/                # must work
touch /workers/test.txt     # must fail
exit

su - worker1
touch /workers/test.txt     # must work
ls /bosses/               # must fail
exit
```

---

## Section 3 — SSH Keys, Cron, Rsyslog, Journalctl

### Task 8 — Create user primjer3 and generate SSH keys `[SERVERA]`
```bash
useradd primjer3
su - primjer3
ssh-keygen -t rsa -b 4096    # press Enter for all prompts

ls -la ~/.ssh/
# id_rsa       private key
# id_rsa.pub   public key
```

### Task 9 — Passwordless SSH from servera (as primjer3) to serverb (as worker)

Step 1 — ensure worker exists `[SERVERB]`:
```bash
useradd worker
passwd worker
```

Step 2 — copy key and test `[SERVERA]` as primjer3:
```bash
ssh-copy-id worker@serverb   # enter worker password once

# test — must not ask for password
ssh worker@serverb
```

### Task 10 — Cron: write to /var/log/custom-log.log every 10 minutes `[SERVERA]` as root
```bash
crontab -e
```
```
*/10 * * * * echo "Probni ispit MI1" >> /var/log/custom-log.log
```
```bash
crontab -l    # verify
```

### Task 11 — Rsyslog: log all info messages to /var/log/info-log.log `[SERVERA]`
```bash
echo "*.info    /var/log/info-log.log" >> /etc/rsyslog.conf
systemctl restart rsyslog

# test
logger -p user.info "test info message"
tail /var/log/info-log.log
```

### Task 12 — Journalctl: logs from 17:00 today to 5 minutes ago `[SERVERA]`
```bash
journalctl --since "today 17:00" --until "$(date -d '5 minutes ago' '+%Y-%m-%d %H:%M:%S')"
```

---

## Section 4 — Network, Hostname, NTP

### Task 13 — Show all IP addresses `[SERVERA]`
```bash
ip a
```

### Task 14 — Show default route `[SERVERB]`
```bash
ip route show
# look for: default via ...
```

### Task 15 — Set hostname `[SERVERB]`
```bash
hostnamectl set-hostname primjer-ispita
hostnamectl    # verify
```
> `hostnamectl` writes to `/etc/hostname` automatically — survives reboot.

### Task 16 — Allow workstation to reach serverb by hostname

Step 1 — get serverb IP `[SERVERB]`:
```bash
ip a
```

Step 2 — add hosts entry `[WORKSTATION]` as root:
```bash
echo "<IP-serverb>  primjer-ispita" >> /etc/hosts

# test
ping -c 3 primjer-ispita
```

### Task 17 — Disable NTP sync `[SERVERA]`
```bash
timedatectl set-ntp false
timedatectl               # verify: NTP service: inactive
systemctl status chronyd  # should be inactive
```

---

## Section 5 — DNS with BIND

### Tasks 18 & 20 — Install and configure BIND `[SERVERB]`
```bash
dnf install -y bind bind-utils
systemctl enable --now named
```

Edit `/etc/named.conf` — change in `options` block:
```bash
vi /etc/named.conf
```
```
listen-on port 53 { any; };
allow-query     { any; };
```
Add zone at end of file:
```
zone "ispit.probni" IN {
    type master;
    file "/var/named/ispit.probni.zone";
};
```

Create zone file `[SERVERB]`:
```bash
vi /var/named/ispit.probni.zone
```
```
$TTL 86400
@   IN  SOA     serverb.ispit.probni. admin.ispit.probni. (
                2024010101  ; Serial — increment this on every change
                3600        ; Refresh
                900         ; Retry
                604800      ; Expire
                86400 )     ; Minimum TTL

@       IN  NS      serverb.ispit.probni.
serverb IN  A       <IP-serverb>

; Task 18 — A records pointing to servera
www     IN  A       <IP-servera>
web     IN  A       <IP-servera>

; Task 20 — MX record for email forwarding
@       IN  MX  10  mail.ispit.probni.
mail    IN  A       <IP-servera>
```

```bash
chown named:named /var/named/ispit.probni.zone

# Always validate before restart
named-checkconf
named-checkzone ispit.probni /var/named/ispit.probni.zone

systemctl restart named
```

### Task 19 — Point workstation DNS to serverb `[WORKSTATION]`
```bash
nmcli connection show
nmcli connection modify "<connection-name>" ipv4.dns "<IP-serverb>"
nmcli connection up "<connection-name>"

# test
dig www.ispit.probni
dig web.ispit.probni
dig MX ispit.probni
```

---

## Section 6 — Apache and Nginx Virtual Hosts

### Task 21 — Install Apache `[SERVERA]`
```bash
dnf install -y httpd
systemctl enable --now httpd
```

### Task 22 — Install Nginx `[SERVERB]`
```bash
dnf install -y nginx
systemctl enable --now nginx
```

### Task 23 — Apache: 2 virtual hosts `[SERVERA]`
```bash
mkdir -p /var/www/web.apache.local
mkdir -p /var/www/www.apache.local

echo "web.apache.local" > /var/www/web.apache.local/index.html
echo "www.apache.local" > /var/www/www.apache.local/index.html
```

```bash
vi /etc/httpd/conf.d/web.apache.local.conf
```
```apache
<VirtualHost *:80>
    ServerName web.apache.local
    DocumentRoot /var/www/web.apache.local
</VirtualHost>
```

```bash
vi /etc/httpd/conf.d/www.apache.local.conf
```
```apache
<VirtualHost *:80>
    ServerName www.apache.local
    DocumentRoot /var/www/www.apache.local
</VirtualHost>
```

```bash
apachectl configtest
systemctl restart httpd
```

If SELinux returns 403:
```bash
chcon -R -t httpd_sys_content_t /var/www/web.apache.local
chcon -R -t httpd_sys_content_t /var/www/www.apache.local
```

### Task 24 — Nginx: 2 server blocks `[SERVERB]`
```bash
mkdir -p /var/www/web.nginx.local
mkdir -p /var/www/www.nginx.local

echo "web.nginx.local" > /var/www/web.nginx.local/index.html
echo "www.nginx.local" > /var/www/www.nginx.local/index.html
```

```bash
vi /etc/nginx/conf.d/web.nginx.local.conf
```
```nginx
server {
    listen 80;
    server_name web.nginx.local;
    root /var/www/web.nginx.local;
    index index.html;
}
```

```bash
vi /etc/nginx/conf.d/www.nginx.local.conf
```
```nginx
server {
    listen 80;
    server_name www.nginx.local;
    root /var/www/www.nginx.local;
    index index.html;
}
```

```bash
nginx -t
systemctl restart nginx
```

If SELinux returns 403:
```bash
chcon -R -t httpd_sys_content_t /var/www/web.nginx.local
chcon -R -t httpd_sys_content_t /var/www/www.nginx.local
```

Firewall — run on whichever server needs it:
```bash
firewall-cmd --add-service=http --permanent
firewall-cmd --reload
```

### Task 25 — Prove functionality `[WORKSTATION]`
```bash
echo "<IP-servera>  web.apache.local" >> /etc/hosts
echo "<IP-servera>  www.apache.local" >> /etc/hosts
echo "<IP-serverb>  web.nginx.local"  >> /etc/hosts
echo "<IP-serverb>  www.nginx.local"  >> /etc/hosts

curl http://web.apache.local
curl http://www.apache.local
curl http://web.nginx.local
curl http://www.nginx.local
```

---

## Section 7 — LVM, Stratis, NFS

### Tasks 1-2 — Create PVs and VG `[SERVERA]`
```bash
lsblk    # find available disks

pvcreate /dev/vdc /dev/vdd
vgcreate primjer7VG /dev/vdc /dev/vdd

pvs
vgs
```

### Task 3 — LV lv-1: 1GB, block size 1024B, mount /primjer-7/lv-1 `[SERVERA]`
```bash
lvcreate -L 1G -n lv-1 primjer7VG
mkfs.ext4 -b 1024 /dev/primjer7VG/lv-1
mkdir -p /primjer-7/lv-1

echo "/dev/primjer7VG/lv-1  /primjer-7/lv-1  ext4  defaults  0 2" >> /etc/fstab
mount -a

tune2fs -l /dev/primjer7VG/lv-1 | grep "Block size"
```

### Task 4 — LV lv-2: 2GB, block size 512B, mount /primjer-7/lv-2 `[SERVERA]`
> ext4 minimum block size is 1024B — use mke2fs directly for 512B (creates ext2)

```bash
lvcreate -L 2G -n lv-2 primjer7VG
mke2fs -b 512 /dev/primjer7VG/lv-2
mkdir -p /primjer-7/lv-2

echo "/dev/primjer7VG/lv-2  /primjer-7/lv-2  ext2  defaults  0 2" >> /etc/fstab
mount -a

tune2fs -l /dev/primjer7VG/lv-2 | grep "Block size"
```

### Tasks 5-6 — NFS: share lv-1 (ro) and lv-2 (rw) `[SERVERA]`
```bash
dnf install -y nfs-utils
systemctl enable --now nfs-server

vi /etc/exports
```
```
/primjer-7/lv-1  *(ro,sync,no_subtree_check)
/primjer-7/lv-2  *(rw,sync,no_subtree_check)
```
```bash
exportfs -arv
showmount -e localhost

firewall-cmd --add-service=nfs --permanent
firewall-cmd --add-service=mountd --permanent
firewall-cmd --add-service=rpc-bind --permanent
firewall-cmd --reload
```

### Tasks 7-8 — Stratis pool and filesystems `[SERVERB]`
```bash
dnf install -y stratisd stratis-cli
systemctl enable --now stratisd

lsblk    # find 3 available block devices

stratis pool create primjer7pool /dev/vdc /dev/vdd /dev/vde

stratis filesystem create primjer7pool stratis-1
stratis filesystem create primjer7pool stratis-2

mkdir -p /primjer-7/stratis-1
mkdir -p /primjer-7/stratis-2

# Get UUIDs
blkid | grep stratis
```

Add to `/etc/fstab` `[SERVERB]` — `x-systemd.requires` is mandatory for Stratis:
```
UUID=<uuid1>  /primjer-7/stratis-1  xfs  defaults,x-systemd.requires=stratisd.service  0 0
UUID=<uuid2>  /primjer-7/stratis-2  xfs  defaults,x-systemd.requires=stratisd.service  0 0
```
```bash
mount -a
df -h /primjer-7/stratis-1 /primjer-7/stratis-2
```

### Task 9 — Mount NFS shares and test access `[SERVERB]`
```bash
dnf install -y nfs-utils

# check what servera is exporting
showmount -e <IP-servera>

mkdir -p /mnt/nfs-lv1
mkdir -p /mnt/nfs-lv2
```

Add to `/etc/fstab` `[SERVERB]`:
```
<IP-servera>:/primjer-7/lv-1  /mnt/nfs-lv1  nfs  defaults  0 0
<IP-servera>:/primjer-7/lv-2  /mnt/nfs-lv2  nfs  defaults  0 0
```
```bash
mount -a

# Test lv-1 (read-only)
touch /mnt/nfs-lv1/test.txt    # must fail
ls /mnt/nfs-lv1/               # must work

# Test lv-2 (read-write)
touch /mnt/nfs-lv2/test.txt    # must work
```

---

## Section 8 — MySQL

### Tasks 10-13 — Install MySQL, create DB, users, grants `[SERVERA]`
```bash
dnf install -y mysql-server
systemctl enable --now mysqld

mysql -u root
```
```sql
-- Task 11: create database
CREATE DATABASE primjer8DB;

-- Task 12: user with full access to entire database
CREATE USER 'primjer8User'@'%' IDENTIFIED BY 'Lozinka1!';
GRANT ALL PRIVILEGES ON primjer8DB.* TO 'primjer8User'@'%';

-- Task 13: table must exist before granting table-level access
USE primjer8DB;
CREATE TABLE primjer8Table (id INT PRIMARY KEY AUTO_INCREMENT, naziv VARCHAR(100));

CREATE USER 'primjer8DrugiUser'@'%' IDENTIFIED BY 'Lozinka1!';
GRANT ALL PRIVILEGES ON primjer8DB.primjer8Table TO 'primjer8DrugiUser'@'%';

FLUSH PRIVILEGES;

-- Verify
SHOW GRANTS FOR 'primjer8User'@'%';
SHOW GRANTS FOR 'primjer8DrugiUser'@'%';

EXIT;
```

---

## Section 9 — SELinux, Web on Custom Port, Process Priority, User Limits

All tasks in this primjer are on `[SERVERB]` unless noted.

### Task 14 — Set SELinux to Enforcing `[SERVERB]`
```bash
setenforce 1                   # immediate, not persistent

vi /etc/selinux/config
# set: SELINUX=enforcing

getenforce                     # verify: Enforcing
```

### Task 15 — Apache serving /webapp on port 85 `[SERVERB]`
```bash
dnf install -y httpd
mkdir -p /webapp
echo "webapp works" > /webapp/index.html

echo "Listen 85" >> /etc/httpd/conf/httpd.conf
```

```bash
vi /etc/httpd/conf.d/webapp.conf
```
```apache
<VirtualHost *:85>
    DocumentRoot /webapp
    <Directory /webapp>
        Require all granted
    </Directory>
</VirtualHost>
```

SELinux — allow port 85 for httpd:
```bash
semanage port -a -t http_port_t -p tcp 85
semanage port -l | grep http    # verify
```

SELinux — set correct context for /webapp:
```bash
semanage fcontext -a -t httpd_sys_content_t "/webapp(/.*)?"
restorecon -Rv /webapp
```

```bash
systemctl enable --now httpd
curl http://localhost:85    # verify
```

### Task 16 — Firewall: allow port 85 `[SERVERB]`
```bash
firewall-cmd --add-port=85/tcp --permanent
firewall-cmd --reload
```

### Task 17 — Set httpd process nice value to 12 `[SERVERB]`
```bash
systemctl edit httpd
```
```ini
[Service]
Nice=12
```
```bash
systemctl daemon-reload
systemctl restart httpd

ps -eo pid,ni,comm | grep httpd    # ni column should show 12
```

### Task 18 — Max open files for user worker = 10000 `[SERVERB]`
```bash
vi /etc/security/limits.conf
```
```
worker  hard  nofile  10000
worker  soft  nofile  10000
```
```bash
# verify
su - worker
ulimit -n    # should show 10000
exit
```

### Task 19 — Default nice value for worker processes = 3 `[SERVERB]`
```bash
vi /etc/security/limits.conf
```
```
worker  hard  priority  3
worker  soft  priority  3
```
```bash
# verify
su - worker
nice    # should show 3
exit
```

---

## General Troubleshooting Checklist

```bash
# Is the service running?
systemctl status <service>

# Is the service enabled for boot?
systemctl is-enabled <service>

# Firewall blocking?
firewall-cmd --list-all

# SELinux blocking?
ausearch -m avc -ts recent
getenforce

# Config syntax valid?
apachectl configtest
nginx -t
named-checkconf
named-checkzone <zone> <zonefile>

# fstab safe to reboot?
mount -a    # if this errors, fix fstab before rebooting
```

---

## Common Mistakes to Avoid

| Mistake | Consequence |
|--------|-------------|
| `systemctl enable` without `start` | Service not running until reboot |
| Device name in fstab instead of UUID | Breaks on disk reorder |
| `mount -a` not tested after fstab edit | Server does not boot |
| Missing `x-systemd.requires` for Stratis fstab | Stratis not mounted after reboot |
| SELinux context missing on web directories | 403 Forbidden with no clear error |
| `semanage port` missing for non-standard ports | httpd silently fails to bind |
| DNS zone Serial not incremented after zone edit | Zone changes not picked up |
| MySQL GRANT before table exists | SQL error on table-level grant |
| Forgetting `exportfs -arv` after editing /etc/exports | NFS changes not applied |
| `setenforce 1` without editing /etc/selinux/config | SELinux permissive again after reboot |
