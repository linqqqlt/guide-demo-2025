# Модуль 1 - Настройка сетевых устройств

## 1.1 Базовая настройка устройств

### Настройка имени хоста

**ISP:**
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

interface to-isp
connect port te0 service-instance to-isp
ip address 172.16.4.2/28

interface to-srv
connect port te1 service-instance to-srv
ip address 192.168.100.1/26

interface to-cli
connect port te1 service-instance to-cli
ip address 192.168.200.1/28
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
ip address 172.16.5.2/28

interface to-srv
connect port te1 service-instance to-srv
ip address 192.168.10.1/27
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
# пароль задается при выполнении команды
usermod -u 1010 sshuser
usermod -aG sudo sshuser
```

### Настройка sudo

```bash
visudo
```

Изменить строку на:
```
%sudo ALL=(ALL) NOPASSWD: ALL
```

## 1.4 Настройка SSH

### Основная конфигурация

Файл: `/etc/openssh/sshd_config`
```
Port 2024
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
int tunnel.52
ip address 10.0.0.1/30
ip tunnel 172.16.4.2 172.16.5.2 mode gre
```

### BR-RTR

```bash
int tunnel.52
ip address 10.0.0.2/30
ip tunnel 172.16.5.2 172.16.4.2 mode gre
```

## 1.6 Настройка OSPF

### HQ-RTR

```bash
router ospf 1
network 192.168.100.0/26 area 52
network 192.168.200.0/28 area 52
network 10.0.0.0/30 area 0
area 0 authentication

int tunnel.52
ip ospf authentication
ip ospf authentication-key P@ssw0rd
```

**Примечание:** Настроить passive interface где необходимо.

### BR-RTR

```bash
router ospf 1
network 192.168.10.0/27 area 52
network 10.0.0.0/30 area 0
area 0 authentication

int tunnel.52
ip ospf authentication
ip ospf authentication-key P@ssw0rd
```

## 1.7 Настройка NAT

### HQ-RTR

```bash
ip nat pool NAT 192.168.100.1-192.168.100.20,192.168.200.1-192.168.200.20
ip nat source dynamic inside pool NAT overload interface to-isp

int to-isp
ip nat outside

int to-cli
ip nat inside

int to-srv
ip nat inside
```

### BR-RTR

```bash
ip nat pool NAT 192.168.10.1-192.168.10.20
ip nat source dynamic inside pool NAT overload interface to-isp

int to-isp
ip nat outside

int to-srv
ip nat inside
```

## 1.8 Настройка DHCP

### Создание пула и сервера

```bash
ip pool DHCP 192.168.200.10-192.168.200.20

dhcp-server 1
pool DHCP 1
mask 28
gateway 192.168.200.1
dns 192.168.100.10
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
apt-get install bind9
```

### Настройка зон

Файл: `/etc/bind/rfc1912.conf `

```bind
zone "au-team.irpo" {
    type master;
    file "au-team.irpo";
    allow-update { none; };
};

zone "168.192.in-addr.arpa" {
    type master;
    file "168.192.in-addr.arpa";
    allow-update { none; };
};

zone "16.172.in-addr.arpa" {
    type master;
    file "16.172.in-addr.arpa";
    allow-update { none; };
};
```

### Файл зоны: 16.172.in-addr.arpa

```bind
$TTL 86400
@       IN  SOA HQ-SRV.au-team.irpo. admin.au-team.irpo. (
            2023010101  ; Serial
            3600        ; Refresh
            1800        ; Retry
            604800      ; Expire
            86400       ; Minimum TTL
)
        IN  NS  HQ-SRV.au-team.irpo.
4.1     IN  PTR HQ-RTR.au-team.irpo.
```

### Файл зоны: 168.192.in-addr.arpa

```bind
$TTL 86400
@       IN  SOA HQ-SRV.au-team.irpo. admin.au-team.irpo. (
            2023010101  ; Serial
            3600        ; Refresh
            1800        ; Retry
            604800      ; Expire
            86400       ; Minimum TTL
)
        IN  NS   HQ-SRV.au-team.irpo.
100.10  IN  PTR  HQ-SRV.au-team.irpo.
200.10  IN  PTR  HQ-CLI.au-team.irpo.
```

### Файл зоны: au-team.irpo

```bind
$TTL 86400
@       IN  SOA HQ-SRV.au-team.irpo. admin.au-team.irpo. (
            2023010101  ; Serial
            3600        ; Refresh
            1800        ; Retry
            604800      ; Expire
            86400       ; Minimum TTL
)
        IN  NS  HQ-SRV.au-team.irpo.
HQ-SRV  IN  A   192.168.100.10
BR-SRV  IN  A   192.168.10.10
HQ-CLI  IN  A   192.168.200.10
HQ-RTR  IN  A   172.16.4.2
BR-RTR  IN  A   172.16.5.2
ISP     IN  A   172.16.4.1
wiki    IN  CNAME  ISP.au-team.irpo.
moodle  IN  CNAME  ISP.au-team.irpo.
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
