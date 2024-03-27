Добавление serial порта в Гипервизоре:
```
qm set <VM ID> -serial0 socket
```
Хост `/etc/init/ttyS0.conf`:
```
# ttyS0 - getty
start on stopped rc RUNLEVEL=[12345]
stop on runlevel [!12345]
respawn
exec /sbin/getty -L 115200 ttyS0 vt102
```
Конфигурация `grub` `/etc/default/grub`:
```
GRUB_CMDLINE_LINUX ='console=tty0 console=ttyS0,115200'
```
Update:
```
update-grub
```
Включение serial порта:
```
systemctl enable serial-getty@ttyS0.service
```
Перезагружаемся и заходим через `xterm.js`. Теперь доступны скроллинг, вставка, копирование и произвольный размер окна.
1.

Полное доменное имя:
```
config
hostname rtr-hq.company.prof
do commit
do confirm
```

Команды для просмотра интерфейсов:
```
sh ip int
sh int stat
```

Настройка ip-адреса (ISP-гекс):
```
interface te1/0/2
ip address 11.11.11.2/30
description ISP
exit
```

Параметры DNS:
```
domain name company.prof
domain name-server 11.11.11.1
```

Настройка шлюза:
```
ip route 0.0.0.0/0 11.11.11.1
```
проверьте доступ в интернет

Настройка подынтерфейсов (Router-on-a-stick) vlan100:
```
interface te1/0/3.100
ip firewall disable
description vlan100
ip address 10.0.10.1/27
exit
```

vlan200:
```
interface te1/0/3.200
ip firewall disable
description vlan200
ip address 10.0.10.33/27
exit
```

vlan300:
```
interface te1/0/3.300
ip firewall disable
description vlan300
ip address 10.0.10.65/27
exit
```

Сохранение:
```
do commit
do confirm
```
развилка-бара
То же самое, кроме IP-адреса

прочее:

Полное доменное имя:
```
hostnamectl set-hostname <VM_NAME>.company.prof;exec bash
```

Настройка интерфейсов:
```
sed -i 's/DISABLED=yes/DISABLED=no/g' /etc/net/ifaces/ens18/options
sed -i 's/NM_CONTROLLED=yes/NM_CONTROLLED=no/g' /etc/net/ifaces/ens18/options
echo 10.0.20.2/24 > /etc/net/ifaces/ens18/ipv4address
echo default via 10.0.20.1 > /etc/net/ifaces/ens18/ipv4route
systemctl restart network
ping -c 4 8.8.8.8
```

сервера защищенная оболочка

Создание пользователя sshuser:
```
adduser sshuser
passwd sshuser
P@ssw0rd
P@ssw0rd
echo "sshuser ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
usermod -aG wheel sshuser
sudo -i
```

SSH

На серверах, к которым подключаемся:
```
mkdir /home/sshuser
```
```
chown sshuser:sshuser /home/sshuser
```
```
cd /home/sshuser
```
```
mkdir -p .ssh/
```
```
chmod 700 .ssh
```
```
touch .ssh/authorized_keys
```
```
chmod 600 .ssh/authorized_keys
```
```
chown sshuser:sshuser .ssh/authorized_keys
```

На машине с которого подключаемся к серверам:
```
ssh-keygen -t rsa -b 2048 -f srv_ssh_key
```
```
mkdir .ssh
```
```
mv srv_ssh_key* .ssh/
```
Конфиг для автоматического подключения `.ssh/config`:
```
Host srv-hq
        HostName 10.0.10.2
        User sshuser
        IdentityFile .ssh/srv_ssh_key
        Port 2023
Host srv-br
        HostName 10.0.20.2
        User sshuser
        IdentityFile .ssh/srv_ssh_key
        Port 2023
```
```
chmod 600 .ssh/config
```
Копирование ключа на удаленный сервер:
```
ssh-copy-id -i .ssh/srv_ssh_key.pub sshuser@10.0.10.2
```
```
ssh-copy-id -i .ssh/srv_ssh_key.pub sshuser@10.0.20.2
```
На сервере `/etc/ssh/sshd_config`:
```
AllowUsers sshuser
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
AuthorizedKeysFile .ssh/authorized_keys
Port 2023
```
```
systemctl restart sshd
```
Подключение:
```
ssh srv-hq
```

2. RAIDS

сервер-гекс

Создаем разделы:
```
gdisk /dev/sdb
```
o - y  
n - 1 - enter - enter - 8e00  
p  
w - y  
То же самое с `sdc`  
Инициализация:
```
pvcreate /dev/sd{b,c}1
```
В группу:
```
vgcreate vg01 /dev/sd{b,c}1
```
Логический том-зеркало RAID1:
```
lvcreate -l 100%FREE -n lvmirror -m1 vg01
```
Создаём ФС:
```
mkfs.ext4 /dev/vg01/lvmirror
```
Директория:
```
mkdir /opt/data
```
Автозагрузка `/etc/fstab`:
```
echo "UUID=$(blkid -s UUID -o value /dev/vg01/lvmirror) /opt/data ext4 defaults 1 2" | tee -a /etc/fstab
```
Монтаж:
```
mount -av
```
Проверка:
```
df -h
```

Сервер-Бара

Создаем разделы:
```
gdisk /dev/sdb
```
o - y  
n - 1 - enter - enter - 8e00  
p  
w - y  
То же самое с `sdc`  
Инициализация:
```
pvcreate /dev/sd{b,c}1
```
В группу:
```
vgcreate vg01 /dev/sd{b,c}1
```
Striped том:
```
lvcreate -i2 -l 100%FREE -n lvstriped vg01
```
> -i2 - количество полос  
> -l 100%FREE - занять всё место  
> -n lvstriped - имя 'lvstriped'  
> vg01 - имя группы

ФС:
```
mkfs.ext4 /dev/vg01/lvstriped
```
Создание ключа:
```
dd if=/dev/urandom of=/root/ext2.key bs=1 count=4096
```
Шифрование тома с помощью ключа:
```
cryptsetup luksFormat /dev/vg01/lvstriped /root/ext2.key
```
Открытие тома ключом:
```
cryptsetup luksOpen -d /root/ext2.key /dev/vg01/lvstriped lvstriped
```
ФС на расшифрованном томе:
```
mkfs.ext4 /dev/mapper/lvstriped
```
Директория монтирования:
```
mkdir /opt/data
```
Автозагрузка `/etc/fstab`
```
echo "UUID=$(blkid -s UUID -o value /dev/mapper/lvstriped) /opt/data ext4 defaults 0 0" | tee -a /etc/fstab
```
Монтаж:
```
mount -av
```
Автозагрузка зашифрованного тома с помощью ключа `/etc/crypttab`:
```
echo "lvstreped UUID=$(blkid -s UUID -o value /dev/vg01/lvstriped) /root/ext2.key luks" | tee -a /etc/crypttab
```
reboot, df -h, lsblk

3. Связь

Сыч-Гекс

Внимание! в интерфейсах ens... не должно быть прочих файлов, кроме `options`, иначе порты не привяжутся

Временное назначение ip-адреса (смотрящего в сторону развилка-гекс):
```
ip link add link ens18 name ens18.300 type vlan id 300
ip link set dev ens18.300 up
ip addr add 10.0.10.66/27 dev ens18.300
ip route add 0.0.0.0/0 via 10.0.10.65
echo nameserver 8.8.8.8 > /etc/resolv.conf
```

Обновление пакетов и установка `openvswitch`:
```
apt-get update && apt-get install -y openvswitch
```
Автозагрузка:
```
systemctl enable --now openvswitch
```

ens18 - rtr-hq  
ens19 - cicd-hq - vlan200
ens20 - srv-hq - vlan100
ens21 - cli-hq - vlan200

Создаем каталоги для ens19,ens20,ens21:
```
mkdir /etc/net/ifaces/ens{19,20,21}
```

Для моста:
```
mkdir /etc/net/ifaces/ovs0
```

Management интерфейс:
```
mkdir /etc/net/ifaces/mgmt
```

Не удалять настройки openvswitch:
```
sed -i "s/OVS_REMOVE=yes/OVS_REMOVE=no/g" /etc/net/ifaces/default/options
```
Мост `/etc/net/ifaces/ovs0/options`:


> TYPE - тип интерфейса, bridge;  
> HOST - добавляемые интерфейсы в bridge.  

mgmt `/etc/net/ifaces/mgmt/options`:


> TYPE - тип интерфейса (internal);  
> BOOTPROTO - статически;  
> CONFIG_IPV4 - использовать ipv4;  
> BRIDGE - определяет к какому мосту необходимо добавить данный интерфейс;  
> VID - определяет принадлежность интерфейса к VLAN.  

Поднимаем интерфейсы:
```
cp /etc/net/ifaces/ens18/options /etc/net/ifaces/ens19/options
cp /etc/net/ifaces/ens18/options /etc/net/ifaces/ens20/options
cp /etc/net/ifaces/ens18/options /etc/net/ifaces/ens21/options
```

Назначаем Ip, default gateway на mgmt:
```
echo 10.0.10.66/27 > /etc/net/ifaces/mgmt/ipv4address
```
```
echo default via 10.0.10.65 > /etc/net/ifaces/mgmt/ipv4route
```

Перезапуск network:
```
systemctl restart network
```

Проверка:
```
ip -c --br a
```
```
ovs-vsctl show
```

ens18 - rtr-hq делаем trunk и пропускаем VLANs:
```
ovs-vsctl set port ens18 trunk=100,200,300
```

ens19 - tag=200
```
ovs-vsctl set port ens19 tag=200
```

ens20 - tag=100:
```
ovs-vsctl set port ens20 tag=100
```

ens21 - tag=200
```
ovs-vsctl set port ens21 tag=200
```

Включаем инкапсулирование пакетов по 802.1q:
```
modprobe 8021q
```
Проверка включения

4. Слон


Сервер-Гекс

Установка `postgresql`:
```
apt-get install -y postgresql16 postgresql16-server postgresql16-contrib
```
Развертываем БД:
```
/etc/init.d/postgresql initdb
```
Автозагрузка:
```
systemctl enable --now postgresql
```
Включаем прослушивание всех адресов:
```
nano /var/lib/pgsql/data/postgresql.conf
```


Перезагружаем:
```
systemctl restart postgresql
```
Проверяем:
```
ss -tlpn | grep postgres
```

Заходим в БД под рутом:
```
psql -U postgres
```
Создаем базы данных "prod","test","dev":
```
CREATE DATABASE prod;
```
```
CREATE DATABASE test;
```
```
CREATE DATABASE dev;
```
Создаем пользователей "produser","testuser","devuser":
```
CREATE USER produser WITH PASSWORD 'P@ssw0rd';
```
```
CREATE USER testuser WITH PASSWORD 'P@ssw0rd';
```
```
CREATE USER devuser WITH PASSWORD 'P@ssw0rd';
```
Назначаем на БД владельца:
```
GRANT ALL PRIVILEGES ON DATABASE prod to produser;
```
```
GRANT ALL PRIVILEGES ON DATABASE test to testuser;
```
```
GRANT ALL PRIVILEGES ON DATABASE dev to devuser;
```
Заполняем БД с помощью `pgbench`:
```
pgqbench -U postgres -i prod
```
```
pgqbench -U postgres -i test
```
```
pgqbench -U postgres -i dev
```
Проверяем


Настройка аутентификации для удаленного доступа:
```
nano /var/lib/pgsql/data/pg_hba.conf
```


Перезапускаем:
```
systemctl restart postgresql
```

сервер-бара

Установка:
```
apt-get install -y postgresql16 postgresql16-server postgresql16-contrib
```


Репликация

сервер-гекса

Конфиг
```
nano /var/lib/pgsql/data/postgresql.conf
```
```
wal_level = replica
max_wal_senders = 2
max_replication_slots = 2
hot_standby = on
hot_standby_feedback = on
```

> wal_level указывает, сколько информации записывается в WAL (журнал операций, который используется для репликации);
> max_wal_senders — количество планируемых слейвов;
> max_replication_slots — максимальное число слотов репликации; 
> hot_standby — определяет, можно ли подключаться к postgresql для выполнения запросов в процессе восстановления;
> hot_standby_feedback — определяет, будет ли сервер slave сообщать мастеру о запросах, которые он выполняет.

```
systemctl restart postgresql
```

Сервер-Барагаон

Останавливаем:
```
systemctl stop postgresql
```

Удаляем каталог:
```
rm -rf /var/lib/pgsql/data/*
```

Запускаем репликацию (может длиться несколько десятков минут):
```
pg_basebackup -h 10.0.10.2 -U postgres -D /var/lib/pgsql/data --wal-method=stream --write-recovery-conf
```



Назначаем владельца:
```
chown -R postgres:postgres /var/lib/pgsql/data/
```

Запускаем:
```
systemctl start postgresql
```

Сервер-Гекса

Создаем тест:
```
psql -U postgres
```
```
CREATE DATABASE testik;
```

Сервер-Барагаон

HAPROXY

Сыч-Гекса

Установка haproxy:
```
apt-get install haproxy
```
Автозагрузка:
```
systemctl enable --now haproxy
```
Конфиг `/etc/haproxy/haproxy.cfg`:
```
listen stats
    bind 0.0.0.0:8989
    mode http
    stats enable
    stats uri /haproxy_stats
    stats realm HAProxy\ Statistics
    stats auth admin:toor
    stats admin if TRUE

frontend postgre
    bind 0.0.0.0:5432
    default_backend my-web

backend postgre
    balance first
    server srv-hq 10.0.10.2:5432 check
    server srv-br 10.0.20.2:5432 check
```


5.	Ната

развилка-гекса:
```
config
security zone public
exit
int te1/0/2
security-zone public
exit
object-group network COMPANY
ip address-range 10.0.10.1-10.0.10.254
exit
object-group network WAN
ip address-range 11.11.11.11
exit
nat source
pool WAN
ip address-range 11.11.11.11
exit
ruleset SNAT
to zone public
rule 1
match source-address COMPANY
action source-nat pool WAN
enable
exit
exit
commit
confirm

```
Развилка-Барагаон:
```
config
security zone public
exit
int te1/0/2
security-zone public
exit
object-group network COMPANY
ip address-range 10.0.20.1-10.0.20.254
exit
object-group network WAN
ip address-range 22.22.22.22
exit
nat source
pool WAN
ip address-range 22.22.22.22
exit
ruleset SNAT
to zone public
rule 1
match source-address COMPANY
action source-nat pool WAN
enable
exit
exit
commit
confirm

```


6.	 конфигурация хостов


Развилка-гекса

DNS-сервер - Сервер-геска
```
configure terminal
ip dhcp-server pool COMPANY-HQ
network 10.0.10.32/27
default-lease-time 3:00:00
address-range 10.0.10.33-10.0.10.62
excluded-address-range 10.0.10.33                      
default-router 10.0.10.33 
dns-server 10.0.10.2
domain-name company.prof
exit
```

Включение DHCP-сервера:
```
ip dhcp-server
```


7.	Настройка DNS для серверов

сервер-гекса

Установка bind и bind-utils:
```
apt-get update && apt-get install -y bind bind-utils
```
Конфиг:
```
nano /etc/bind/options.conf
```
```
listen-on { any; };
allow-query { any; };
allow-transfer { 10.0.20.2; }; 
```


Включаем resolv:
```
nano /etc/net/ifaces/ens18/resolv.conf
```
```
systemctl restart network
```
Автозагрузка bind:
```
systemctl enable --now bind
```
Создаем прямую и обратные зоны:
```
nano /etc/bind/local.conf
```


Копируем дефолты:
```
cp /etc/bind/zone/{localhost,company.db}
```
```
cp /etc/bind/zone/127.in-addr.arpa /etc/bind/zone/10.0.10.in-addr.arpa.db
```
```
cp /etc/bind/zone/127.in-addr.arpa /etc/bind/zone/20.0.10.in-addr.arpa.db
```
Назначаем права:
```
chown root:named /etc/bind/zone/company.db
```
```
chown root:named /etc/bind/zone/10.0.10.in-addr.arpa.db
```
```
chown root:named /etc/bind/zone/20.0.10.in-addr.arpa.db
```
Настраиваем зону прямого просмотра `/etc/bind/zone/company.prof`:

![изображение](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/9d3bddf9-e8db-4cfc-8971-b3a6ff0f38ac)

Настраиваем зону обратного просмотра `/etc/bind/zone/10.0.10.in-addr.arpa.db`:

![изображение](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/3fee1532-458b-4e10-967f-b858a4a43b63)

Настраиваем зону обратного просмотра `/etc/bind/zone/20.0.10.in-addr.arpa.db`:

![изображение](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/c9f18689-b324-4125-836d-91bdb23b1075)

Проверка зон:
```
named-checkconf -z
```
![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/d2de05e9-3830-4846-86a1-2974bb64dbf5)

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/e1b6fb80-63df-4cf5-8dd6-2f3d46a1dc3b)

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/21d53c0b-27d9-4af9-bbe3-b64035b0ffad)

Сервер-Барагаон

Конфиг
```
vim /etc/bind/options.conf
```

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/e90d49ce-6735-4fdb-b44a-0b1c62b8305a)

Добавляем зоны

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/ea22291d-0b41-4044-a271-1fbb32f26185)

Резолв `/etc/net/ifaces/ens18/resolv.conf`:
```
search company.prof
nameserver 10.0.10.2
nameserver 10.0.20.2
```
Перезапуск адаптера:
```
systemctl restart network
```
Автозагрузка:
```
systemctl enable --now bind
```
SLAVE:
```
control bind-slave enabled
```

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/dc174c7e-e960-42c6-8b9d-7522df00989a)

Разрешение имени хоста test

Сервер-гекса

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/a73800a1-b7cf-48e3-8160-49b3b14a0060)

Копируем дефолт для зоны:
```
cp /etc/bind/zone/{localdomain,test.company.db}
```

Задаём права, владельца:
```
chown root:named /etc/bind/zone/test.company.db
```
Настраиваем зону:
```
vim /etc/bind/zone/test.company.db
```

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/ae1dcd6e-d5b9-4980-8105-d14b74905083)

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/c8735610-56d5-419b-bdc8-3efc48a969b0)

Перезапускаем:
```
systemctl restart bind
```

Сервер-Барагаон
Добавляем зону `/etc/bind/local.conf`:

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/2d279b81-8bfd-453b-8efa-dc3d87476c40)

Задаём права, владельца:
```
chown root:named /etc/bind/zone/test.company.db
```

Редактируем зону `/etc/bind/zone/test.company.db`:

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/1aa398ac-dd3a-4887-b975-14ff7a3bf633)

Перезапускаем:
```
systemctl restart bind
```
8.ансамбль

Установка:
```
apt-get install -y ansible sshpass
```
```
nano /etc/ansible/inventory.yml
```
```yml
Networking:
  hosts:
    rtr-hq:
      ansible_host: 10.0.10.1
      ansible_user: admin
      ansible_password: P@ssw0rd
      ansible_port: 22
    rtr-br:
      ansible_host: 10.0.20.1
      ansible_user: admin
      ansible_password: P@ssw0rd
      ansible_port: 22
Servers:
  hosts:
    srv-hq:
      ansible_host: 10.0.10.2
      ansible_ssh_private_key_file: ~/.ssh/srv_ssh_key
      ansible_user: sshuser
      ansible_port: 2023
    srv-br:
      ansible_host: 10.0.20.2
      ansible_ssh_private_key_file: ~/.ssh/srv_ssh_key
      ansible_user: sshuser
      ansible_port: 22
Clients:
  hosts:
    cli-hq:
      ansible_host: 10.0.10.34
      ansible_user: root
      ansible_password: P@ssw0rd
      ansible_port: 22
    cli-br:
      ansible_host: 10.0.20.34
      ansible_user: root
      ansible_password: P@ssw0rd
      ansible_port: 22
```
```
nano /etc/ansible/ansible.cfg
```

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/534a5b2d-380e-4659-8455-98883556eb9f)

```
nano /etc/ansible/gathering.yml
```
```yml
---
- name: show ip and hostname from routers
  hosts: Networking
  gather_facts: false
  tasks:
  - name: show ip
    esr_command:
      commands: show ip interfaces

- name: show ip and fqdn
  hosts: Servers, Clients
  tasks:
  - name: print ip and hostname
    debug:
      msg: "IP address: {{ ansible_default_ipv4.address }}, Hostname {{ ansible_hostname }}"
  - name: create file
    copy:
      content: ""
      dest: /etc/ansible/output.yml
      mode: '0644'

  - name: save output
    copy:
      content: "IP address: {{ ansible_default_ipv4.address }}, Hostname {{ ansible_hostname }}"
      dest: /etc/ansible/output.yml
```
```
cd /etc/ansible
```

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/a3c7e3de-d1fb-4a08-bc55-cc0765404a36)

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/f2ccec52-702a-4ba5-8c61-ba3dd9c5d147)

9.	Защищенная связь


```
tunnel gre 1
  ttl 16
  ip firewall disable
  local address 11.11.11.11
  remote address 22.22.22.22
  ip address 172.16.1.1/30
  enable
exit
```
```
security zone-pair public self
  rule 1
    description "ICMP"
    action permit
    match protocol icmp
    enable
  exit
  rule 2
    description "GRE"
    action permit
    match protocol gre
    enable
  exit
exit
```

```
security ike proposal ike_prop1
  authentication algorithm md5
  encryption algorithm aes128
  dh-group 2
exit
```

```
security ike policy ike_pol1
  pre-shared-key ascii-text P@ssw0rd
  proposal ike_prop1
exit
```

```
security ike gateway ike_gw1
  ike-policy ike_pol1
  local address 11.11.11.11
  local network 11.11.11.11/32 protocol gre 
  remote address 22.22.22.22
  remote network 22.22.22.22/32 protocol gre 
  mode policy-based
exit
```

```
security ike gateway ike_gw1
  ike-policy ike_pol1
  local address 22.22.22.22
  local network 22.22.22.22/32 protocol gre 
  remote address 11.11.11.11
  remote network 11.11.11.11/32 protocol gre 
  mode policy-based
exit
```

```
security ipsec proposal ipsec_prop1
  authentication algorithm md5
  encryption algorithm aes128
  pfs dh-group 2
exit
```

```
security ipsec policy ipsec_pol1
  proposal ipsec_prop1
exit
```
```
security ipsec vpn ipsec1
  ike establish-tunnel route
  ike gateway ike_gw1
  ike ipsec-policy ipsec_pol1
  enable
exit
```

```
security zone-pair public self
  rule 3
    description "ESP"
    action permit
    match protocol esp
    enable
  exit
  rule 4
    description "AH"
    action permit
    match protocol ah
    enable
  exit
exit
```

```
router ospf 1
  area 0.0.0.0
    enable
  exit
  enable
exit
```

```
tunnel gre 1
  ip ospf instance 1
  ip ospf
exit

interface gigabitethernet 1/0/2.100
  ip ospf instance 1
  ip ospf
exit

interface gigabitethernet 1/0/2.200
  ip ospf instance 1
  ip ospf
exit

interface gigabitethernet 1/0/2.300
  ip ospf instance 1
  ip ospf
exit
```

```
security zone-pair public self
  rule 5
    description "OSPF"
    action permit
    match protocol ospf
    enable
  exit
exit
```

Проверка 
```
sh ip ospf neighbors
```
```
sh ip route
```

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/c77249a2-eec9-4b23-9992-a10e891f4a68)

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/fa2b8166-cc1f-4d7d-a230-7c0f8f882e6c)


10.	 FreeIPA


смотри в 10.md

11. proxy-сервер

сервер-гекса

Установка:
```
apt-get install alterator-squid
```
Доступ осуществляется по `https://ip-address:8080/`

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/1def4702-a3ce-4862-8d03-ceb34e7b1fdd)

# Обеспечение отказоустойчивости


Буду использовать Mobaxterm, который базирован на putty

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/d5e36f86-9a5e-452b-87e4-88b44ed16b4a)

Tools - MobaKeyGen

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/363a48d1-5411-4440-aab4-a26d336ef793)

Generate - копируем публичный ключ - Save private key - меняем расширение ключа на `pem`

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/e4390e4b-c7d2-4ed7-b887-956e73d2e6f6)

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/6aebff10-f997-4026-bb77-27a56759f02e)

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/99cc5ea4-1fd4-45b9-93d5-94dfa01ea772)

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/4ee74e84-096a-4a58-91ee-5070bea69f67)

```
sudo -i
```

```
apt-get update && apt-get install -y wget unzip
```

Зеркало Яндекса
```
wget https://hashicorp-releases.yandexcloud.net/terraform/1.7.2/terraform_1.7.2_linux_amd64.zip
```

Зеркало VK
```
wget https://hashicorp-releases.mcs.mail.ru/terraform/1.7.3/terraform_1.7.3_linux_amd64.zip
```

```
unzip terraform_1.7.2_linux_amd64.zip -d /usr/local/bin/
```

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/b8af500d-480f-4eab-9ad6-59d3a40cc252)
