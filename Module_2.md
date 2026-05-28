# Модуль 2 - Дополнительные сервисы

## 2.1 Доменный контроллер Samba на BR-SRV

### Установка и первоначальная настройка

### 1. Контроллер домена Samba DC на BR-SRV

**Установка:**
```bash
apt-get install task-samba-dc -y
```

**Подготовка:**
```bash
rm -f /etc/samba/smb.conf
rm -rf /var/lib/samba /var/cache/samba
mkdir -p /var/lib/samba/sysvol
```

**Провизионирование:**
```bash
samba-tool domain provision
```
- Realm: **AU-TEAM.IRPO**
- Domain: **AU-TEAM**
- Server Role: **dc**
- DNS backend: **SAMBA_INTERNAL**
- DNS forwarder: **77.88.8.8**

```bash
systemctl enable --now samba
```

**Создание пользователей:**
```bash
for i in 1 2 3 4 5; do
    samba-tool user create hquser${i} P@ssw0rd
    samba-tool user setexpiry hquser${i} --noexpiry
done
```

**Создание группы:**
```bash
samba-tool group add hq
for i in 1 2 3 4 5; do
    samba-tool group addmembers hq hquser${i}
done
```

**Ввод HQ-CLI в домен:**
```bash
# На HQ-CLI:
apt-get install task-auth-ad-sssd -y
system-auth write ad au-team.irpo AU-TEAM BR-SRV.au-team.irpo administrator P@ssw0rd
```

**Права группы hq на HQ-CLI (visudo):**
```
%hq ALL=(ALL) NOPASSWD: /bin/cat, /bin/grep, /usr/bin/id
```

## 2.2 Файловое хранилище

### Создание RAID 5 на HQ-SRV

```bash
mdadm --create --level=5 --raid-devices=3 /dev/md/md0 /dev/sdb /dev/sdc /dev/sdd
mkfs -t ext4 /dev/md/md0
```

### Монтирование RAID массива

Файл: `/etc/fstab`
```
/dev/md/md0    /raid5    ext4    defaults    0    0
```

```bash
mkdir -p /raid5
mount -a
systemctl daemon-reload
```

### Настройка NFS сервера

```bash
apt-get update
apt-get install nfs-kernel-server

mkdir -p /raid5/nfs
chown nobody:nogroup /raid5/nfs
chmod 777 /raid5/nfs
```

**Конфигурация экспорта:**

Файл: `/etc/exports`
```
/raid5/nfs 192.168.200.0/28(rw,sync,no_subtree_check,no_root_squash)
```
**тут айпишник сети клиента, то есть то КУДА мы раздаем диск сетевой**

```bash
exportfs -a
systemctl start nfs-kernel-server
systemctl enable nfs-kernel-server
```

### Автомонтирование на HQ-CLI

```bash
apt-get install nfs-common
mkdir -p /mnt/nfs
mount 192.168.100.10:/raid5/nfs /mnt/nfs
```

Файл: `/etc/fstab`
```
192.168.100.10:/raid5/nfs    /mnt/nfs    nfs    defaults    0    0
```

## 2.3 Настройка времени (Chrony)

### Настройка NTP сервера на ISP

```bash
apt-get install chrony
```

Файл: `/etc/chrony.conf`
```
pool ru.pool.ntp.org iburst
allow all
local stratum 5
```

```bash
systemctl enable --now chronyd
```

### Настройка на роутерах (HQ-RTR & BR-RTR)

```bash
ntp server 172.16.4.1
```

### Настройка на серверах (BR-SRV, HQ-SRV & HQ-CLI)

```bash
apt-get install chrony
systemctl enable --now chronyd
```

Файл: `/etc/chrony.conf`
```
server 172.16.4.1
```

## 5. Ansible
```bash
apt-get update && apt-get install –y ansible sshpass 
```

Файл : `vim /etc/ansible/hosts`

```bash
[Servers ]
HQ-SRV ansible_host=192.168.100.2

[Routers]
HQ-RTR ansible_host=10.10.10.1
BR-RTR ansible_host=192.168.0.1

[Clients]
HQ-CLI ansible_host=192.168.200.2

[Servers : vars ]
ansible_user=sshuser
ansible_password=Pessw0rd
ansible_port=2026

[Routers : vars ]
ansible_user=net_admin
ansible_password=Pessw0rd
ansible_conect ion=network_cli
ansible_network_os=ios

[Clients :vars ]
ansible_user=user
ansible_password=resu

[all :vars]
ansible_python_interpreter=/usr/bin/python3
```

`vim /etc/ansible/ansible.cfg`

```bash
[defaults]

inventory = /etc/ansible/hosts
ost_key_checking = False
```

`apt-get install –y python3-module-pip
pip3 install ansible-pylibssh
`
### HQ-RTR, BR-RTR

`security none`

### BR-SRV

`ansible -m ping all`

**ЕСЛИ ПИНГ НА КАКОЕ-ТО УСТРОЙСТВО НЕ ПОШЛО ПРОВЕРЬ НАСТРОЙКИ SSH НА УСТРОЙСТВАХ И В ФАЙЛЕ `vim/etc/ansible/hosts`**

## 8. Проброс портов

### HQ-RTR

```
hq-rtr(config)#ip nat source static tcp 192.168.100.2 80 172.16.1.2 8080
hq-rtr(config)#ip nat source static tcp 192.168.100.2 2026 172.16.1.2 2026
hq-rtr(config)#write memory
```

```
br-rtr(config)#ip nat source static tcp 192.168.0.2 8080 172.16.2.2 8080
br-rtr(config)#ip nat source static tcp 192.168.0.2 2026 172.16.2.2 2026
br-rtr(config)#write memory
```

## 6. Веб приложение

### BR-SRV

```bash
apt-get install –y docker-engine docker-compose-v2
systemctl enable --now docker.service
mount /dev/sr0 /mnt/
docker load < /mnt/docker/site_latest.tar
docker load < /mnt/docker/mariadb_latest.tar
```

Файл: `vim compose.yaml`
```bash
services :
database:
conta iner_name: db
image: mar iadb : 10.11
restart: always
ports:
- "3306:3306"
environment :
MARIADB_DATABASE: "testdb"
MARIADB_USER: "testc"
MARIADB_PASSWORD: "Pessw0rd"
MARIADB_ROOT_PASSWORD: "toor"

app:
conta iner_name: testapp
image: site: latest
restart: always
ports:
- "8080:8000"
environment :
DB_TYPE: "maria"
DB_HOST: "192.168.0.2"
DB_PORT: "3306"
DB_NAME: "testdb"
DB_USER: "testc"
DB_PASS: "Pessw0rd"
depends_on:
- database
```

```bash
docker compose up -d
```

## 7. Веб приложение на HQ-SRV

```bash
apt-get install –y lamp-server
mount /dev/sr0 /mnt/
cp /mnt/web/index.php /var/www/html
cp /mnt/web/logo.png /var/www/html
```

Файл: `vim /var/www/html/index.php`

```bash
?php
Sservername = "localhost";
Şusername = "webc";
Spassword = "Pessw0rd";
Sdbname = "webdb";

Sconn = new mysqli($servername, $username, $password, $dbname);
```

```bash
systemctl enable --now mariadb
mariadb –u root
CREATE DATABASE webdb;
CREATE USER ‘webc’@’localhost’ IDENTIFIED BY ‘P@ssw0rd’;
GRANT ALL PRIVILEGES ON webdb.* TO ‘webc’@’localhost’ WITH GRANT OPTION;
EXIT;
```

```bash
mariadb –u webc –p –D webdb < /mnt/web/dump.sql
```

```bash
systemctl enable --now httpd2
```

## Обратный прокси NGINX

```bash
apt-get install -y nginx
```

Файл: `vim /etc/nginx/sites-available.d/default.conf`

```bash
server {
listen 80;
server_name web.au-team. irpo;

location / {
proxy_pass http://172.16.1.2:8080;
proxy_set_header Host Shost;
proxy_set_header X-Real-IP Sremote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;

server {
listen 80;
server_name docker.au-team. irpo;

location / {
proxy_pass http://172.16.2.2:8080;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

```bash
ln -s /etc/nginx/sites-available.d/default.conf /etc/nginx/sites-enabled.d/
```

```bash
systemctl enable --now nginx
```

## 11. YANDEX ZZZ GOYDA

```bash
apt-get install –y yandex-browser-stable
```
