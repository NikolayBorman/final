# Проектная работа: Построение ЛВС деревообрабатывающего предприятия

## Цель проекта
Спроектировать и настроить масштабируемую, отказоустойчивую и защищённую локальную вычислительную сеть деревообрабатывающего предприятия на основе оборудования Cisco.

## Задачи
1. Разработать иерархическую топологию сети и спланировать адресное пространство.
2. Настроить коммутацию: VLAN, trunk, Port Security, SSH.
3. Реализовать маршрутизацию между VLAN (Router-on-a-stick) и выход в Интернет (NAT).
4. Развернуть DHCP-сервис для автоматической выдачи адресов.
5. Обеспечить безопасность с помощью расширенных ACL.

## Топология сети


<img width="955" height="665" alt="image" src="https://github.com/user-attachments/assets/cda582c3-adaf-4350-bec7-4c58ab0101fa" />



## План VLAN и IP-адресации

| VLAN | Назначение              | Подсеть          | Шлюз          |
|------|-------------------------|------------------|---------------|
| 10   | Администрация           | 192.168.10.0/26  | 192.168.10.1  |
| 20   | Бухгалтерия             | 192.168.20.0/27  | 192.168.20.1  |
| 30   | Производство (цеха)     | 192.168.30.0/28  | 192.168.30.1  |
| 40   | Склад/Проходная         | 192.168.40.0/28  | 192.168.40.1  |
| 50   | Видеонаблюдение         | 192.168.50.0/27  | 192.168.50.1  |
| 99   | Управление              | 192.168.99.0/29  | 192.168.99.1  |

---

## 1. ISP-Router (Cisco 2911) — имитация провайдера

```cisco
enable
configure terminal
hostname ISP-Router
no ip domain-lookup
!
interface GigabitEthernet0/0
 description To Core-Router
 ip address 10.0.0.2 255.255.255.252
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

## 2. Core-Router (Cisco 2911) — ядро сети

```cisco
enable
configure terminal
hostname Core-Router
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
```

### 2.1. Интерфейс к провайдеру

```cisco
interface GigabitEthernet0/0
 description To ISP-Router
 ip address 10.0.0.1 255.255.255.252
 ip nat outside
 no shutdown
```

### 2.2. Интерфейс к SW-Floor1

```cisco
interface GigabitEthernet0/1
 description To SW-Floor1
 no shutdown
!
interface GigabitEthernet0/1.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.192
 ip nat inside
!
interface GigabitEthernet0/1.99
 encapsulation dot1Q 99
 ip address 192.168.99.1 255.255.255.248
 ip nat inside
```

### 2.3. Интерфейс к SW-Floor2

```cisco
interface GigabitEthernet0/2
 description To SW-Floor2
 no shutdown
!
interface GigabitEthernet0/2.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.224
 ip nat inside
```

### 2.4. Интерфейс к SW-Agg (через HWIC-1GE-SFP)

```cisco
interface GigabitEthernet0/3/0
 description To SW-Agg (fiber)
 no shutdown
!
interface GigabitEthernet0/3/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.240
 ip nat inside
!
interface GigabitEthernet0/3/0.40
 encapsulation dot1Q 40
 ip address 192.168.40.1 255.255.255.240
 ip nat inside
!
interface GigabitEthernet0/3/0.50
 encapsulation dot1Q 50
 ip address 192.168.50.1 255.255.255.224
 ip nat inside
!
interface GigabitEthernet0/3/0.99
 encapsulation dot1Q 99
 ip address 192.168.99.1 255.255.255.248
 ip nat inside
```

### 2.5. DHCP-сервер

```cisco
ip dhcp excluded-address 192.168.10.1 192.168.10.9
ip dhcp excluded-address 192.168.20.1 192.168.20.5
ip dhcp excluded-address 192.168.30.1 192.168.30.2
ip dhcp excluded-address 192.168.40.1 192.168.40.2
ip dhcp excluded-address 192.168.50.1 192.168.50.5
ip dhcp excluded-address 192.168.99.1 192.168.99.5
!
ip dhcp pool VLAN10_ADMIN
 network 192.168.10.0 255.255.255.192
 default-router 192.168.10.1
 dns-server 8.8.8.8
!
ip dhcp pool VLAN20_ACCOUNT
 network 192.168.20.0 255.255.255.224
 default-router 192.168.20.1
 dns-server 8.8.8.8
!
ip dhcp pool VLAN30_PRODUCTION
 network 192.168.30.0 255.255.255.240
 default-router 192.168.30.1
 dns-server 8.8.8.8
!
ip dhcp pool VLAN40_WAREHOUSE
 network 192.168.40.0 255.255.255.240
 default-router 192.168.40.1
 dns-server 8.8.8.8
!
ip dhcp pool VLAN50_CCTV
 network 192.168.50.0 255.255.255.224
 default-router 192.168.50.1
 dns-server 8.8.8.8
```

### 2.6. NAT (PAT)

```cisco
access-list 1 permit 192.168.0.0 0.0.255.255
ip nat inside source list 1 interface GigabitEthernet0/0 overload
```

### 2.7. Маршрут по умолчанию

```cisco
ip route 0.0.0.0 0.0.0.0 10.0.0.2
```

### 2.8. ACL — изоляция Server0 от цехов и проходной

```cisco
access-list 120 deny ip 192.168.30.0 0.0.0.15 host 192.168.10.10
access-list 120 deny ip 192.168.40.0 0.0.0.15 host 192.168.10.10
access-list 120 permit ip any any
!
interface GigabitEthernet0/3/0.30
 ip access-group 120 in
!
interface GigabitEthernet0/3/0.40
 ip access-group 120 in
!
end
write memory
```

## 3. SW-Floor1 (Cisco 3560-24PS) — 1-й этаж, VLAN 10

**Подключено:** Server0 → fa0/1, PC0 → fa0/2. Транк к ядру — gi0/1.

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
line console 0
 password cisco
 login
 logging synchronous
line vty 0 15
 login local
 transport input ssh
ip domain-name example.com
crypto key generate rsa modulus 1024
!
vlan 10
 name Administration
vlan 99
 name Management
!
interface vlan 10
 ip address 192.168.10.2 255.255.255.192
 no shutdown
ip default-gateway 192.168.10.1
!
interface fa0/1
 description To Server0
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
!
interface fa0/2
 description To PC0
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
!
interface gi0/1
 description To Core-Router
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,99
!
end
write memory
```

## 4. SW-Floor2 (Cisco 3560-24PS) — 2-й этаж, VLAN 20

**Подключено:** PC1 → fa0/1, Printer0 → fa0/2. Транк к ядру — gi0/1.

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
line console 0
 password cisco
 login
 logging synchronous
line vty 0 15
 login local
 transport input ssh
ip domain-name example.com
crypto key generate rsa modulus 1024
!
vlan 20
 name Accounting
!
interface vlan 20
 ip address 192.168.20.2 255.255.255.224
 no shutdown
ip default-gateway 192.168.20.1
!
interface fa0/1
 description To PC1
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
!
interface fa0/2
 description To Printer0
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
!
interface gi0/1
 description To Core-Router
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 20
!
end
write memory
```

## 5. SW-Agg (Cisco 3650-24PS) — агрегация, серверная

**Подключено:** gi1/1/1 → Core (оптика), gi1/1/2 → SW-Workshop1, gi1/1/3 → SW-Workshop2, gi1/1/4 → SW-Enter.

```cisco
enable
configure terminal
hostname SW-Agg
no ip domain-lookup
!
enable secret class
service password-encryption
!
username admin secret AdminPass123
line console 0
 password cisco
 login
 logging synchronous
line vty 0 15
 login local
 transport input ssh
ip domain-name example.com
crypto key generate rsa modulus 1024
!
vlan 30
 name Production
vlan 40
 name Warehouse
vlan 50
 name CCTV
vlan 99
 name Management
!
interface vlan 99
 ip address 192.168.99.2 255.255.255.248
 no shutdown
ip default-gateway 192.168.99.1
!
interface GigabitEthernet1/1/1
 description To Core-Router (fiber)
 switchport mode trunk
 switchport trunk allowed vlan 30,40,50,99
!
interface GigabitEthernet1/1/2
 description To SW-Workshop1 (fiber)
 switchport mode trunk
 switchport trunk allowed vlan 30,50,99
!
interface GigabitEthernet1/1/3
 description To SW-Workshop2 (fiber)
 switchport mode trunk
 switchport trunk allowed vlan 30,50,99
!
interface GigabitEthernet1/1/4
 description To SW-Enter (fiber)
 switchport mode trunk
 switchport trunk allowed vlan 40,50,99
!
end
write memory
```

## 6. SW-Workshop1 (Cisco 3650-24PS) — Цех №1

**Подключено:** PC3 → Gi1/0/1. Транк к SW-Agg — Gi1/1/1.

```cisco
enable
configure terminal
hostname SW-Workshop1
no ip domain-lookup
!
enable secret class
service password-encryption
!
username admin secret AdminPass123
line console 0
 password cisco
 login
 logging synchronous
line vty 0 15
 login local
 transport input ssh
ip domain-name example.com
crypto key generate rsa modulus 1024
!
vlan 30
 name Production
vlan 50
 name CCTV
vlan 99
 name Management
!
interface vlan 99
 ip address 192.168.99.3 255.255.255.248
 no shutdown
ip default-gateway 192.168.99.1
!
interface GigabitEthernet1/0/1
 description To PC3
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
!
interface GigabitEthernet1/1/1
 description To SW-Agg (fiber)
 switchport mode trunk
 switchport trunk allowed vlan 30,50,99
!
end
write memory
```

## 7. SW-Workshop2 (Cisco 3650-24PS) — Цех №2

**Подключено:** PC4 → Gi1/0/1. Транк к SW-Agg — Gi1/1/1.

```cisco
enable
configure terminal
hostname SW-Workshop2
no ip domain-lookup
!
enable secret class
service password-encryption
!
username admin secret AdminPass123
line console 0
 password cisco
 login
 logging synchronous
line vty 0 15
 login local
 transport input ssh
ip domain-name example.com
crypto key generate rsa modulus 1024
!
vlan 30
 name Production
vlan 50
 name CCTV
vlan 99
 name Management
!
interface vlan 99
 ip address 192.168.99.4 255.255.255.248
 no shutdown
ip default-gateway 192.168.99.1
!
interface GigabitEthernet1/0/1
 description To PC4
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
!
interface GigabitEthernet1/1/1
 description To SW-Agg (fiber)
 switchport mode trunk
 switchport trunk allowed vlan 30,50,99
!
end
write memory
```

## 8. SW-Enter (Cisco 3650-24PS) — Проходная

**Подключено:** PC2 → Gi1/0/1. Транк к SW-Agg — Gi1/1/1.

```cisco
enable
configure terminal
hostname SW-Enter
no ip domain-lookup
!
enable secret class
service password-encryption
!
username admin secret AdminPass123
line console 0
 password cisco
 login
 logging synchronous
line vty 0 15
 login local
 transport input ssh
ip domain-name example.com
crypto key generate rsa modulus 1024
!
vlan 40
 name Warehouse
vlan 50
 name CCTV
vlan 99
 name Management
!
interface vlan 99
 ip address 192.168.99.5 255.255.255.248
 no shutdown
ip default-gateway 192.168.99.1
!
interface GigabitEthernet1/0/1
 description To PC2
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
!
interface GigabitEthernet1/1/1
 description To SW-Agg (fiber)
 switchport mode trunk
 switchport trunk allowed vlan 40,50,99
!
end
write memory
```

## 9. Настройки конечных устройств

| Устройство | Подключение       | IP-адрес              | Маска           | Шлюз         |
|------------|-------------------|-----------------------|-----------------|--------------|
| Server0    | SW-Floor1 fa0/1   | 192.168.10.10 (статич.) | 255.255.255.192 | 192.168.10.1 |
| PC0        | SW-Floor1 fa0/2   | DHCP                  | —               | —            |
| PC1        | SW-Floor2 fa0/1   | DHCP                  | —               | —            |
| Printer0   | SW-Floor2 fa0/2   | 192.168.20.10 (статич.) | 255.255.255.224 | 192.168.20.1 |
| PC3        | SW-WS1 Gi1/0/1    | DHCP                  | —               | —            |
| PC4        | SW-WS2 Gi1/0/1    | DHCP                  | —               | —            |
| PC2        | SW-Enter Gi1/0/1  | DHCP                  | —               | —            |

> DHCP-сервер на Core-Router выдаёт адреса автоматически. Server0 и Printer0 рекомендуется настроить статически, чтобы ACL и обращения по IP были стабильными.

## 10. Проверка работоспособности

### На Core-Router:
```cisco
show ip route
show ip nat translations
show access-lists 120
```

### На коммутаторах:
```cisco
show vlan brief
show interfaces trunk
show port-security
```

### С ПК:
```cisco
ipconfig                  ! проверка DHCP
ping 192.168.10.10        ! Server0 — должен быть недоступен с PC3, PC4, PC2
ping 192.168.20.1         ! шлюз VLAN 20 — должен работать
ping 8.8.8.8              ! выход в интернет — должен работать
```

### Проверка SSH:
```cisco
ssh -l admin 192.168.99.2   ! SW-Agg
ssh -l admin 192.168.99.3   ! SW-Workshop1
```

## 11. Результаты

| Требование                | Реализация                               |
|---------------------------|------------------------------------------|
| VLAN-сегментация          | VLAN 10, 20, 30, 40, 50, 99              |
| Маршрутизация между VLAN  | Router-on-a-stick на Core-Router         |
| DHCP                      | Пулы для каждой VLAN на Core-Router      |
| NAT (PAT)                 | Выход в интернет через gi0/0             |
| Безопасность портов       | Port Security на всех портах доступа     |
| ACL                       | Изоляция Server0 от цехов и проходной    |
| Удалённое управление      | SSH на всех коммутаторах                 |
| Отказоустойчивость        | Оптические линки, STP                    |
| Масштабируемость          | 3650 в агрегации поддерживает StackWise  |

