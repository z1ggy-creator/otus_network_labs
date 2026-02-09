# eBGP Underlay сеть на unnumbered IPv6. VxLAN. Routing.

---

## 1. План работ  

### Настройка Муотихоум для clients-01, clients-02
- [ ] Разместить двух "клиентов" в разных VRF в рамках одной фабрики.
- [ ] Настроить маршрутизацию между клиентами через внешнее устройство (граничный роутер\фаерволл\etc)


### Тестирование и проверка  
- [ ] Проверить маршруты 5 типа на коммутаторах фабрики
- [ ] Проверить связь между клиентами

---

## 2. Адресное пространство  

### 2.1. Loopback интерфейсы
Для уникальных локальных адресов (ULA) используется префикс FD00::/8. Мы выделим блок /48 для лупбэков из ULA. 

**Адрес сети:** `fd00:cafe:beef::/48`  

| Устройство | IPv6-адрес       |
|------------|----------------|
| Spine-1    | fd00:cafe:beef::1/128 |
| Spine-2    | fd00:cafe:beef::2/128  |
| Leaf-1     | fd00:cafe:beef:1::1/128 |
| Leaf-2     | fd00:cafe:beef:1::2/128 |
| Leaf-3     | fd00:cafe:beef:1::3/128 |

### 2.2. Point-to-Point интерфейсы   

Для IPv6 Unnumbered не требуются статические адреса. Достаточно включить ipv6 address auto-config для генерации линк-локал адресов. 

---

## 3. Схема Underlay и Overlay сети на eBGP  

### 3.1. Топология  

![topology_vxlan_routing.png](topology_vxlan_routing.png)

### 3.2. Параметры eBGP  

#### Общие настройки:  
- **AS SPINE** Для спайн выделим AS `65000`
- **AS LEAF** Для лиф выделим диапазон AS `65100-65200`. Так как каждый лиф находится в своей AS
- **ipv6** Для работы ipv6 маршрутизации включим ipv6 unicast-routing
 - **multi-agent** Для работы несколько AFI\SAFI включим на каждом коммутаторе - service routing protocols model multi-agent

### 3.3. Таблица Автономных систем  

| Устройство | AS |
|------------|-----------|
| **Spine-1**| 65000    | 
| **Spine-2**| 65000    |
| **Leaf-01** | 65101    | 
| **Leaf-02** | 65102    | 
| **Leaf-03** | 65103    |
| **Leaf-04** | 65104    | 


## 4. Конфигурация протокола eBGP и интерфейсов.  

### 4.0. SPINE
На спайн настраиваем фильтр автоновных систем лифов, с которыми будем устанавливать соединения. 
```
peer-filter LEAVES_ASN
   10 match as-range 65100-65300 result accept

```

### 4.1. SPINE-1 
```
interface Ethernet1
   description TO-LEAF-1
   mtu 9000
   no switchport
   ipv6 enable
   ipv6 address auto-config
!
interface Ethernet2
   description TO-LEAF-2
   mtu 9000
   no switchport
   ipv6 enable
   ipv6 address auto-config
!
interface Ethernet3
   description TO-LEAF-3
   mtu 9000
   no switchport
   ipv6 enable
   ipv6 address auto-config
!
interface Ethernet4
   description TO-LEAF-04
   mtu 9000
   no switchport
   ipv6 enable
   ipv6 address auto-config
!
interface Loopback0
   description Router-ID
   ipv6 enable
   ipv6 address fd00:cafe:beef::1/128
```
```
router bgp 65000
   router-id 10.255.255.1
   no bgp default ipv4-unicast
   timers bgp 1 3
   distance bgp 20 200 200
   maximum-paths 2
   bgp listen range fe80::/10 peer-group UNDERLAY peer-filter LEAVES_ASN
   neighbor OVERLAY peer group
   neighbor OVERLAY next-hop-unchanged
   neighbor OVERLAY out-delay 0
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY ebgp-multihop 3
   neighbor OVERLAY send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY out-delay 0
   neighbor fd00:cafe:beef:1::1 peer group OVERLAY
   neighbor fd00:cafe:beef:1::1 remote-as 65101
   neighbor fd00:cafe:beef:1::2 peer group OVERLAY
   neighbor fd00:cafe:beef:1::2 remote-as 65102
   neighbor fd00:cafe:beef:1::3 peer group OVERLAY
   neighbor fd00:cafe:beef:1::3 remote-as 65103
   neighbor fd00:cafe:beef:1::4 peer group OVERLAY
   neighbor fd00:cafe:beef:1::4 remote-as 65104
   !
   address-family evpn
      neighbor OVERLAY activate
   !
   address-family ipv6
      neighbor UNDERLAY activate
      network fd00:cafe:beef::1/128
!
end
```
### 4.2. SPINE-2
```
interface Ethernet1
   description TO-LEAF-1
   mtu 9000
   no switchport
   ipv6 enable
   ipv6 address auto-config
!
interface Ethernet2
   description TO-LEAF-2
   mtu 9000
   no switchport
   ipv6 enable
   ipv6 address auto-config
!
interface Ethernet3
   description TO-LEAF-3
   mtu 9000
   no switchport
   ipv6 enable
   ipv6 address auto-config
!
interface Ethernet4
   description TO-LEAF-04
   mtu 9000
   no switchport
   ipv6 enable
   ipv6 address auto-config
!
interface Loopback0
   description Router-ID
   ipv6 enable
   ipv6 address fd00:cafe:beef::2/128
!
```
```
router bgp 65000
   router-id 10.255.255.2
   no bgp default ipv4-unicast
   timers bgp 1 3
   distance bgp 20 200 200
   maximum-paths 2
   bgp listen range fe80::/10 peer-group UNDERLAY peer-filter LEAVES_ASN
   neighbor OVERLAY peer group
   neighbor OVERLAY next-hop-unchanged
   neighbor OVERLAY out-delay 0
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY ebgp-multihop 3
   neighbor OVERLAY send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY out-delay 0
   neighbor fd00:cafe:beef:1::1 peer group OVERLAY
   neighbor fd00:cafe:beef:1::1 remote-as 65101
   neighbor fd00:cafe:beef:1::2 peer group OVERLAY
   neighbor fd00:cafe:beef:1::2 remote-as 65102
   neighbor fd00:cafe:beef:1::3 peer group OVERLAY
   neighbor fd00:cafe:beef:1::3 remote-as 65103
   neighbor fd00:cafe:beef:1::4 peer group OVERLAY
   neighbor fd00:cafe:beef:1::4 remote-as 65104
   !
   address-family evpn
      neighbor OVERLAY activate
   !
   address-family ipv6
      neighbor UNDERLAY activate
      network fd00:cafe:beef::2/128

```
### 4.3. LEAF-1
```
vlan 100
   name vlan-100
!
ip routing vrf vrf-red
vrf instance vrf-red
!
interface Port-Channel1
   description TO_CLIENTS-01
   switchport access vlan 100
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:1111
      route-target import 00:00:00:00:11:11
   lacp system-id 0000.0000.1111
   link tracking group UPLINKS downstream
!
interface Ethernet1
   description TO-SPINE-1
   mtu 9000
   no switchport
   ipv6 enable
   ipv6 address auto-config
   link tracking group UPLINKS upstream
!
interface Ethernet2
   description TO-SPINE-2
   mtu 9000
   no switchport
   ipv6 enable
   ipv6 address auto-config
   link tracking group UPLINKS upstream
!
interface Ethernet3
   description TO-CLIENTS-1
   channel-group 1 mode active
!
interface Loopback0
   description Router-ID & Overlay Endpoint
   ipv6 enable
   ipv6 address fd00:cafe:beef:1::1/128
!
interface Vlan100
   vrf vrf-red
   ip address virtual 10.10.10.254/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan encapsulation ipv6
   vxlan vlan 100 vni 19100
   vxlan vrf vrf-red vni 3089
!
ip virtual-router mac-address 00:00:00:00:00:12

```
```
router bgp 65101
   router-id 10.255.255.11
   no bgp default ipv4-unicast
   timers bgp 1 3
   distance bgp 20 200 200
   maximum-paths 2
   neighbor OVERLAY peer group
   neighbor OVERLAY remote-as 65000
   no neighbor OVERLAY next-hop-unchanged
   neighbor OVERLAY out-delay 0
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY ebgp-multihop 3
   neighbor OVERLAY send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY out-delay 0
   neighbor UNDERLAY send-community extended
   neighbor fd00:cafe:beef::1 peer group OVERLAY
   neighbor fd00:cafe:beef::2 peer group OVERLAY
   neighbor interface Et1-2 peer-group UNDERLAY remote-as 65000
   !
   vlan 100
      rd auto
      route-target both 65101:19100
      redistribute learned
   !
   address-family evpn
      neighbor OVERLAY activate
   !
   address-family ipv6
      neighbor UNDERLAY activate
      network fd00:cafe:beef:1::1/128
   !
   vrf vrf-red
      rd 10.255.255.11:3089
      route-target import evpn 65000:3089
      route-target export evpn 65000:3089
!
end
```

### 4.4. LEAF-2
```
vlan 100
   name vlan-100
!
ip routing vrf vrf-red
vrf instance vrf-red
!
interface Port-Channel1
   description TO_CLIENTS-01
   switchport access vlan 100
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:1111
      route-target import 00:00:00:00:11:11
   lacp system-id 0000.0000.1111
   link tracking group UPLINKS downstream
!
interface Ethernet1
   description TO-SPINE-1
   mtu 9000
   no switchport
   ipv6 enable
   ipv6 address auto-config
   link tracking group UPLINKS upstream
!
interface Ethernet2
   description TO-SPINE-2
   mtu 9000
   no switchport
   ipv6 enable
   ipv6 address auto-config
   link tracking group UPLINKS upstream
!
interface Ethernet4
   description TO_CLIENTS-01
   channel-group 1 mode active
!
interface Loopback0
   description Router-ID & Overlay Endpoint
   ipv6 enable
   ipv6 address fd00:cafe:beef:1::2/128
!
interface Vlan100
   vrf vrf-red
   ip address virtual 10.10.10.254/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan encapsulation ipv6
   vxlan vlan 100 vni 19100
   vxlan vrf vrf-red vni 3089
!
ip virtual-router mac-address 00:00:00:00:00:12
```
```
router bgp 65102
   router-id 10.255.255.12
   no bgp default ipv4-unicast
   timers bgp 1 3
   distance bgp 20 200 200
   maximum-paths 2
   neighbor OVERLAY peer group
   neighbor OVERLAY remote-as 65000
   neighbor OVERLAY out-delay 0
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY ebgp-multihop 3
   neighbor OVERLAY send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY out-delay 0
   neighbor UNDERLAY send-community extended
   neighbor fd00:cafe:beef::1 peer group OVERLAY
   neighbor fd00:cafe:beef::2 peer group OVERLAY
   redistribute connected
   neighbor interface Et1-2 peer-group UNDERLAY remote-as 65000
   !
   vlan 100
      rd auto
      route-target both 65101:19100
   !
   address-family evpn
      neighbor OVERLAY activate
   !
   address-family ipv6
      neighbor UNDERLAY activate
      network fd00:cafe:beef:1::2/128
   !
   vrf vrf-red
      rd 10.255.255.12:3089
      route-target import evpn 65000:3089
      route-target export evpn 65000:3089
!
end
```

### 4.5. LEAF-3
```
vlan 300
   name vlan-300
!
ip routing vrf vrf-red
vrf instance vrf-red
!
interface Port-Channel1
   description TO_CLIENTS-02
   switchport access vlan 300
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:3333
      route-target import 00:00:00:00:33:33
   lacp system-id 0000.0000.3333
   link tracking group UPLINKS downstream
!
interface Ethernet1
   description TO-SPINE-1
   mtu 9000
   no switchport
   ipv6 enable
   ipv6 address auto-config
   link tracking group UPLINKS upstream
!
interface Ethernet2
   description TO-SPINE-2
   mtu 9000
   no switchport
   ipv6 enable
   ipv6 address auto-config
   link tracking group UPLINKS upstream
!
interface Ethernet3
   description TO_CLIENTS_02
   channel-group 1 mode active
!
interface Loopback0
   ipv6 enable
   ipv6 address fd00:cafe:beef:1::3/128
!
interface Vlan300
   vrf vrf-red
   ip address virtual 30.30.30.254/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan encapsulation ipv6
   vxlan vlan 300 vni 19300
   vxlan vrf vrf-red vni 3089
!
ip virtual-router mac-address 00:00:00:00:00:34

```
```
router bgp 65103
   router-id 10.255.255.13
   no bgp default ipv4-unicast
   timers bgp 1 3
   distance bgp 20 200 200
   maximum-paths 2
   neighbor OVERLAY peer group
   neighbor OVERLAY remote-as 65000
   neighbor OVERLAY out-delay 0
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY ebgp-multihop 3
   neighbor OVERLAY send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY out-delay 0
   neighbor UNDERLAY send-community extended
   neighbor fd00:cafe:beef::1 peer group OVERLAY
   neighbor fd00:cafe:beef::2 peer group OVERLAY
   neighbor interface Et1-2 peer-group UNDERLAY remote-as 65000
   !
   vlan 300
      rd auto
      route-target both 65103:19300
      redistribute learned
   !
   address-family evpn
      neighbor OVERLAY activate
   !
   address-family ipv6
      neighbor UNDERLAY activate
      network fd00:cafe:beef:1::3/128
   !
   vrf vrf-red
      rd 10.255.255.13:3089
      route-target import evpn 65000:3089
      route-target export evpn 65000:3089
!
end
```
### 4.5. LEAF-4

```
vlan 300
   name vlan-300
!
ip routing vrf vrf-red
vrf instance vrf-red
!
interface Port-Channel1
   description TO_CLIENTS-02
   switchport access vlan 300
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:3333
      route-target import 00:00:00:00:33:33
   lacp system-id 0000.0000.3333
   link tracking group UPLINKS downstream
!
interface Ethernet1
   description TO-SPINE-1
   mtu 9000
   no switchport
   ipv6 enable
   ipv6 address auto-config
!
interface Ethernet2
   description TO-SPINE-2
   mtu 9000
   no switchport
   ipv6 enable
   ipv6 address auto-config
!
interface Ethernet3
!
interface Ethernet4
   description TO_CLIENTS_02
   channel-group 1 mode active
!
interface Loopback0
   ipv6 enable
   ipv6 address fd00:cafe:beef:1::4/128
!
interface Vlan300
   vrf vrf-red
   ip address virtual 30.30.30.254/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan encapsulation ipv6
   vxlan vlan 300 vni 19300
   vxlan vrf vrf-red vni 3089
!
ip virtual-router mac-address 00:00:00:00:00:34

```
```
router bgp 65104
   router-id 10.255.255.14
   no bgp default ipv4-unicast
   timers bgp 1 3
   distance bgp 20 200 200
   maximum-paths 2
   neighbor OVERLAY peer group
   neighbor OVERLAY remote-as 65000
   neighbor OVERLAY out-delay 0
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY ebgp-multihop 3
   neighbor OVERLAY send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY out-delay 0
   neighbor UNDERLAY send-community extended
   neighbor fd00:cafe:beef::1 peer group OVERLAY
   neighbor fd00:cafe:beef::2 peer group OVERLAY
   neighbor interface Et1-2 peer-group UNDERLAY remote-as 65000
   !
   vlan 300
      rd auto
      route-target both 65103:19300
      redistribute learned
   !
   address-family evpn
      neighbor OVERLAY activate
   !
   address-family ipv6
      neighbor UNDERLAY activate
      network fd00:cafe:beef:1::4/128
   !
   vrf vrf-red
      rd 10.255.255.14:3089
      route-target import evpn 65000:3089
      route-target export evpn 65000:3089
!
end
```

### 4.6. CLIENTS-1
```
vlan 100
   name vlan-100
!
interface Port-Channel1
   description TO_FABRIC
   switchport access vlan 100
!
interface Ethernet1
   channel-group 1 mode active
!
interface Ethernet2
   description TO_LEAF-02
   channel-group 1 mode active
!
interface Vlan100
   description CLIENTS_NETWORK
   ip address 10.10.10.1/24
!
ip routing
!
ip route 0.0.0.0/0 10.10.10.254
!
end

```
### 4.6. CLIENTS-2
```
vlan 300
   name vlane-300
!
interface Port-Channel1
   description TO_FABRIC
   switchport access vlan 300
!
interface Ethernet1
   description TO-LEAF-03
   channel-group 1 mode active
!
interface Ethernet2
   description TO-LEAF-04
   channel-group 1 mode active
!
interface Vlan300
   ip address 30.30.30.1/24
!
ip routing
!
ip route 0.0.0.0/0 30.30.30.254
!
end

```
## 5. Тестирование и проверка eBGP

### 5.0 Проверить маршруты 5 типа на коммутаторах фабрики

```

```



### 5.1 Проверить связь между клиентами

```


```
 