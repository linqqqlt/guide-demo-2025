# Модуль 2 - Дополнительные сервисы

## 2.1 Доменный контроллер Samba на BR-SRV

### Установка и первоначальная настройка

```bash
apt-get install task-samba-dc
rm -f /etc/samba/smb.conf
rm -rf /var/lib/samba /var/cache/samba
mkdir –p /var/lib/samba/sysvol
```

```bash
samba-tool domain provision
```

**Параметры конфигурации:**
- Realm: `AU-TEAM.IRPO`
- Domain: `AU-TEAM`
- Server role: `dc`
- DNS backend: `SAMBA_INTERNAL`
- Server forward: `77.88.8.8`

```bash
systemctl enable --now samba
```

### Настройка DNS зон

**Создание обратных зон:**
```bash
samba-tool dns zonecreate 192.168.10.10 168.192.in-addr.arpa
samba-tool dns zonecreate 192.168.10.10 16.172.in-addr.arpa
```

**Добавление записей:**
```bash
samba-tool dns add 192.168.10.10 au-team.irpo HQ-RTR A 172.16.4.2 -U Administrator%P@ssw0rd
samba-tool dns add 192.168.10.10 16.172.in-addr.arpa 4.2 PTR HQ-RTR -U Administrator%P@ssw0rd

samba-tool dns add 192.168.10.10 au-team.irpo BR-RTR A 172.16.5.2 -U Administrator%P@ssw0rd
samba-tool dns add 192.168.10.10 16.172.in-addr.arpa 5.2 PTR BR-RTR -U Administrator%P@ssw0rd

samba-tool dns add 192.168.10.10 au-team.irpo HQ-SRV A 192.168.100.10 -U Administrator%P@ssw0rd
samba-tool dns add 192.168.10.10 168.192.in-addr.arpa 100.10 PTR HQ-SRV -U Administrator%P@ssw0rd

samba-tool dns add 192.168.10.10 au-team.irpo HQ-CLI A 192.168.200.10 -U Administrator%P@ssw0rd
samba-tool dns add 192.168.10.10 168.192.in-addr.arpa 200.10 PTR HQ-CLI -U Administrator%P@ssw0rd

samba-tool dns add 192.168.10.10 au-team.irpo ISP A 172.16.4.1 -U Administrator%P@ssw0rd
samba-tool dns add 192.168.10.10 16.172.in-addr.arpa 4.1 PTR ISP -U Administrator%P@ssw0rd

samba-tool dns add 192.168.10.10 au-team.irpo moodle CNAME ISP -U Administrator%P@ssw0rd
samba-tool dns add 192.168.10.10 au-team.irpo wiki CNAME ISP -U Administrator%P@ssw0rd
```

### Создание пользователей и групп

**Создание пользователей:**
```bash
samba-tool user create user1.hq P@ssw0rd && samba-tool user setexpiry user1.hq
samba-tool user create user2.hq P@ssw0rd && samba-tool user setexpiry user2.hq
samba-tool user create user3.hq P@ssw0rd && samba-tool user setexpiry user3.hq
samba-tool user create user4.hq P@ssw0rd && samba-tool user setexpiry user4.hq
samba-tool user create user5.hq P@ssw0rd && samba-tool user setexpiry user5.hq
```

**Создание группы и добавление пользователей:**
```bash
samba-tool group add hq
samba-tool group addmembers hq user1.hq
samba-tool group addmembers hq user2.hq
samba-tool group addmembers hq user3.hq
samba-tool group addmembers hq user4.hq
samba-tool group addmembers hq user5.hq
```

### Настройка прав пользователей на HQ-CLI

Файл: `/etc/sudoers`
```bash
%hq ALL=(ALL) NOPASSWD: /bin/cat,/bin/grep,/usr/bin/id
%hq ALL=(ALL) !ALL
```

Альтернативный вариант:
```bash
%AU-TEAM.IRPO\\hq ALL=(ALL) NOPASSWD: /bin/cat,/bin/grep,/usr/bin/id
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

Альтернативно:
```
server isp.au-team.irpo
```

## 2.4 MediaWiki в Docker

### Установка Docker

```bash
apt-get install docker-engine docker-compose -y
systemctl enable --now docker.service
```

### Создание конфигурации Docker Compose

Файл: `/home/user/wiki.yml`
```yaml
version: '3.8'

services:
  wiki:
    image: mediawiki
    restart: always
    volumes:
      - /var/www/html/images
      # - ./LocalSettings.php:/var/www/html/LocalSettings.php
    ports:
      - "8080:80"
    depends_on:
      - db

  db:
    image: mariadb
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: 'WikiP@ssw0rd'
      MYSQL_DATABASE: 'mediawiki'
      MYSQL_USER: 'wiki'
      MYSQL_PASSWORD: 'WikiP@ssw0rd'
    volumes:
      - dbvolume:/var/lib/mariadb

volumes:
  dbvolume:
```

### Запуск контейнеров

```bash
cd /home/user
docker-compose -f wiki.yml up -d
```

### Настройка MediaWiki

1. Зайти с клиента по адресу `http://IP_СЕРВЕРА:8080`
2. Пройти веб-установку
3. Скачать файл `LocalSettings.php`
4. Скопировать файл на сервер:
   ```bash
   scp user@192.168.10.10:/home/user/Загрузки/LocalSettings.php /home/user/
   ```
5. Раскомментировать строку в `wiki.yml` и перезапустить:
   ```bash
   docker-compose -f wiki.yml restart
   ```

## 2.5 Статическая трансляция на ISP

### Проброс MediaWiki (172.16.4.1:80 > 192.168.10.10:8080)

```bash
# Перенаправление трафика
iptables -t nat -A PREROUTING -d 172.16.4.1 -p tcp --dport 80 -j DNAT --to-destination 192.168.10.10:8080
iptables -A FORWARD -d 192.168.10.10 -p tcp --dport 8080 -j ACCEPT
iptables -t nat -A POSTROUTING -d 192.168.10.10 -p tcp --dport 8080 -j MASQUERADE
```

### Проброс SSH HQ-SRV (172.16.4.1:2024 > 192.168.100.10:2024)

```bash
iptables -t nat -A PREROUTING -d 172.16.4.1 -p tcp --dport 2024 -j DNAT --to-destination 192.168.100.10:2024
iptables -A FORWARD -d 192.168.100.10 -p tcp --dport 2024 -j ACCEPT
iptables -t nat -A POSTROUTING -d 192.168.100.10 -p tcp --dport 2024 -j MASQUERADE
```

### Проброс SSH BR-SRV (172.16.4.1:2025 > 192.168.10.10:2024)

```bash
iptables -t nat -A PREROUTING -d 172.16.4.1 -p tcp --dport 2025 -j DNAT --to-destination 192.168.10.10:2024
iptables -A FORWARD -d 192.168.10.10 -p tcp --dport 2024 -j ACCEPT
iptables -t nat -A POSTROUTING -d 192.168.10.10 -p tcp --dport 2024 -j MASQUERADE
```

### Сохранение правил

```bash
iptables-save > /etc/sysconfig/iptables
systemctl restart iptables
```

## 2.6 Moodle

### Установка пакетов

```bash
apt-get install moodle moodle-apache2 moodle-local-mysql mariadb-server
systemctl enable --now mariadb
systemctl enable --now apache2
```

### Создание базы данных

mysql -u root

```sql
CREATE DATABASE moodle DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci;
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, CREATE TEMPORARY TABLES, DROP, INDEX, ALTER ON moodle.* TO moodle@localhost IDENTIFIED BY 'P@ssw0rd';
FLUSH PRIVILEGES;
quit
```

```bash
mysqladmin -u root reload
```

### Настройка конфигурации

1. Перейти по адресу сервера с клиента и пройти веб-установку
2. После установки отредактировать файл `/var/www/webapps/moodle/config.php`:
   - Заменить тип базы данных на `mariadb`
3. Настроить PHP в файле `/etc/php/8.0/apache2-mod_php/php.ini`

## 2.7 Перенаправление через NGINX

### Установка NGINX

```bash
apt-get install nginx
```

### Создание конфигурации виртуального хоста

Файл: `/etc/nginx/sites-available/moodle`
```nginx
server {
    listen 80;
    server_name moodle.au-team.irpo;

    location / {
        proxy_pass http://192.168.100.10;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Активация конфигурации

```bash
ln -s /etc/nginx/sites-available/moodle /etc/nginx/sites-enabled/
systemctl enable --now nginx
systemctl reload nginx
```
