# Проектная работа: Построение ЛВС деревообрабатывающего предприятия

## 1. Цель и задачи

**Цель:** Спроектировать и настроить масштабируемую, отказоустойчивую и защищённую ЛВС деревообрабатывающего предприятия.

**Задачи:**
1. Спроектировать двухуровневую модель сети.
2. Разработать план VLAN и VLSM-адресации.
3. Настроить коммутацию: VLAN, trunk, STP, EtherChannel, Port Security.
4. Реализовать маршрутизацию между VLAN на L3-коммутаторе.
5. Развернуть DHCPv4 для автоматической выдачи адресов.
6. Настроить NAT для выхода в Интернет.
7. Обеспечить изоляцию VLAN с помощью стандартных ACL.
8. Настроить управление: NTP, SNMP, CDP/LLDP.

---

## 2. Используемое оборудование и модули

| Устройство | Роль в проекте | Количество |
|------------|----------------|------------|
| **Cisco 2911** | ISP-Router (имитация провайдера) | 1 |
| **Cisco 2911** | Edge-Router (NAT, OSPF) | 1 |
| **Cisco 3650-24PS** | Ядро (L3-коммутация, маршрутизация) | 1 |
| **Cisco 3650-24PS** | Оптическая агрегация (SFP, L3) | 2 |
| **Cisco 2960-24TT** | Коммутаторы доступа этажей (VLAN 10, Guest) | 2 |
| **Cisco 3650-24PS** | Коммутаторы доступа цехов и проходных (SFP + PoE) | 6 |
| **GLC-LH-SMD** | Оптический SFP-трансивер 1 Гбит/с, 10 км | 14 |



---

## 3. Топология

<img width="940" height="712" alt="image" src="https://github.com/user-attachments/assets/4861d371-3ff8-4ed5-a038-f4e22cd24210" />

---

## 4. План VLAN и IP-адресации

| VLAN | Имя | Назначение | Подсеть | Шлюз |
|------|-----|------------|---------|------|
| 10 | DOMAIN | Доменные ПК и принтеры | 192.168.10.0/24 | 192.168.10.1 |
| 20 | Ceh1 | Цех №1 (изолирован) | 192.168.20.0/28 | 192.168.20.1 |
| 21 | Ceh2 | Цех №2 (изолирован) | 192.168.21.0/28 | 192.168.21.1 |
| 22 | Ceh3 | Цех №3 (изолирован) | 192.168.22.0/28 | 192.168.22.1 |
| 23 | Ceh4 | Цех №4 (изолирован) | 192.168.23.0/28 | 192.168.23.1 |
| 30 | TP | Проходная №1 (изолирована) | 192.168.30.0/28 | 192.168.30.1 |
| 31 | ATP | Проходная №2 (изолирована) | 192.168.31.0/28 | 192.168.31.1 |
| 50 | Video | Видеонаблюдение | 192.168.50.0/24 | 192.168.50.1 |
| 60 | Guest | Гостевой Wi-Fi (только Интернет) | 192.168.60.0/24 | 192.168.60.1 |
| 99 | MGMT | Управление оборудованием | 192.168.99.0/28 | 192.168.99.1 |


**IP управления коммутаторов (VLAN 99):**

| Устройство | IP |
|------------|-----|
| Core-SW | 192.168.99.1 |
| SW-Optical-1 | 192.168.99.2 |
| SW-Optical-2 | 192.168.99.3 |
| SW-Floor1 | 192.168.99.4 |
| SW-Floor2 | 192.168.99.5 |
| SW-Ceh1 | 192.168.99.6 |
| SW-Ceh2 | 192.168.99.7 |
| SW-Ceh3 | 192.168.99.8 |
| SW-Ceh4 | 192.168.99.9 |
| SW-TP | 192.168.99.10 |
| SW-ATP | 192.168.99.11 |

---

## 5. Конфигурации устройств

### 5.1. ISP-Router (Cisco 2911)

```
enable
configure terminal
hostname ISP-Router
no ip domain-lookup
!
interface GigabitEthernet0/0
 description To Edge-Router
 ip address 10.0.0.1 255.255.255.252
 no shutdown
!
interface Loopback0
 description Imitation of Internet Server
 ip address 8.8.8.8 255.255.255.255
!
ip route 0.0.0.0 0.0.0.0 Loopback0
!
end
write memory
```

**Пояснение:** ISP-Router имитирует интернет-провайдера. Loopback-интерфейс с адресом 8.8.8.8 служит имитацией внешнего сервера. Маршрут по умолчанию направлен на Loopback.

### 5.2. Edge-Router (Cisco 2911)

```
enable
configure terminal
hostname Edge-Router
no ip domain-lookup
!
enable secret class
service password-encryption
!
username admin secret AdminPass123
!
line console 0
 password cisco
 login
 logging synchronous
 exec-timeout 5 0
!
line vty 0 15
 login local
 transport input ssh
 exec-timeout 5 0
!
ip domain-name example.com
crypto key generate rsa modulus 1024
!
banner motd ^C Authorized Access Only! ^C
!
interface GigabitEthernet0/0
 description To ISP
 ip address 10.0.0.2 255.255.255.252
 ip nat outside
 no shutdown
!
interface GigabitEthernet0/1
 description To Core-SW
 ip address 10.0.1.1 255.255.255.252
 ip nat inside
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 10.0.0.1
!
access-list 1 permit 192.168.0.0 0.0.255.255
ip nat inside source list 1 interface GigabitEthernet0/0 overload
!
router ospf 1
 network 10.0.0.0 0.0.0.3 area 0
 network 10.0.1.0 0.0.0.3 area 0
!
end
write memory
```

**Пояснение:** Edge-Router выполняет NAT для выхода в Интернет. OSPF настроен для обмена маршрутами с Core-SW. ACL 1 разрешает трансляцию для всех внутренних сетей 192.168.x.x.

### 5.3. Core-SW (Cisco 3650, L3)

```
enable
configure terminal
hostname Core-SW
no ip domain-lookup
!
ip routing
!
enable secret class
service password-encryption
!
username admin secret AdminPass123
!
line console 0
 password cisco
 login
 logging synchronous
 exec-timeout 5 0
!
line vty 0 15
 login local
 transport input ssh
 exec-timeout 5 0
!
ip domain-name example.com
crypto key generate rsa modulus 1024
!
banner motd ^C Authorized Access Only! ^C
!
vlan 10
 name DOMAIN
vlan 20
 name Ceh1
vlan 21
 name Ceh2
vlan 22
 name Ceh3
vlan 23
 name Ceh4
vlan 30
 name TP
vlan 31
 name ATP
vlan 50
 name Video
vlan 60
 name Guest
vlan 99
 name MGMT
!
interface vlan 10
 ip address 192.168.10.1 255.255.255.0
!
interface vlan 20
 ip address 192.168.20.1 255.255.255.240
!
interface vlan 21
 ip address 192.168.21.1 255.255.255.240
!
interface vlan 22
 ip address 192.168.22.1 255.255.255.240
!
interface vlan 23
 ip address 192.168.23.1 255.255.255.240
!
interface vlan 30
 ip address 192.168.30.1 255.255.255.240
!
interface vlan 31
 ip address 192.168.31.1 255.255.255.240
!
interface vlan 50
 ip address 192.168.50.1 255.255.255.0
!
interface vlan 60
 ip address 192.168.60.1 255.255.255.0
!
interface vlan 99
 ip address 192.168.99.1 255.255.255.240
!
interface GigabitEthernet1/0/1
 description To Edge-Router
 no switchport
 ip address 10.0.1.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet1/0/2
 description To SW-Floor1
 switchport mode trunk
 switchport trunk allowed vlan 10,60,99
!
interface GigabitEthernet1/0/3
 description To SW-Floor2
 switchport mode trunk
 switchport trunk allowed vlan 10,60,99
!
interface GigabitEthernet1/1/1
 description To SW-Optical-1 (fiber)
 switchport mode trunk
 switchport trunk allowed vlan 20,21,22,23,30,31,50,60,99
!
spanning-tree vlan 1-4094 root primary
!
ip route 0.0.0.0 0.0.0.0 10.0.1.1
!
router ospf 1
 network 10.0.1.0 0.0.0.3 area 0
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.15 area 0
 network 192.168.21.0 0.0.0.15 area 0
 network 192.168.22.0 0.0.0.15 area 0
 network 192.168.23.0 0.0.0.15 area 0
 network 192.168.30.0 0.0.0.15 area 0
 network 192.168.31.0 0.0.0.15 area 0
 network 192.168.50.0 0.0.0.255 area 0
 network 192.168.60.0 0.0.0.255 area 0
 network 192.168.99.0 0.0.0.15 area 0
!
ip dhcp excluded-address 192.168.10.1 192.168.10.10
ip dhcp excluded-address 192.168.20.1 192.168.20.2
ip dhcp excluded-address 192.168.21.1 192.168.21.2
ip dhcp excluded-address 192.168.22.1 192.168.22.2
ip dhcp excluded-address 192.168.23.1 192.168.23.2
ip dhcp excluded-address 192.168.30.1 192.168.30.2
ip dhcp excluded-address 192.168.31.1 192.168.31.2
ip dhcp excluded-address 192.168.50.1 192.168.50.10
ip dhcp excluded-address 192.168.60.1 192.168.60.10
ip dhcp excluded-address 192.168.99.1 192.168.99.12
!
ip dhcp pool DOMAIN
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
!
ip dhcp pool CEH1
 network 192.168.20.0 255.255.255.240
 default-router 192.168.20.1
 dns-server 8.8.8.8
!
ip dhcp pool CEH2
 network 192.168.21.0 255.255.255.240
 default-router 192.168.21.1
 dns-server 8.8.8.8
!
ip dhcp pool CEH3
 network 192.168.22.0 255.255.255.240
 default-router 192.168.22.1
 dns-server 8.8.8.8
!
ip dhcp pool CEH4
 network 192.168.23.0 255.255.255.240
 default-router 192.168.23.1
 dns-server 8.8.8.8
!
ip dhcp pool TP
 network 192.168.30.0 255.255.255.240
 default-router 192.168.30.1
 dns-server 8.8.8.8
!
ip dhcp pool ATP
 network 192.168.31.0 255.255.255.240
 default-router 192.168.31.1
 dns-server 8.8.8.8
!
ip dhcp pool VIDEO
 network 192.168.50.0 255.255.255.0
 default-router 192.168.50.1
 dns-server 8.8.8.8
!
ip dhcp pool GUEST
 network 192.168.60.0 255.255.255.0
 default-router 192.168.60.1
 dns-server 8.8.8.8
!
ntp master 3
!
snmp-server community public RO
snmp-server location Server Room
snmp-server contact admin@example.com
!
lldp run
!
end
write memory
```

**Пояснение:** Core-SW — ядро сети. Выполняет маршрутизацию между VLAN, DHCP, NTP (мастер), SNMP. OSPF анонсирует все внутренние подсети. STP настроен так, что Core-SW является корневым мостом. Транк к SW-Optical-1 пропускает в том числе VLAN 60 (Guest).

### 5.4. SW-Optical-1 (Cisco 3650)

```
enable
configure terminal
hostname SW-Optical-1
no ip domain-lookup
!
enable secret class
service password-encryption
!
username admin secret AdminPass123
!
line console 0
 password cisco
 login
 logging synchronous
line vty 0 15
 login local
 transport input ssh
!
ip domain-name example.com
crypto key generate rsa modulus 1024
!
vlan 20
 name Ceh1
vlan 21
 name Ceh2
vlan 22
 name Ceh3
vlan 23
 name Ceh4
vlan 30
 name TP
vlan 31
 name ATP
vlan 50
 name Video
vlan 60
 name Guest
vlan 99
 name MGMT
!
interface vlan 99
 ip address 192.168.99.2 255.255.255.240
 no shutdown
ip default-gateway 192.168.99.1
!
interface GigabitEthernet1/1/1
 description To Core-SW (fiber)
 switchport mode trunk
 switchport trunk allowed vlan 20,21,22,23,30,31,50,60,99
!
interface GigabitEthernet1/1/2
 description To SW-Ceh1 (fiber)
 switchport mode trunk
 switchport trunk allowed vlan 20,50
!
interface GigabitEthernet1/1/3
 description To SW-Ceh2 (fiber)
 switchport mode trunk
 switchport trunk allowed vlan 21,50
!
interface GigabitEthernet1/1/4
 description To SW-Ceh3 (fiber)
 switchport mode trunk
 switchport trunk allowed vlan 22,50
!
interface range GigabitEthernet1/0/1-2
 channel-group 1 mode active
 switchport mode trunk
 switchport trunk allowed vlan 20,21,22,23,30,31,50,60,99
!
interface port-channel 1
 switchport mode trunk
 switchport trunk allowed vlan 20,21,22,23,30,31,50,60,99
!
spanning-tree mode rapid-pvst
!
end
write memory
```

**Пояснение:** SW-Optical-1 агрегирует оптические линки от цехов 1–3. EtherChannel (LACP) к SW-Optical-2 увеличивает пропускную способность и обеспечивает резервирование.

### 5.5. SW-Optical-2 (Cisco 3650)

```
enable
configure terminal
hostname SW-Optical-2
no ip domain-lookup
!
enable secret class
service password-encryption
!
username admin secret AdminPass123
!
line console 0
 password cisco
 login
 logging synchronous
line vty 0 15
 login local
 transport input ssh
!
ip domain-name example.com
crypto key generate rsa modulus 1024
!
vlan 20
 name Ceh1
vlan 21
 name Ceh2
vlan 22
 name Ceh3
vlan 23
 name Ceh4
vlan 30
 name TP
vlan 31
 name ATP
vlan 50
 name Video
vlan 60
 name Guest
vlan 99
 name MGMT
!
interface vlan 99
 ip address 192.168.99.3 255.255.255.240
 no shutdown
ip default-gateway 192.168.99.1
!
interface GigabitEthernet1/1/1
 description To SW-Ceh4 (fiber)
 switchport mode trunk
 switchport trunk allowed vlan 23,50
!
interface GigabitEthernet1/1/2
 description To SW-TP (fiber)
 switchport mode trunk
 switchport trunk allowed vlan 30,50
!
interface GigabitEthernet1/1/3
 description To SW-ATP (fiber)
 switchport mode trunk
 switchport trunk allowed vlan 31,50
!
interface range GigabitEthernet1/0/1-2
 channel-group 1 mode active
 switchport mode trunk
 switchport trunk allowed vlan 20,21,22,23,30,31,50,60,99
!
interface port-channel 1
 switchport mode trunk
 switchport trunk allowed vlan 20,21,22,23,30,31,50,60,99
!
spanning-tree mode rapid-pvst
!
end
write memory
```

**Пояснение:** SW-Optical-2 агрегирует линки от цеха 4 и проходных. Соединён с SW-Optical-1 через EtherChannel.

### 5.6. SW-Floor1 (Cisco 2960, VLAN 10 + Guest)

```
enable
configure terminal
hostname SW-Floor1
no ip domain-lookup
!
enable secret class
service password-encryption
!
username admin secret AdminPass123
!
line console 0
 password cisco
 login
 logging synchronous
line vty 0 15
 login local
 transport input ssh
!
ip domain-name example.com
crypto key generate rsa modulus 1024
!
vlan 10
 name DOMAIN
vlan 60
 name Guest
vlan 99
 name MGMT
!
interface vlan 99
 ip address 192.168.99.4 255.255.255.240
 no shutdown
ip default-gateway 192.168.99.1
!
interface GigabitEthernet0/1
 description To Core-SW
 switchport mode trunk
 switchport trunk allowed vlan 10,60,99
!
interface range FastEthernet0/1-20
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 no cdp enable
!
interface range FastEthernet0/21-24
 description Guest Wi-Fi AP
 switchport mode access
 switchport access vlan 60
 spanning-tree portfast
!
spanning-tree mode rapid-pvst
!
ntp server 192.168.99.1
lldp run
!
end
write memory
```

**Пояснение:** SW-Floor1 — коммутатор доступа 1-го этажа на Cisco 2960. Порты Fa0/1–20 — VLAN 10 (домен), Fa0/21–24 — VLAN 60 (Guest) для подключения гостевых точек доступа. Port Security ограничивает подключение одним MAC-адресом на порт. CDP отключён на портах доступа. Uplink — Gi0/1.

### 5.7. SW-Floor2 (Cisco 2960, VLAN 10 + Guest)

```
enable
configure terminal
hostname SW-Floor2
no ip domain-lookup
!
enable secret class
service password-encryption
!
username admin secret AdminPass123
!
line console 0
 password cisco
 login
 logging synchronous
line vty 0 15
 login local
 transport input ssh
!
ip domain-name example.com
crypto key generate rsa modulus 1024
!
vlan 10
 name DOMAIN
vlan 60
 name Guest
vlan 99
 name MGMT
!
interface vlan 99
 ip address 192.168.99.5 255.255.255.240
 no shutdown
ip default-gateway 192.168.99.1
!
interface GigabitEthernet0/1
 description To Core-SW
 switchport mode trunk
 switchport trunk allowed vlan 10,60,99
!
interface range FastEthernet0/1-20
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 no cdp enable
!
interface range FastEthernet0/21-24
 description Guest Wi-Fi AP
 switchport mode access
 switchport access vlan 60
 spanning-tree portfast
!
spanning-tree mode rapid-pvst
!
ntp server 192.168.99.1
lldp run
!
end
write memory
```

### 5.8. SW-Ceh1 (Cisco 3650, VLAN 20)

```
enable
configure terminal
hostname SW-Ceh1
no ip domain-lookup
!
enable secret class
service password-encryption
!
username admin secret AdminPass123
!
line console 0
 password cisco
 login
 logging synchronous
line vty 0 15
 login local
 transport input ssh
!
ip domain-name example.com
crypto key generate rsa modulus 1024
!
vlan 20
 name Ceh1
vlan 50
 name Video
vlan 99
 name MGMT
!
interface vlan 99
 ip address 192.168.99.6 255.255.255.240
 no shutdown
ip default-gateway 192.168.99.1
!
interface GigabitEthernet1/1/1
 description To SW-Optical-1 (fiber)
 switchport mode trunk
 switchport trunk allowed vlan 20,50,99
!
interface range GigabitEthernet1/0/1-10
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
!
interface range GigabitEthernet1/0/11-20
 switchport mode access
 switchport access vlan 50
 spanning-tree portfast
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
!
spanning-tree mode rapid-pvst
!
ntp server 192.168.99.1
lldp run
!
end
write memory
```

**Пояснение:** SW-Ceh1 — коммутатор цеха №1. Порты 1–10 — промышленные ПК (VLAN 20), порты 11–20 — камеры (VLAN 50, PoE). Транк к SW-Optical-1 по оптике.

### 5.9. SW-Ceh2 (Cisco 3650, VLAN 21)

```
enable
configure terminal
hostname SW-Ceh2
no ip domain-lookup
!
enable secret class
service password-encryption
!
username admin secret AdminPass123
!
line console 0
 password cisco
 login
 logging synchronous
line vty 0 15
 login local
 transport input ssh
!
ip domain-name example.com
crypto key generate rsa modulus 1024
!
vlan 21
 name Ceh2
vlan 50
 name Video
vlan 99
 name MGMT
!
interface vlan 99
 ip address 192.168.99.7 255.255.255.240
 no shutdown
ip default-gateway 192.168.99.1
!
interface GigabitEthernet1/1/1
 description To SW-Optical-1 (fiber)
 switchport mode trunk
 switchport trunk allowed vlan 21,50,99
!
interface range GigabitEthernet1/0/1-10
 switchport mode access
 switchport access vlan 21
 spanning-tree portfast
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
!
interface range GigabitEthernet1/0/11-20
 switchport mode access
 switchport access vlan 50
 spanning-tree portfast
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
!
spanning-tree mode rapid-pvst
!
ntp server 192.168.99.1
lldp run
!
end
write memory
```

### 5.10. SW-Ceh3 (Cisco 3650, VLAN 22)

```
enable
configure terminal
hostname SW-Ceh3
no ip domain-lookup
!
enable secret class
service password-encryption
!
username admin secret AdminPass123
!
line console 0
 password cisco
 login
 logging synchronous
line vty 0 15
 login local
 transport input ssh
!
ip domain-name example.com
crypto key generate rsa modulus 1024
!
vlan 22
 name Ceh3
vlan 50
 name Video
vlan 99
 name MGMT
!
interface vlan 99
 ip address 192.168.99.8 255.255.255.240
 no shutdown
ip default-gateway 192.168.99.1
!
interface GigabitEthernet1/1/1
 description To SW-Optical-1 (fiber)
 switchport mode trunk
 switchport trunk allowed vlan 22,50,99
!
interface range GigabitEthernet1/0/1-10
 switchport mode access
 switchport access vlan 22
 spanning-tree portfast
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
!
interface range GigabitEthernet1/0/11-20
 switchport mode access
 switchport access vlan 50
 spanning-tree portfast
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
!
spanning-tree mode rapid-pvst
!
ntp server 192.168.99.1
lldp run
!
end
write memory
```

### 5.11. SW-Ceh4 (Cisco 3650, VLAN 23)

```
enable
configure terminal
hostname SW-Ceh4
no ip domain-lookup
!
enable secret class
service password-encryption
!
username admin secret AdminPass123
!
line console 0
 password cisco
 login
 logging synchronous
line vty 0 15
 login local
 transport input ssh
!
ip domain-name example.com
crypto key generate rsa modulus 1024
!
vlan 23
 name Ceh4
vlan 50
 name Video
vlan 99
 name MGMT
!
interface vlan 99
 ip address 192.168.99.9 255.255.255.240
 no shutdown
ip default-gateway 192.168.99.1
!
interface GigabitEthernet1/1/1
 description To SW-Optical-2 (fiber)
 switchport mode trunk
 switchport trunk allowed vlan 23,50,99
!
interface range GigabitEthernet1/0/1-10
 switchport mode access
 switchport access vlan 23
 spanning-tree portfast
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
!
interface range GigabitEthernet1/0/11-20
 switchport mode access
 switchport access vlan 50
 spanning-tree portfast
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
!
spanning-tree mode rapid-pvst
!
ntp server 192.168.99.1
lldp run
!
end
write memory
```

### 5.12. SW-TP (Cisco 3650, VLAN 30)

```
enable
configure terminal
hostname SW-TP
no ip domain-lookup
!
enable secret class
service password-encryption
!
username admin secret AdminPass123
!
line console 0
 password cisco
 login
 logging synchronous
line vty 0 15
 login local
 transport input ssh
!
ip domain-name example.com
crypto key generate rsa modulus 1024
!
vlan 30
 name TP
vlan 50
 name Video
vlan 99
 name MGMT
!
interface vlan 99
 ip address 192.168.99.10 255.255.255.240
 no shutdown
ip default-gateway 192.168.99.1
!
interface GigabitEthernet1/1/1
 description To SW-Optical-2 (fiber)
 switchport mode trunk
 switchport trunk allowed vlan 30,50,99
!
interface range GigabitEthernet1/0/1-10
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
!
interface range GigabitEthernet1/0/11-20
 switchport mode access
 switchport access vlan 50
 spanning-tree portfast
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
!
spanning-tree mode rapid-pvst
!
ntp server 192.168.99.1
lldp run
!
end
write memory
```

### 5.13. SW-ATP (Cisco 3650, VLAN 31)

```
enable
configure terminal
hostname SW-ATP
no ip domain-lookup
!
enable secret class
service password-encryption
!
username admin secret AdminPass123
!
line console 0
 password cisco
 login
 logging synchronous
line vty 0 15
 login local
 transport input ssh
!
ip domain-name example.com
crypto key generate rsa modulus 1024
!
vlan 31
 name ATP
vlan 50
 name Video
vlan 99
 name MGMT
!
interface vlan 99
 ip address 192.168.99.11 255.255.255.240
 no shutdown
ip default-gateway 192.168.99.1
!
interface GigabitEthernet1/1/1
 description To SW-Optical-2 (fiber)
 switchport mode trunk
 switchport trunk allowed vlan 31,50,99
!
interface range GigabitEthernet1/0/1-10
 switchport mode access
 switchport access vlan 31
 spanning-tree portfast
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
!
interface range GigabitEthernet1/0/11-20
 switchport mode access
 switchport access vlan 50
 spanning-tree portfast
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
!
spanning-tree mode rapid-pvst
!
ntp server 192.168.99.1
lldp run
!
end
write memory
```

### 5.14. Настройка конечных устройств (DHCP-клиенты)

В Cisco Packet Tracer по умолчанию сетевые адаптеры ПК находятся в режиме **Static**. Чтобы они получали адреса автоматически, необходимо на каждом ПК выставляем DHCP в настройках сетевой карты.

---

## 6. Изоляция VLAN (стандартные ACL)

Все ACL применяются **outbound** на SVI соответствующих VLAN на Core-SW.

### 6.1. Изоляция цехов

**ACL 20 (VLAN 20 — Ceh1):**

```
access-list 20 deny 192.168.21.0 0.0.0.15
access-list 20 deny 192.168.22.0 0.0.0.15
access-list 20 deny 192.168.23.0 0.0.0.15
access-list 20 deny 192.168.30.0 0.0.0.15
access-list 20 deny 192.168.31.0 0.0.0.15
access-list 20 deny 192.168.60.0 0.0.0.255
access-list 20 permit any
```

**ACL 21 (VLAN 21 — Ceh2):**

```
access-list 21 deny 192.168.20.0 0.0.0.15
access-list 21 deny 192.168.22.0 0.0.0.15
access-list 21 deny 192.168.23.0 0.0.0.15
access-list 21 deny 192.168.30.0 0.0.0.15
access-list 21 deny 192.168.31.0 0.0.0.15
access-list 21 deny 192.168.60.0 0.0.0.255
access-list 21 permit any
```

**ACL 22 (VLAN 22 — Ceh3):**

```
access-list 22 deny 192.168.20.0 0.0.0.15
access-list 22 deny 192.168.21.0 0.0.0.15
access-list 22 deny 192.168.23.0 0.0.0.15
access-list 22 deny 192.168.30.0 0.0.0.15
access-list 22 deny 192.168.31.0 0.0.0.15
access-list 22 deny 192.168.60.0 0.0.0.255
access-list 22 permit any
```

**ACL 23 (VLAN 23 — Ceh4):**

```
access-list 23 deny 192.168.20.0 0.0.0.15
access-list 23 deny 192.168.21.0 0.0.0.15
access-list 23 deny 192.168.22.0 0.0.0.15
access-list 23 deny 192.168.30.0 0.0.0.15
access-list 23 deny 192.168.31.0 0.0.0.15
access-list 23 deny 192.168.60.0 0.0.0.255
access-list 23 permit any
```

**Применение:**

```
interface vlan 20
 ip access-group 20 out
interface vlan 21
 ip access-group 21 out
interface vlan 22
 ip access-group 22 out
interface vlan 23
 ip access-group 23 out
```

### 6.2. Изоляция проходных

**ACL 30 (VLAN 30 — TP):**

```
access-list 30 deny 192.168.10.0 0.0.0.255
access-list 30 deny 192.168.20.0 0.0.0.15
access-list 30 deny 192.168.21.0 0.0.0.15
access-list 30 deny 192.168.22.0 0.0.0.15
access-list 30 deny 192.168.23.0 0.0.0.15
access-list 30 deny 192.168.31.0 0.0.0.15
access-list 30 deny 192.168.60.0 0.0.0.255
access-list 30 permit any
```

**ACL 31 (VLAN 31 — ATP):**

```
access-list 31 deny 192.168.10.0 0.0.0.255
access-list 31 deny 192.168.20.0 0.0.0.15
access-list 31 deny 192.168.21.0 0.0.0.15
access-list 31 deny 192.168.22.0 0.0.0.15
access-list 31 deny 192.168.23.0 0.0.0.15
access-list 31 deny 192.168.30.0 0.0.0.15
access-list 31 deny 192.168.60.0 0.0.0.255
access-list 31 permit any
```

**Применение:**

```
interface vlan 30
 ip access-group 30 out
interface vlan 31
 ip access-group 31 out
```

### 6.3. Доступ к видеонаблюдению (только из VLAN 10)

**ACL 50 (VLAN 50 — Video):**

```
access-list 50 deny 192.168.20.0 0.0.0.15
access-list 50 deny 192.168.21.0 0.0.0.15
access-list 50 deny 192.168.22.0 0.0.0.15
access-list 50 deny 192.168.23.0 0.0.0.15
access-list 50 deny 192.168.30.0 0.0.0.15
access-list 50 deny 192.168.31.0 0.0.0.15
access-list 50 deny 192.168.60.0 0.0.0.255
access-list 50 deny 192.168.99.0 0.0.0.15
access-list 50 permit any
```

**Применение:**

```
interface vlan 50
 ip access-group 50 out
```

### 6.4. Изоляция гостевого VLAN (Guest)

**ACL 10 (VLAN 10 — DOMAIN):**

```
access-list 10 deny 192.168.60.0 0.0.0.255
access-list 10 permit any
```

**ACL 99 (VLAN 99 — MGMT):**

```
access-list 99 deny 192.168.60.0 0.0.0.255
access-list 99 permit any
```

**Применение:**

```
interface vlan 10
 ip access-group 10 out
interface vlan 99
 ip access-group 99 out
```

**Пояснение:** Все ACL применяются outbound на SVI. Трафик Guest, направленный в любую внутреннюю подсеть, блокируется на выходе в соответствующий VLAN. Трафик Guest в Интернет идёт через gi0/1 к Edge-Router, минуя внутренние SVI, поэтому NAT и выход в глобальную сеть работают без ограничений.

---

## 7. Таблица магистральных линков (Trunk)

| № | Устройство A | Порт A | Устройство B | Порт B | Тип линка | Разрешённые VLAN |
|---|--------------|--------|--------------|--------|-----------|------------------|
| 1 | Core-SW | Gi1/0/2 | SW-Floor1 | Gi0/1 | Медь | 10, 60, 99 |
| 2 | Core-SW | Gi1/0/3 | SW-Floor2 | Gi0/1 | Медь | 10, 60, 99 |
| 3 | Core-SW | Gi1/1/1 | SW-Optical-1 | Gi1/1/1 | Оптика | 20,21,22,23,30,31,50,60,99 |
| 4 | SW-Optical-1 | Gi1/1/2 | SW-Ceh1 | Gi1/1/1 | Оптика | 20, 50, 99 |
| 5 | SW-Optical-1 | Gi1/1/3 | SW-Ceh2 | Gi1/1/1 | Оптика | 21, 50, 99 |
| 6 | SW-Optical-1 | Gi1/1/4 | SW-Ceh3 | Gi1/1/1 | Оптика | 22, 50, 99 |
| 7 | SW-Optical-2 | Gi1/1/1 | SW-Ceh4 | Gi1/1/1 | Оптика | 23, 50, 99 |
| 8 | SW-Optical-2 | Gi1/1/2 | SW-TP | Gi1/1/1 | Оптика | 30, 50, 99 |
| 9 | SW-Optical-2 | Gi1/1/3 | SW-ATP | Gi1/1/1 | Оптика | 31, 50, 99 |
| 10 | SW-Optical-1 | Gi1/0/1-2 (Po1) | SW-Optical-2 | Gi1/0/1-2 (Po1) | Медь (EtherChannel) | 20,21,22,23,30,31,50,60,99 |

> **Примечание:** Для EtherChannel между SW-Optical-1 и SW-Optical-2 используется LACP (mode active), логический интерфейс Port-channel 1. VLAN-политика задана как на физических портах, так и на port-channel.

---

## 8. Проверка работоспособности

### 8.1. На Core-SW

```
show ip route
show ip ospf neighbor
show ip dhcp binding
show access-lists
show vlan brief
show interfaces trunk
```

### 8.2. На коммутаторах

```
show vlan brief
show interfaces trunk
show spanning-tree
show etherchannel summary
show port-security
```

### 8.3. С конечных устройств

| Проверка | Команда | Ожидаемый результат |
|----------|---------|---------------------|
| DHCP | `ipconfig` | Адрес из нужной подсети |
| Связь внутри VLAN | `ping <шлюз>` | Успешно |
| Изоляция цехов | `ping 192.168.21.1` с PC в VLAN 20 | **Недоступно** |
| Изоляция проходных | `ping 192.168.10.1` с PC в VLAN 30 | **Недоступно** |
| Video из домена | `ping 192.168.50.1` с PC в VLAN 10 | Успешно |
| Video из цеха | `ping 192.168.50.1` с PC в VLAN 20 | **Недоступно** |
| Guest к внутренним | `ping 192.168.10.1` с PC в VLAN 60 | **Недоступно** |
| Guest к Интернету | `ping 8.8.8.8` с PC в VLAN 60 | Успешно |
| Интернет из домена | `ping 8.8.8.8` с PC в VLAN 10 | Успешно |
| SSH | `ssh -l admin 192.168.99.1` | Успешный вход |

---

## 9. Примечания и ограничения

1. **48-портовые коммутаторы:** В Cisco Packet Tracer 9.0 нет 48-портовых моделей. Использованы 24-портовые модели. В реальной сети применяются **Cisco 2960-48PST-L** (этажи, цеха) и **Cisco 3560-24PS** (проходные).

2. **PoE на этажах:** В CPT 9.0 коммутаторы 2960 не поддерживают PoE. Для гостевых Wi-Fi точек на этажах в реальной сети используются внешние PoE-инжекторы либо коммутаторы с PoE (2960-48PST-L).

3. **PoE в цехах и на проходных:** В CPT 9.0 используется **3650-24PS** для питания камер. В реальной сети — **2960-48PST-L** (цеха) и **3560-24PS** (проходные) с PoE.

4. **Оптические порты:** В CPT только **3650-24PS** имеет SFP-порты (4 шт.). Для 7 оптических линков использованы два коммутатора: **SW-Optical-1** и **SW-Optical-2**, соединённые EtherChannel.

5. **Standard ACL:** Использованы стандартные ACL (1–99). Они фильтруют только по источнику и размещаются близко к назначению. Для более точного контроля в реальной сети применяются расширенные ACL.

6. **FHRP:** Не используется, так как в проекте одно ядро. При добавлении второго ядра можно настроить HSRP/VRRP.

7. **IPv6 и WLAN:** Не включены в базовый проект. Могут быть добавлены как расширение (SLAAC, DHCPv6, WLC + точки доступа).

---

## 10. Выводы

Проект реализует:
- Иерархическую топологию с ядром на L3-коммутаторе Cisco 3650.
- Сегментацию на 10 VLAN: домен, 4 цеха, 2 проходные, Video, Guest, управление.
- Изоляцию промышленных сетей и проходных через стандартные ACL.
- Доступ к видеонаблюдению только из доменной сети.
- Полную изоляцию гостевой сети от внутренних ресурсов с сохранением выхода в Интернет.
- Автоматическую выдачу адресов через DHCPv4.
- Выход в Интернет через NAT (PAT).
- Управление через NTP, SNMP, CDP/LLDP, SSH.
- Отказоустойчивость через STP (Rapid-PVST+) и EtherChannel (LACP).
- Масштабируемость за счёт L3-ядра и модульной архитектуры.
- Экономию бюджета за счёт использования Cisco 2960-24TT на этажах.
