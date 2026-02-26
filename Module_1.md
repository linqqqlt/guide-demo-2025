# Модуль 1 - Настройка сетевых устройств

## 1.1 Базовая настройка устройств

### Настройка имени хоста

**Все Alt-подобные:**
```bash
/etc/hostname
```

**HQ-RTR и BR-RTR:**
```bash
hostname <новое имя>
```

### Настройка IPv4 на всех устройствах

Путь: `/etc/net/ifaces`

Создать каталог `ens` и два файла: `options`, `ipv4address`

**Для DHCP:**
```
TYPE=eth
BOOTPROTO=dhcp
```

**Для статического IP:**
```
TYPE=eth
BOOTPROTO=static
```

**На HQ-CLI, HQ-SRV, BR-SRV добавить маршрут по умолчанию:**

Путь: `/etc/net/ifaces/ens*/ipv4route`
```bash
default via <ip-адрес_шлюза>
```

### Настройка HQ-RTR

```bash
port te0
service-instance to-isp
encapsulation untagged

port te1
service-instance to-srv
encapsulation dot1q 100 exact
rewrite pop 1

service-instance to-cli
encapsulation dot1q 200 exact
rewrite pop 1

service-instance vl999
encapsulation dot1q 999 exact
rewrite pop 1

interface to-isp
connect port te0 service-instance to-isp
ip address 172.16.1.2/28

interface to-srv
connect port te1 service-instance to-srv
ip address 192.168.100.1/27

interface to-cli
connect port te1 service-instance to-cli
ip address 192.168.200.1/24

interface vl999
ip address 192.168.99.1/29
```

### Настройка BR-RTR

```bash
port te0
service-instance to-isp
encapsulation untagged

port te1
service-instance to-srv
encapsulation untagged

interface to-isp
connect port te0 service-instance to-isp
ip address 172.16.2.2/28

interface to-srv
connect port te1 service-instance to-srv
ip address 192.168.0.1/28
```

## 1.2 Настройка ISP

### NAT и маршрутизация

```bash
iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
```

### Включение IP forwarding

Файл: `/etc/net/sysctl.conf`
```
net.ipv4.ip_forward = 1
```

## 1.3 Создание учетных записей

### На роутерах:

```bash
username net_admin
password P@$$word
role admin
```

### Создание пользователя sshuser:

```bash
adduser sshuser
# пароль задается при выполнении команды или же с помощью
passwd sshuser

usermod -u 1010 sshuser
usermod -aG wheel sshuser
```

### Настройка sudo

```bash
visudo
```

Найти строку с группой WHEEL и раскоментировать ее:
```
%WHEEL ALL=(ALL) NOPASSWD: ALL
```

## 1.4 Настройка SSH

### Основная конфигурация

Файл: `/etc/openssh/sshd_config`
```
Port 2026
AllowUsers sshuser
Banner /etc/openssh/banner
```

### Создание баннера

Файл: `/etc/openssh/banner`
```
Authorized access only
```

## 1.5 Настройка туннеля

### HQ-RTR

```bash
int tunnel.1
ip address 10.0.0.1/30
ip tunnel 172.16.1.2 172.16.2.2 mode gre
```

### BR-RTR

```bash
int tunnel.1
ip address 10.0.0.2/30
ip tunnel 172.16.2.2 172.16.1.2 mode gre
```

## 1.6 Настройка OSPF

### HQ-RTR

```bash
router ospf 1
network 192.168.100.0/27 area 0
network 192.168.200.0/24 area 0
network 10.0.0.0/30 area 0

int tunnel.1
ip ospf authentication
ip ospf authentication-key P@ssw0rd
```



### BR-RTR

```bash
router ospf 1
network 192.168.0.0/28 area 0
network 10.0.0.0/30 area 0

int tunnel.52
ip ospf authentication
ip ospf authentication-key P@ssw0rd
```

**Примечание:** Настроить passive interface где необходимо

## 1.7 Настройка NAT

### HQ-RTR

```bash
ip nat pool NAT 192.168.100.1-192.168.100.20,192.168.200.1-192.168.200.20
ip nat source dynamic inside-to-outside pool NAT overload interface to-isp

int to-isp
ip nat outside

int to-cli
ip nat inside

int to-srv
ip nat inside

int vl999
ip nat inside
```

### BR-RTR

```bash
ip nat pool NAT 192.168.0.1-192.168.0.20
ip nat source dynamic inside-to-outside pool NAT overload interface to-isp

int to-isp
ip nat outside

int to-srv
ip nat inside
```

## 1.8 Настройка DHCP

### Создание пула и сервера

```bash
ip pool DHCP 192.168.200.1-192.168.200.20

dhcp-server 1
pool DHCP 1
mask 24
gateway 192.168.200.1
dns 192.168.100.1
domain-name au-team.irpo
domain-search au-team.irpo
```

### Присвоение DHCP сервера интерфейсу

```bash
int to-cli
dhcp-server 1
```

## 1.9 Настройка DNS

### Установка BIND

```bash
apt-get install bind9 bind-utils -y 
```

### Настройка зон

Файл: `/etc/bind/rfc1912.conf `

```bind
zone "au-team.irpo" {
    type master;
    file "au-team.irpo";
    allow-update { none; };
};

zone "100.168.192.in-addr.arpa" {
    type master;
    file "100.168.192.in-addr.arpa";
    allow-update { none; };
};

zone "200.168.192.in-addr.arpa" {
    type master;
    file "200.168.192.in-addr.arpa";
    allow-update { none; };
};
```
#### Копируете файлы зоны, которые уже есть в каталоге /etc/bind/zone, меняя им названия:

```bash
cp /etc/bind/zone/empty /var/lib/bind/etc/zone/au-team.irpo
cp /etc/bind/zone/empty /var/lib/bind/etc/zone/100.168.192.in-addr.arpa
cp /etc/bind/zone/empty /var/lib/bind/etc/zone/200.168.192.in-addr.arpa
```

### Файл зоны: 100.168.192.in-addr.arpa

```bind
$TTL 86400
@       IN  SOA au-team.irpo. root.au-team.irpo. (
            2023010101  ; Serial
            3600        ; Refresh
            1800        ; Retry
            604800      ; Expire
            86400       ; Minimum TTL
)
        IN  NS  au-team.irpo.
1       IN  PTR hq-rtr.au-team.irpo.
2       IN  PTR hq-srv.au-team.irpo.
```

### Файл зоны: 200.168.192.in-addr.arpa

```bind
$TTL 86400
@       IN  SOA au-team.irpo. root.au-team.irpo. (
            2023010101  ; Serial
            3600        ; Refresh
            1800        ; Retry
            604800      ; Expire
            86400       ; Minimum TTL
)
        IN  NS   au-team.irpo.
1       IN  PTR  hq-rtr.au-team.irpo.
2       IN  PTR  hq-cli.au-team.irpo.
```

### Файл зоны: au-team.irpo

```bind
$TTL 86400
@       IN  SOA au-team.irpo. root.au-team.irpo. (
            2023010101  ; Serial
            3600        ; Refresh
            1800        ; Retry
            604800      ; Expire
            86400       ; Minimum TTL
)
        IN  NS  au-team.irpo.
        IN  A   192.168.100.2
hq-srv  IN  A   192.168.100.2
hq-cli  IN  A   192.168.200.2
hq-rtr  IN  A   192.168.100.1
hq-rtr  IN  A   192.168.200.1
hq-rtr  IN  A   192.168.99.1
docker  IN  A   172.16.1.1
web     IN  A   172.16.2.1
br-srv  IN  A   192.168.0.2
br-rtr  IN  A   192.168.0.1
```

#### После всех этих настроек главное прописать 2 команды и перезагрузить сервис

```bash
rndc-confgen > /var/lib/bind/etc/rndc.key
sed -i ‘6,$d’ /var/lib/bind/etc/rndc.key

systemctl enable --now bind.service
systemctl restart bind.service
```

### Поменять nameserver в файле /etc/resolv.conf

```bash
search au-team.irpo
nameserver 192.168.100.2
```

## 1.10 Настройка часового пояса

### Установка и настройка

```bash
apt-get install tzdata
timedatectl set-timezone Europe/Moscow
```

На роутерах:

```bash
ntp timezone utc+3
```
