## Полное руководство по настройке инфраструктуры

### 1. Базовая настройка

#### 1.1 Настройка имен устройств

На каждом устройстве установите FQDN:
```bash
sudo hostnamectl set-hostname <fqdn>
```
Пример:
```bash
sudo hostnamectl set-hostname r-dt.au.team
```
Проверьте изменения:
```bash
hostnamectl
```

#### 1.2 Настройка IP-адресов

Пример конфигурации для серверов в `/etc/network/interfaces` (для Alt Linux):
```ini
auto eth0
iface eth0 inet static
  address 192.168.11.10
  netmask 255.255.255.0
  gateway 192.168.11.1
  dns-nameservers 192.168.11.2 8.8.8.8
```
Применение изменений:
```bash
sudo systemctl restart networking
```

#### 1.3 Создание пользователя `sshuser`

```bash
sudo useradd -m -s /bin/bash sshuser
sudo passwd sshuser
sudo usermod -aG sudo sshuser
```
Настроить `sudo` без пароля:
```bash
echo "sshuser ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/sshuser
```

### 2. Настройка коммутаторов (SW1-HQ, SW2-HQ, SW3-HQ)

#### 2.1 Установка Open vSwitch
```bash
sudo apt install openvswitch-switch -y
```
Создание коммутатора:
```bash
sudo ovs-vsctl add-br SW1-HQ
```
Добавление портов:
```bash
sudo ovs-vsctl add-port SW1-HQ eth1
sudo ovs-vsctl add-port SW1-HQ eth2
```

#### 2.2 Создание VLAN
```bash
sudo ovs-vsctl add-br vlan330
sudo ovs-vsctl set port vlan330 tag=330
```

### 3. Настройка маршрутизаторов (R-DT, R-HQ) (EcoRouter через CLI)

#### 3.1 Настройка WAN на R-DT
```bash
configure
set interfaces ethernet eth0 address 172.16.4.14/28
set interfaces ethernet eth0 description "WAN"
set protocols static route 0.0.0.0/0 next-hop 172.16.4.1
set nat source rule 10 outbound-interface eth0
set nat source rule 10 source address 192.168.33.0/24
set nat source rule 10 translation address masquerade
set system ipv4-forwarding enable
commit
save
```

#### 3.2 Настройка LAN на R-DT
```bash
set interfaces ethernet eth1 address 192.168.33.1/24
set interfaces ethernet eth1 description "LAN"
commit
save
```

#### 3.3 Настройка WAN на R-HQ
```bash
configure
set interfaces ethernet eth0 address 172.16.5.14/28
set interfaces ethernet eth0 description "WAN"
set protocols static route 0.0.0.0/0 next-hop 172.16.5.1
set nat source rule 10 outbound-interface eth0
set nat source rule 10 source address 192.168.11.0/24
set nat source rule 10 translation address masquerade
set system ipv4-forwarding enable
commit
save
```

#### 3.4 Настройка LAN на R-HQ
```bash
set interfaces ethernet eth1 address 192.168.11.1/24
set interfaces ethernet eth1 description "LAN"
commit
save
```

### 4. Настройка туннеля GRE между R-DT и R-HQ
```bash
configure
set interfaces tunnel tun0 mode gre
set interfaces tunnel tun0 local-address 172.16.4.14
set interfaces tunnel tun0 remote-address 172.16.5.14
set interfaces tunnel tun0 address 10.10.10.1/30
commit
save
```
На R-HQ:
```bash
configure
set interfaces tunnel tun0 mode gre
set interfaces tunnel tun0 local-address 172.16.5.14
set interfaces tunnel tun0 remote-address 172.16.4.14
set interfaces tunnel tun0 address 10.10.10.2/30
commit
save
```

### 5. Настройка OSPF для связи между офисами
```bash
configure
set protocols ospf area 0 network 192.168.33.0/24
set protocols ospf area 0 network 10.10.10.0/30
commit
save
```
На R-HQ:
```bash
configure
set protocols ospf area 0 network 192.168.11.0/24
set protocols ospf area 0 network 10.10.10.0/30
commit
save
```

### 6. Проверка соединения
```bash
ping 10.10.10.2 # на R-DT
ping 10.10.10.1 # на R-HQ
```

