# **1.1	Произведите базовую настройку устройств**

ISP
/etc/hostname
HQ-RTR BR-RTR
hostname <новое имя>

**На всех устройствах необходимо сконфигурировать IPv4**
etc/net/ifaces
Создать ens каталог и два файла: options, ipv4address
TYPE=eth              TYPE=eth
BOOTPROTO=dhcp        BOOTPROTO=static

На HQ-CLI, HQ-SRV, BR-SRV 
default via <ip-адрес_шлюза>

**HQ-RTR**
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

**BR-RTR**
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

# **1.2 Настройка ISP**

Iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
Iptables-save > /etc/sysconfig/iptables

/etc/net/sysctl.conf 
net.ipv4.ip_forward = 1

# **1.3 Создание учеток**

**Routers:**
Username net_admin
Password P@$$word
Role admin

Adduser sshuser, пароль задается при работе команды
Usermod -u 1010 sshuser
Usermod -aG sudo sshuser

Visudo
Изменить строку 
Sudo ALL=(ALL) NOPASSWD: ALL

# **1.4 SSH**

etc/openssh/sshd_config
Port 2024
AllowUsers sshuser
Banner
/etc/openssh/banner
В файле прописать содержимое баннера
Authorized access only

# **1.5 Tunnel**

**HQ-RTR**
int tunnel.52
Ip address 10.0.0.1/30
Ip tunnel 172.16.4.2 172.16.5.2 mode gre 

**BR-RTR**
int tunnel.52
Ip address 10.0.0.2/30
Ip tunnel 172.16.5.2 172.16.4.2 mode gre

# **1.6 OSPF**

**Passive interface**

**HQ-RTR**
Router ospf 1
Network 192.168.100.0/26 area 52
Network 192.168.200.0/28 area 52
Network 10.0.0.0/30 area 0
Area 0 authentication 

Int tunnel.52
Ip osp authentication
Ip osp authentication-key P@ssw0rd


**BR-RTR**
Network 192.168.10.0/27 area 52
Network 10.0.0.0/30 area 0
Area 0 authentication 

Int tunnel.52
Ip osp authentication
Ip osp authentication-key P@ssw0rd

# **1.7 NAT**

**HQ-RTR**
Ip nat pool NAT 192.168.100.1-192.168.100.20,192.168.200.1-192.168.200.20
Ip nat source dynamic inside pool NAT overload interface to-isp
Int to-isp
Ip nat outside
Int to-cli
Ip nat inside
Int to-srv
Ip nat inside

**BR-RTR**
Ip nat pool NAT 192.168.10.1-192.168.10.20
Ip nat source dynamic inside pool NAT overload interface to-isp
Int to-isp
Ip nat outside
Int to-srv
Ip nat inside

# **1.8 DHCP**

Ip pool DHCP 192.168.200.10-192.1.6.8.200.10
Dhcp-server 1
Pool DHCP 1
Mask 28 
Gateway 192.168.200.1
Dns 192.168.100.10
Domain-name au-team.irpo
Domain-search au-team.irpo
Теперь нужно присвоить интерфейсу нужный dhcp сервер
Int to-cli
Dhcp-server 1

# **1.9 DNS**

Apt-get install bind
В файле /etc/bind/rfc1912.conf нужно добавить строки с зонами

zone "au-team. irpo" {
type master;
file "au-team. irpo";
allow-update { none; };

zone "168.192. in-addr.arpa" {
type master;
file "168.192. in-addr.arpa";
allow-update { none; };

zone "16.172. in-addr.arpa" {
type master;
file "16.172. in-addr.arpa";
allow-update { none; };

16.172.in-addr.arpa

>
     IN  NS  HQ-SRV.au-team.irpo.
4.1  IN  NS  HQ-RTR.au-team.irpo.

168.192.in-addr.arpa

>
        IN  NS   HQ-SRV.au-team.irpo.
100.10  IN  PTR  HQ-SRV.au-team.irpo.
200.10  IN  PTR  HQ-CLI.au-team.irpo.

Au-team.irpo

>
        IN  NS  HQ-SRV.au-team.irpo.
HQ-SRV  IN  A  192.168.100.10
BR-SRV  IN  A  192.168.10.10
HQ-CLI  IN  A  192.168.200.10
HQ-RTR  IN  A  172.16.4.1
BR-RTR  IN  A  172.16.5.1
ISP     IN  A  172.16.4.1
wiki    IN  CNAME  ISP.au-team.irpo.
moodle  IN  CNAME  ISP.au-team.irpo.

# **1.10 Timezone**

apt-get install tzdata
timedatectl set-timezone Europa/Moscow

ntp timezone utc+3
























