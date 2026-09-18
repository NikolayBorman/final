# Проектная работа: Построение ЛВС деревообрабатывающего предприятия

## 1. Цель и задачи

**Цель:** Спроектировать и настроить масштабируемую, отказоустойчивую и защищённую ЛВС деревообрабатывающего предприятия.

**Задачи:**
1. Спроектировать двухуровневую модель сети (collapsed core).
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

> **Примечание:** В Cisco Packet Tracer 9.0 коммутаторы 2960 не поддерживают PoE и SFP. Поэтому коммутаторы в здании управления реализованы на 2960-24TT. Коммутаторы цехов и проходных — на 3650-24PS.

---

## 3. Топология

<img width="1058" height="833" alt="image" src="https://github.com/user-attachments/assets/cf33ca0d-7bae-484c-9835-c3ed4000a594" />

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

**WAN-линки:**
- ISP ↔ Edge: `10.0.0.0/30` (ISP: .1, Edge: .2)
- Edge ↔ Core-SW: `10.0.1.0/30` (Edge: .1, Core: .2)

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

```cisco
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

```cisco
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
ip domain-name dok.local
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
 router-id 1.1.1.1
 passive-interface default
 no passive-interface GigabitEthernet0/1
 network 10.0.0.0 0.0.0.3 area 0
 network 10.0.1.0 0.0.0.3 area 0
 default-information originate
!
end
write memory
```

**Пояснение:** Edge-Router выполняет NAT для выхода в Интернет. OSPF настроен для обмена маршрутами с Core-SW. ACL 1 разрешает трансляцию для всех внутренних сетей 192.168.x.x.

### 5.3. Core-SW (Cisco 3650, L3)

```cisco
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
ip domain-name dok.local
crypto key generate rsa general-keys modulus 1024
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
router ospf 1
 router-id 2.2.2.2
 passive-interface default
 no passive-interface GigabitEthernet1/0/1
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
!
lldp run
!
end
write memory
```

**Пояснение:** Core-SW — ядро сети. Выполняет маршрутизацию между VLAN, DHCP, NTP (мастер), SNMP . OSPF анонсирует все внутренние подсети. Core-SW является корневым мостом.

### 5.4. SW-Optical-1 (Cisco 3650)

```cisco
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
ip domain-name dok.local
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
 switchport trunk allowed vlan 20,50,99
!
interface GigabitEthernet1/1/3
 description To SW-Ceh2 (fiber)
 switchport mode trunk
 switchport trunk allowed vlan 21,50,99
!
interface GigabitEthernet1/1/4
 description To SW-Ceh3 (fiber)
 switchport mode trunk
 switchport trunk allowed vlan 22,50,99
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

**Пояснение:** SW-Optical-1 агрегирует оптические линки от цехов 1,2,3. EtherChannel (LACP) к SW-Optical-2 увеличивает пропускную способность и обеспечивает резервирование.

### 5.5. SW-Optical-2 (Cisco 3650)

```cisco
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
ip domain-name dok.local
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
 switchport trunk allowed vlan 23,50,99
!
interface GigabitEthernet1/1/2
 description To SW-TP (fiber)
 switchport mode trunk
 switchport trunk allowed vlan 30,50,99
!
interface GigabitEthernet1/1/3
 description To SW-ATP (fiber)
 switchport mode trunk
 switchport trunk allowed vlan 31,50,99
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

**Пояснение:** SW-Optical-2 агрегирует линки от цеха 4,табельной и автотранспортой проходных. Соединён с SW-Optical-1 через EtherChannel по объединенным портам GigabitEthernet1/0/1-2.

### 5.6. SW-Floor1 (Cisco 2960, VLAN 10 + Guest)

```cisco
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
ip domain-name dok.local
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

### 5.7. SW-Floor2 (Cisco 2960, VLAN 10 + Guest)

```cisco
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
ip domain-name dok.local
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

```cisco
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
ip domain-name dok.local
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

### 5.9. SW-Ceh2 (Cisco 3650, VLAN 21)

```cisco
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
ip domain-name dok.local
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

```cisco
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
ip domain-name dok.local
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

```cisco
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
ip domain-name dok.local
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

```cisco
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
ip domain-name dok.local
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

```cisco
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
ip domain-name dok.local
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

В Cisco Packet Tracer по умолчанию сетевые адаптеры ПК находятся в режиме **Static**. Чтобы конечные устройства получали ip от нашего DHCP-сервера, Ставим в настройках **DHCP**.
После этого проверяем получение адреса на ПК
```cmd
C:\>ipconfig

FastEthernet0 Connection:(default port)

   Connection-specific DNS Suffix..: 
   Link-local IPv6 Address.........: FE80::20C:85FF:FE67:26DE
   IPv6 Address....................: ::
   IPv4 Address....................: 192.168.20.3
   Subnet Mask.....................: 255.255.255.240
   Default Gateway.................: ::
                                     192.168.20.1

Bluetooth Connection:

   Connection-specific DNS Suffix..: 
   Link-local IPv6 Address.........: ::
   IPv6 Address....................: ::
   IPv4 Address....................: 0.0.0.0
   Subnet Mask.....................: 0.0.0.0
   Default Gateway.................: ::
                                     0.0.0.0
```

## 6. Изоляция VLAN (стандартные ACL)

Все ACL применяются **outbound** на SVI соответствующих VLAN на Core-SW.

### 6.1. ACL 10 — DOMAIN (VLAN 10)

```cisco
access-list 10 deny 192.168.20.0 0.0.0.15
access-list 10 deny 192.168.21.0 0.0.0.15
access-list 10 deny 192.168.22.0 0.0.0.15
access-list 10 deny 192.168.23.0 0.0.0.15
access-list 10 deny 192.168.30.0 0.0.0.15
access-list 10 deny 192.168.31.0 0.0.0.15
access-list 10 deny 192.168.60.0 0.0.0.255
access-list 10 permit any
```

### 6.2. ACL 20 — Ceh1 (VLAN 20)

```cisco
access-list 20 deny 192.168.10.0 0.0.0.255
access-list 20 deny 192.168.21.0 0.0.0.15
access-list 20 deny 192.168.22.0 0.0.0.15
access-list 20 deny 192.168.23.0 0.0.0.15
access-list 20 deny 192.168.30.0 0.0.0.15
access-list 20 deny 192.168.31.0 0.0.0.15
access-list 20 deny 192.168.50.0 0.0.0.255
access-list 20 deny 192.168.60.0 0.0.0.255
access-list 20 deny 192.168.99.0 0.0.0.15
access-list 20 permit any
```

### 6.3. ACL 21 — Ceh2 (VLAN 21)

```cisco
access-list 21 deny 192.168.10.0 0.0.0.255
access-list 21 deny 192.168.20.0 0.0.0.15
access-list 21 deny 192.168.22.0 0.0.0.15
access-list 21 deny 192.168.23.0 0.0.0.15
access-list 21 deny 192.168.30.0 0.0.0.15
access-list 21 deny 192.168.31.0 0.0.0.15
access-list 21 deny 192.168.50.0 0.0.0.255
access-list 21 deny 192.168.60.0 0.0.0.255
access-list 21 deny 192.168.99.0 0.0.0.15
access-list 21 permit any
```

### 6.4. ACL 22 — Ceh3 (VLAN 22)

```cisco
access-list 22 deny 192.168.10.0 0.0.0.255
access-list 22 deny 192.168.20.0 0.0.0.15
access-list 22 deny 192.168.21.0 0.0.0.15
access-list 22 deny 192.168.23.0 0.0.0.15
access-list 22 deny 192.168.30.0 0.0.0.15
access-list 22 deny 192.168.31.0 0.0.0.15
access-list 22 deny 192.168.50.0 0.0.0.255
access-list 22 deny 192.168.60.0 0.0.0.255
access-list 22 deny 192.168.99.0 0.0.0.15
access-list 22 permit any
```

### 6.5. ACL 23 — Ceh4 (VLAN 23)

```cisco
access-list 23 deny 192.168.10.0 0.0.0.255
access-list 23 deny 192.168.20.0 0.0.0.15
access-list 23 deny 192.168.21.0 0.0.0.15
access-list 23 deny 192.168.22.0 0.0.0.15
access-list 23 deny 192.168.30.0 0.0.0.15
access-list 23 deny 192.168.31.0 0.0.0.15
access-list 23 deny 192.168.50.0 0.0.0.255
access-list 23 deny 192.168.60.0 0.0.0.255
access-list 23 deny 192.168.99.0 0.0.0.15
access-list 23 permit any
```

### 6.6. ACL 30 — TP (VLAN 30)

```cisco
access-list 30 deny 192.168.10.0 0.0.0.255
access-list 30 deny 192.168.20.0 0.0.0.15
access-list 30 deny 192.168.21.0 0.0.0.15
access-list 30 deny 192.168.22.0 0.0.0.15
access-list 30 deny 192.168.23.0 0.0.0.15
access-list 30 deny 192.168.31.0 0.0.0.15
access-list 30 deny 192.168.50.0 0.0.0.255
access-list 30 deny 192.168.60.0 0.0.0.255
access-list 30 deny 192.168.99.0 0.0.0.15
access-list 30 permit any
```

### 6.7. ACL 31 — ATP (VLAN 31)

```cisco
access-list 31 deny 192.168.10.0 0.0.0.255
access-list 31 deny 192.168.20.0 0.0.0.15
access-list 31 deny 192.168.21.0 0.0.0.15
access-list 31 deny 192.168.22.0 0.0.0.15
access-list 31 deny 192.168.23.0 0.0.0.15
access-list 31 deny 192.168.30.0 0.0.0.15
access-list 31 deny 192.168.50.0 0.0.0.255
access-list 31 deny 192.168.60.0 0.0.0.255
access-list 31 deny 192.168.99.0 0.0.0.15
access-list 31 permit any
```

### 6.8. ACL 50 — Video (VLAN 50)

```cisco
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

### 6.9. ACL 60 — Guest (VLAN 60)

```cisco
access-list 60 deny 192.168.10.0 0.0.0.255
access-list 60 deny 192.168.20.0 0.0.0.15
access-list 60 deny 192.168.21.0 0.0.0.15
access-list 60 deny 192.168.22.0 0.0.0.15
access-list 60 deny 192.168.23.0 0.0.0.15
access-list 60 deny 192.168.30.0 0.0.0.15
access-list 60 deny 192.168.31.0 0.0.0.15
access-list 60 deny 192.168.50.0 0.0.0.255
access-list 60 deny 192.168.99.0 0.0.0.15
access-list 60 permit any
```

### 6.10. ACL 99 — MGMT (VLAN 99)

```cisco
access-list 99 deny 192.168.20.0 0.0.0.15
access-list 99 deny 192.168.21.0 0.0.0.15
access-list 99 deny 192.168.22.0 0.0.0.15
access-list 99 deny 192.168.23.0 0.0.0.15
access-list 99 deny 192.168.30.0 0.0.0.15
access-list 99 deny 192.168.31.0 0.0.0.15
access-list 99 deny 192.168.60.0 0.0.0.255
access-list 99 permit any
```

### 6.11. Применение ACL ко всем SVI

```cisco
interface vlan 10
 ip access-group 10 out
interface vlan 20
 ip access-group 20 out
interface vlan 21
 ip access-group 21 out
interface vlan 22
 ip access-group 22 out
interface vlan 23
 ip access-group 23 out
interface vlan 30
 ip access-group 30 out
interface vlan 31
 ip access-group 31 out
interface vlan 50
 ip access-group 50 out
interface vlan 60
 ip access-group 60 out
interface vlan 99
 ip access-group 99 out
exit
write memory
```
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

---

## 8. Проверка работоспособности

### 8.1. На Core-SW

**Маршрутизация:**
```cisco
Core-SW#show ip route
Core-SW#show ip route
Codes: C - connected, S - static, I - IGRP, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, E - EGP
       i - IS-IS, L1 - IS-IS level-1, L2 - IS-IS level-2, ia - IS-IS inter area
       * - candidate default, U - per-user static route, o - ODR
       P - periodic downloaded static route

Gateway of last resort is 10.0.1.1 to network 0.0.0.0

     10.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
O       10.0.0.0/30 [110/2] via 10.0.1.1, 00:11:10, GigabitEthernet1/0/1
C       10.0.1.0/30 is directly connected, GigabitEthernet1/0/1
L       10.0.1.2/32 is directly connected, GigabitEthernet1/0/1
     192.168.10.0/24 is variably subnetted, 2 subnets, 2 masks
C       192.168.10.0/24 is directly connected, Vlan10
L       192.168.10.1/32 is directly connected, Vlan10
     192.168.20.0/24 is variably subnetted, 2 subnets, 2 masks
C       192.168.20.0/28 is directly connected, Vlan20
L       192.168.20.1/32 is directly connected, Vlan20
     192.168.21.0/24 is variably subnetted, 2 subnets, 2 masks
C       192.168.21.0/28 is directly connected, Vlan21
L       192.168.21.1/32 is directly connected, Vlan21
     192.168.22.0/24 is variably subnetted, 2 subnets, 2 masks
C       192.168.22.0/28 is directly connected, Vlan22
L       192.168.22.1/32 is directly connected, Vlan22
     192.168.23.0/24 is variably subnetted, 2 subnets, 2 masks
C       192.168.23.0/28 is directly connected, Vlan23
L       192.168.23.1/32 is directly connected, Vlan23
     192.168.30.0/24 is variably subnetted, 2 subnets, 2 masks
C       192.168.30.0/28 is directly connected, Vlan30
L       192.168.30.1/32 is directly connected, Vlan30
     192.168.31.0/24 is variably subnetted, 2 subnets, 2 masks
C       192.168.31.0/28 is directly connected, Vlan31
L       192.168.31.1/32 is directly connected, Vlan31
     192.168.50.0/24 is variably subnetted, 2 subnets, 2 masks
C       192.168.50.0/24 is directly connected, Vlan50
L       192.168.50.1/32 is directly connected, Vlan50
     192.168.60.0/24 is variably subnetted, 2 subnets, 2 masks
C       192.168.60.0/24 is directly connected, Vlan60
L       192.168.60.1/32 is directly connected, Vlan60
     192.168.99.0/24 is variably subnetted, 2 subnets, 2 masks
C       192.168.99.0/28 is directly connected, Vlan99
L       192.168.99.1/32 is directly connected, Vlan99
S*   0.0.0.0/0 [1/0] via 10.0.1.1
```

**ACL**
```cisco
Core-SW#show access-lists
Standard IP access list 10
    10 deny 192.168.20.0 0.0.0.15
    20 deny 192.168.21.0 0.0.0.15
    30 deny 192.168.22.0 0.0.0.15
    40 deny 192.168.23.0 0.0.0.15
    50 deny 192.168.30.0 0.0.0.15
    60 deny 192.168.31.0 0.0.0.15
    70 deny 192.168.60.0 0.0.0.255
    80 permit any
Standard IP access list 20
    10 deny 192.168.10.0 0.0.0.255
    20 deny 192.168.21.0 0.0.0.15
    30 deny 192.168.22.0 0.0.0.15
    40 deny 192.168.23.0 0.0.0.15
    50 deny 192.168.30.0 0.0.0.15
    60 deny 192.168.31.0 0.0.0.15
    70 deny 192.168.50.0 0.0.0.255
    80 deny 192.168.60.0 0.0.0.255
    90 deny 192.168.99.0 0.0.0.15
   100 permit any
Standard IP access list 50
    10 deny 192.168.20.0 0.0.0.15
    20 deny 192.168.21.0 0.0.0.15
    30 deny 192.168.22.0 0.0.0.15
    40 deny 192.168.23.0 0.0.0.15
    50 deny 192.168.30.0 0.0.0.15
    60 deny 192.168.31.0 0.0.0.15
    70 deny 192.168.60.0 0.0.0.255
    80 deny 192.168.99.0 0.0.0.15
    90 permit any
Standard IP access list 60
    10 deny 192.168.10.0 0.0.0.255
    20 deny 192.168.20.0 0.0.0.15
    30 deny 192.168.21.0 0.0.0.15
    40 deny 192.168.22.0 0.0.0.15
    50 deny 192.168.23.0 0.0.0.15
    60 deny 192.168.30.0 0.0.0.15
    70 deny 192.168.31.0 0.0.0.15
    80 deny 192.168.50.0 0.0.0.255
    90 deny 192.168.99.0 0.0.0.15
   100 permit any
```

**VLAN:**
```cisco
Core-SW#show vlan brief 

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Gig1/0/4, Gig1/0/5, Gig1/0/6, Gig1/0/7
                                                Gig1/0/8, Gig1/0/9, Gig1/0/10, Gig1/0/11
                                                Gig1/0/12, Gig1/0/13, Gig1/0/14, Gig1/0/15
                                                Gig1/0/16, Gig1/0/17, Gig1/0/18, Gig1/0/19
                                                Gig1/0/20, Gig1/0/21, Gig1/0/22, Gig1/0/23
                                                Gig1/0/24, Gig1/1/2, Gig1/1/3, Gig1/1/4
10   DOMAIN                           active    
20   Ceh1                             active    
21   Ceh2                             active    
22   Ceh3                             active    
23   Ceh4                             active    
30   TP                               active    
31   ATP                              active    
50   Video                            active    
60   Guest                            active    
99   MGMT                             active    
1002 fddi-default                     active    
1003 token-ring-default               active    
1004 fddinet-default                  active    
1005 trnet-default                    active    
```

**Транки**
```cisco
Core-SW#show interfaces trunk
Core-SW#show interfaces trunk 
Port        Mode         Encapsulation  Status        Native vlan
Gig1/0/2    on           802.1q         trunking      1
Gig1/0/3    on           802.1q         trunking      1
Gig1/1/1    on           802.1q         trunking      1

Port        Vlans allowed on trunk
Gig1/0/2    10,60,99
Gig1/0/3    10,60,99
Gig1/1/1    20-23,30-31,50,60,99

Port        Vlans allowed and active in management domain
Gig1/0/2    10,60,99
Gig1/0/3    10,60,99
Gig1/1/1    20,21,22,23,30,31,50,60,99

Port        Vlans in spanning tree forwarding state and not pruned
Gig1/0/2    10,60,99
Gig1/0/3    10,60,99
Gig1/1/1    20,21,22,23,30,31,50,60,99
```

### 8.2. На коммутаторах

**VLAN на SW-Floor1:**
```cisco
SW-Floor1#show vlan brief 

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Gig0/2
10   DOMAIN                           active    Fa0/1, Fa0/2, Fa0/3, Fa0/4
                                                Fa0/5, Fa0/6, Fa0/7, Fa0/8
                                                Fa0/9, Fa0/10, Fa0/11, Fa0/12
                                                Fa0/13, Fa0/14, Fa0/15, Fa0/16
                                                Fa0/17, Fa0/18, Fa0/19, Fa0/20
60   Guest                            active    Fa0/21, Fa0/22, Fa0/23, Fa0/24
99   MGMT                             active    
1002 fddi-default                     active    
1003 token-ring-default               active    
1004 fddinet-default                  active    
1005 trnet-default                    active   
```

**STP на SW-Ceh2:**
```cisco
W-Ceh2#show spanning-tree 
VLAN0001
  Spanning tree enabled protocol rstp
  Root ID    Priority    24577
             Address     0005.5EAB.BA60
             Cost        8
             Port        25(GigabitEthernet1/1/1)
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
             Address     00E0.A343.7BCE
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Gi1/1/1          Root FWD 4         128.25   P2p

VLAN0021
  Spanning tree enabled protocol rstp
  Root ID    Priority    24597
             Address     0005.5EAB.BA60
             Cost        8
             Port        25(GigabitEthernet1/1/1)
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32789  (priority 32768 sys-id-ext 21)
             Address     00E0.A343.7BCE
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Gi1/1/1          Root FWD 4         128.25   P2p
Gi1/0/1          Desg FWD 19        128.1    P2p

VLAN0050
  Spanning tree enabled protocol rstp
  Root ID    Priority    24626
             Address     0005.5EAB.BA60
             Cost        8
             Port        25(GigabitEthernet1/1/1)
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32818  (priority 32768 sys-id-ext 50)
             Address     00E0.A343.7BCE
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Gi1/1/1          Root FWD 4         128.25   P2p

VLAN0099
  Spanning tree enabled protocol rstp
  Root ID    Priority    24675
             Address     0005.5EAB.BA60
             Cost        8
             Port        25(GigabitEthernet1/1/1)
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32867  (priority 32768 sys-id-ext 99)
             Address     00E0.A343.7BCE
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Gi1/1/1          Root FWD 4         128.25   P2p
```

**EtherChannel на SW-Optical-1:**
```cisco
SW-Optical-1#show etherchannel summary 
Flags:  D - down        P - in port-channel
        I - stand-alone s - suspended
        H - Hot-standby (LACP only)
        R - Layer3      S - Layer2
        U - in use      f - failed to allocate aggregator
        u - unsuitable for bundling
        w - waiting to be aggregated
        d - default port


Number of channel-groups in use: 1
Number of aggregators:           1

Group  Port-channel  Protocol    Ports
------+-------------+-----------+----------------------------------------------

1      Po1(SU)           LACP   Gig1/0/1(P) Gig1/0/2(P) 
```

**Port Security на SW-Floor1:**
```cisco
SW-Floor1#show port-security 
Secure Port MaxSecureAddr CurrentAddr SecurityViolation Security Action
               (Count)       (Count)        (Count)
--------------------------------------------------------------------
        Fa0/1        1          1                 0         Shutdown
        Fa0/2        1          0                 0         Shutdown
        Fa0/3        1          0                 0         Shutdown
        Fa0/4        1          0                 0         Shutdown
        Fa0/5        1          0                 0         Shutdown
        Fa0/6        1          0                 0         Shutdown
        Fa0/7        1          0                 0         Shutdown
        Fa0/8        1          0                 0         Shutdown
        Fa0/9        1          0                 0         Shutdown
       Fa0/10        1          0                 0         Shutdown
       Fa0/11        1          0                 0         Shutdown
       Fa0/12        1          0                 0         Shutdown
       Fa0/13        1          0                 0         Shutdown
       Fa0/14        1          0                 0         Shutdown
       Fa0/15        1          0                 0         Shutdown
       Fa0/16        1          0                 0         Shutdown
       Fa0/17        1          0                 0         Shutdown
       Fa0/18        1          0                 0         Shutdown
       Fa0/19        1          0                 0         Shutdown
       Fa0/20        1          0                 0         Shutdown
----------------------------------------------------------------------
```

### 8.3. Проверка с конечных устройств

Проверяем работу DHCP-сервера на PC0, находящемся на первом этаже управления.
```cmd
C:\>ipconfig

FastEthernet0 Connection:(default port)

   Connection-specific DNS Suffix..: 
   Link-local IPv6 Address.........: FE80::203:E4FF:FEDA:E846
   IPv6 Address....................: ::
   IPv4 Address....................: 192.168.10.11
   Subnet Mask.....................: 255.255.255.0
   Default Gateway.................: ::
                                     192.168.10.1

Bluetooth Connection:

   Connection-specific DNS Suffix..: 
   Link-local IPv6 Address.........: ::
   IPv6 Address....................: ::
   IPv4 Address....................: 0.0.0.0
   Subnet Mask.....................: 0.0.0.0
   Default Gateway.................: ::
         
```

**Связь с шлюзом (ping 192.168.20.1):**
Проверяем с ПК цеха №1
```cmd
C:\>ping 192.168.20.1

Pinging 192.168.20.1 with 32 bytes of data:

Reply from 192.168.20.1: bytes=32 time<1ms TTL=255
Reply from 192.168.20.1: bytes=32 time<1ms TTL=255
Reply from 192.168.20.1: bytes=32 time<1ms TTL=255
Reply from 192.168.20.1: bytes=32 time<1ms TTL=255

Ping statistics for 192.168.20.1:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 0ms, Average = 0ms
```

### 8.4. Проверка изоляции
С ПК цеха№1 пробуем пропинговать ПК в управлении.
```cmd
C:\>ping 192.168.10.11

Pinging 192.168.10.11 with 32 bytes of data:

Reply from 192.168.20.1: Destination host unreachable.
Reply from 192.168.20.1: Destination host unreachable.
Reply from 192.168.20.1: Destination host unreachable.
Reply from 192.168.20.1: Destination host unreachable.

Ping statistics for 192.168.10.11:
    Packets: Sent = 4, Received = 0, Lost = 4 (100% loss),
```
Проверяем доступность камеры, подключенной к SW-Ceh1 Gi1/0/18
```cmd
C:\>ping 192.168.50.11

Pinging 192.168.50.11 with 32 bytes of data:

Reply from 192.168.20.1: Destination host unreachable.
Reply from 192.168.20.1: Destination host unreachable.
Reply from 192.168.20.1: Destination host unreachable.
Reply from 192.168.20.1: Destination host unreachable.

Ping statistics for 192.168.50.11:
    Packets: Sent = 4, Received = 0, Lost = 4 (100% loss),
```
Проверяем доступность видеонаблюдения с доменного ПК
```cmd
C:\>ping 192.168.50.11

Pinging 192.168.50.12 with 32 bytes of data:

Reply from 192.168.50.12: bytes=32 time<1ms TTL=127
Reply from 192.168.50.12: bytes=32 time<1ms TTL=127
Reply from 192.168.50.12: bytes=32 time<1ms TTL=127
Reply from 192.168.50.12: bytes=32 time<1ms TTL=127

Ping statistics for 192.168.50.12:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
```
**Списков доступа на Core-SW:**
```cisco
Core-SW#show access-lists 20
Standard IP access list 20
    deny 192.168.10.0 0.0.0.255
    deny 192.168.21.0 0.0.0.15
    deny 192.168.22.0 0.0.0.15
    deny 192.168.23.0 0.0.0.15
    deny 192.168.30.0 0.0.0.15
    deny 192.168.31.0 0.0.0.15
    deny 192.168.50.0 0.0.0.255
    deny 192.168.60.0 0.0.0.255
    deny 192.168.99.0 0.0.0.15
    permit any

```

### 8.5. Проверка выхода в Интернет
Проверяем с ПК автотранспортной проходной.
```
C:\>ping 8.8.8.8

Pinging 8.8.8.8 with 32 bytes of data:

Reply from 8.8.8.8: bytes=32 time<1ms TTL=253
Reply from 8.8.8.8: bytes=32 time<1ms TTL=253
Reply from 8.8.8.8: bytes=32 time<1ms TTL=253
Reply from 8.8.8.8: bytes=32 time<1ms TTL=253

Ping statistics for 8.8.8.8:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 0ms, Average = 0ms

```

### 8.6. Проверка SSH
С ПК на АТР
```cmd
C:\>ssh -l admin 192.168.99.1

Password: 

 Authorized Access Only! 

Core-SW>
```

---

## 9. Примечания и ограничения

1. **48-портовые коммутаторы:** В Cisco Packet Tracer 9.0 нет 48-портовых моделей. Использованы 24-портовые. В реальной сети — Cisco 2960-48PST-L (этажи, цеха) и Cisco 3560-24PS (проходные).

2. **PoE на этажах:** В CPT 9.0 коммутаторы 2960 не поддерживают PoE. Для гостевых Wi-Fi точек — внешние PoE-инжекторы или 2960-48PST-L.

3. **PoE в цехах и на проходных:** В CPT 9.0 используется 3650-24PS. В реальной сети — 2960-48PST-L (цеха) и 3560-24PS (проходные).

4. **Оптические порты:** Только 3650-24PS имеет SFP-порты (4 шт.). Для 7 оптических линков — два коммутатора (SW-Optical-1 и SW-Optical-2), соединённые EtherChannel.

5. **Standard ACL:** Использованы стандартные ACL (1–99). Для более точного контроля в реальной сети применяются расширенные ACL.

6. **ACL на SVI:** `out` — фильтрует трафик, входящий в VLAN.

7. **Пинг до SVI:** Пинг до IP-адреса SVI не фильтруется outbound ACL — это control plane трафик.


8. **Collapsed core:** Двухуровневая модель (Core + Distribution объединены в Core-SW) одобренная Cisco для предприятий среднего размера.

9. **Доменное имя:** `dok.local` — для SSH и NTP.

---

## 10. Выводы

Проект реализует:
- Иерархическую топологию с ядром на L3-коммутаторе Cisco 3650 (collapsed core).
- Сегментацию на 10 VLAN: домен, 4 цеха, 2 проходные, Video, гостевой , доменные пк ( управление).
- Изоляцию промышленных сетей и проходных через стандартные ACL.
- Доступ к видеонаблюдению только из доменной сети.
- Полную изоляцию гостевой сети от внутренних ресурсов с сохранением выхода в Интернет.
- Автоматическую выдачу адресов через DHCPv4.
- Выход в Интернет через NAT (PAT).
- Динамическую маршрутизацию OSPF между Edge-Router и Core-SW.
- Управление через NTP, SNMP (community), CDP/LLDP, SSH.
- Отказоустойчивость через STP (Rapid-PVST+) и EtherChannel (LACP).
- Масштабируемость за счёт L3-ядра и модульной архитектуры.
- Экономию бюджета за счёт использования Cisco 2960-24TT на этажах.
