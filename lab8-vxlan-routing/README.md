# VXLAN EVPN - External Routing Between VRFs

## Цель

Цель лабораторной работы - разместить двух клиентов в разных VRF внутри одной VXLAN EVPN фабрики и настроить маршрутизацию между ними через внешнее устройство.

В данной лабораторной работе используются два клиентских VRF:

* `RED` - клиент Host1;
* `BLUE` - клиент Host2.

Маршрутизация между VRF `RED` и `BLUE` выполняется не через route leaking внутри фабрики, а через внешний маршрутизатор `R1`. В качестве внешнего маршрутизатора используется Cisco C7200.

## Задачи

В рамках работы выполнено:

1. Настроена Spine-Leaf фабрика на Arista vEOS.
2. Настроен eBGP underlay между Spine и Leaf.
3. Настроен EVPN control-plane.
4. Настроены два VRF: `RED` и `BLUE`.
5. Настроены два L3VNI:

   * `50100` для VRF `RED`;
   * `50200` для VRF `BLUE`.
6. Настроены клиентские L2VNI:

   * `10010` для VLAN 10;
   * `10020` для VLAN 20.
7. Host1 размещен в VRF `RED`.
8. Host2 размещен в VRF `BLUE`.
9. Leaf3 настроен как Border Leaf.
10. R1 подключен к Leaf3 через trunk и выполняет маршрутизацию между VRF.
11. Проверена связность между Host1 и Host2.

## Топология

![Топология](./topology.png)

## Логическая схема

```text
Host1
  |
Leaf1
  |
VRF RED / L3VNI 50100
  |
VXLAN EVPN Fabric
  |
Leaf3
  |
VRF RED transit
  |
R1
  |
VRF BLUE transit
  |
Leaf3
  |
VXLAN EVPN Fabric
  |
VRF BLUE / L3VNI 50200
  |
Leaf2
  |
Host2
```

Основная логика маршрутизации:

```text
Host1 -> Leaf1 VRF RED -> VXLAN EVPN -> Leaf3 VRF RED -> R1
R1 -> Leaf3 VRF BLUE -> VXLAN EVPN -> Leaf2 VRF BLUE -> Host2
```

Обратный трафик проходит симметрично через R1.

## Устройства

| Устройство | Роль                            |
| ---------- | ------------------------------- |
| Spine1     | Spine-коммутатор                |
| Spine2     | Spine-коммутатор                |
| Leaf1      | Leaf для клиента RED            |
| Leaf2      | Leaf для клиента BLUE           |
| Leaf3      | Border Leaf                     |
| R1         | Внешний маршрутизатор между VRF |
| Host1      | Клиент в VRF RED                |
| Host2      | Клиент в VRF BLUE               |

## Адресный план

### ASN

| Устройство |   ASN |
| ---------- | ----: |
| Spine1     | 65000 |
| Spine2     | 65001 |
| Leaf1      | 65101 |
| Leaf2      | 65102 |
| Leaf3      | 65103 |

### Loopback

| Устройство |      Loopback0 | Loopback1 / VTEP |
| ---------- | -------------: | ---------------: |
| Spine1     |     1.1.1.1/32 |                - |
| Spine2     |     2.2.2.2/32 |                - |
| Leaf1      | 11.11.11.11/32 | 100.100.100.1/32 |
| Leaf2      | 22.22.22.22/32 | 100.100.100.2/32 |
| Leaf3      | 33.33.33.33/32 | 100.100.100.3/32 |

### Underlay-сети

| Линк           |         Сеть |
| -------------- | -----------: |
| Spine1 - Leaf1 |  10.0.0.0/31 |
| Spine1 - Leaf2 |  10.0.0.2/31 |
| Spine1 - Leaf3 |  10.0.0.4/31 |
| Spine2 - Leaf1 |  10.0.0.6/31 |
| Spine2 - Leaf2 |  10.0.0.8/31 |
| Spine2 - Leaf3 | 10.0.0.10/31 |

### VRF и VNI

| VRF  | L3VNI | Назначение            |
| ---- | ----: | --------------------- |
| RED  | 50100 | Клиентская сеть Host1 |
| BLUE | 50200 | Клиентская сеть Host2 |

### VLAN и L2VNI

| VLAN | L2VNI | VRF  | Назначение |
| ---: | ----: | ---- | ---------- |
|   10 | 10010 | RED  | Host1      |
|   20 | 10020 | BLUE | Host2      |

### Клиентские сети

| Клиент | VRF  |             IP |    Gateway |
| ------ | ---- | -------------: | ---------: |
| Host1  | RED  | 10.10.10.10/24 | 10.10.10.1 |
| Host2  | BLUE | 10.20.20.10/24 | 10.20.20.1 |

### Transit-сети к R1

R1 подключен к Leaf3 через trunk. На R1 используются subinterface.

| VRF  | VLAN |        Leaf3 IP |           R1 IP |
| ---- | ---: | --------------: | --------------: |
| RED  |  100 | 172.16.100.1/30 | 172.16.100.2/30 |
| BLUE |  200 | 172.16.200.1/30 | 172.16.200.2/30 |

## Принцип маршрутизации

В лабораторной работе нет прямого route leaking между VRF `RED` и `BLUE` внутри VXLAN EVPN фабрики.

На Leaf3 настроены статические маршруты:

```text
VRF RED:
10.20.20.0/24 -> 172.16.100.2

VRF BLUE:
10.10.10.0/24 -> 172.16.200.2
```

То есть Leaf3 отправляет трафик между VRF на внешний маршрутизатор R1.

На R1 настроены обратные маршруты:

```text
10.10.10.0/24 -> 172.16.100.1
10.20.20.0/24 -> 172.16.200.1
```

Таким образом, межVRF-маршрутизация выполняется через внешний router-on-a-stick.

## Конфигурации

Полные конфигурации устройств находятся в каталоге `configs`.

```text
configs/
├── Spine1.cfg
├── Spine2.cfg
├── Leaf1.cfg
├── Leaf2.cfg
├── Leaf3.cfg
├── R1.cfg
├── Host1.sh
└── Host2.sh
```

## Ключевые фрагменты конфигурации

### Leaf1 - VRF RED

```eos
vlan 10
   name HOST1_RED

vrf instance RED

interface Ethernet3
   description TO_HOST1
   switchport access vlan 10
   spanning-tree portfast

interface Vlan10
   description HOST1_GW_RED
   vrf RED
   ip address virtual 10.10.10.1/24

interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vrf RED vni 50100

router bgp 65101
   vlan 10
      rd 11.11.11.11:10010
      route-target both 10010:10010
      redistribute learned
      redistribute learned remote

   address-family ipv4
      network 11.11.11.11/32
      network 100.100.100.1/32

   address-family evpn
      neighbor 10.0.0.0 activate
      neighbor 10.0.0.6 activate

   vrf RED
      rd 11.11.11.11:50100
      route-target import evpn 50100:50100
      route-target export evpn 50100:50100
      redistribute connected
```

### Leaf2 - VRF BLUE

```eos
vlan 20
   name HOST2_BLUE

vrf instance BLUE

interface Ethernet3
   description TO_HOST2
   switchport access vlan 20
   spanning-tree portfast

interface Vlan20
   description HOST2_GW_BLUE
   vrf BLUE
   ip address virtual 10.20.20.1/24

interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 20 vni 10020
   vxlan vrf BLUE vni 50200

router bgp 65102
   vlan 20
      rd 22.22.22.22:10020
      route-target both 10020:10020
      redistribute learned
      redistribute learned remote

   address-family ipv4
      network 22.22.22.22/32
      network 100.100.100.2/32

   address-family evpn
      neighbor 10.0.0.2 activate
      neighbor 10.0.0.8 activate

   vrf BLUE
      rd 22.22.22.22:50200
      route-target import evpn 50200:50200
      route-target export evpn 50200:50200
      redistribute connected
```

### Leaf3 - Border Leaf

```eos
vlan 100
   name RED_TRANSIT_TO_R1

vlan 200
   name BLUE_TRANSIT_TO_R1

vrf instance RED
vrf instance BLUE

interface Ethernet1
   description TO_SPINE2_E2
   no switchport
   ip address 10.0.0.11/31

interface Ethernet2
   description TO_SPINE1_E3
   no switchport
   ip address 10.0.0.5/31

interface Ethernet3
   description TRUNK_TO_R1_G0_0
   switchport mode trunk
   switchport trunk allowed vlan 100,200

interface Vlan100
   description RED_TRANSIT_TO_R1
   vrf RED
   ip address 172.16.100.1/30

interface Vlan200
   description BLUE_TRANSIT_TO_R1
   vrf BLUE
   ip address 172.16.200.1/30

interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vrf RED vni 50100
   vxlan vrf BLUE vni 50200

ip routing
ip routing vrf RED
ip routing vrf BLUE

ip route vrf RED 10.20.20.0/24 172.16.100.2
ip route vrf BLUE 10.10.10.0/24 172.16.200.2

router bgp 65103
   router-id 33.33.33.33

   address-family ipv4
      network 33.33.33.33/32
      network 100.100.100.3/32

   address-family evpn
      neighbor 10.0.0.4 activate
      neighbor 10.0.0.10 activate

   vrf RED
      rd 33.33.33.33:50100
      route-target import evpn 50100:50100
      route-target export evpn 50100:50100
      redistribute connected
      redistribute static

   vrf BLUE
      rd 33.33.33.33:50200
      route-target import evpn 50200:50200
      route-target export evpn 50200:50200
      redistribute connected
      redistribute static
```

### R1 - Cisco C7200

```cisco
hostname R1

ip cef
ip routing

interface GigabitEthernet0/0
 description TRUNK_TO_LEAF3_E3
 no ip address
 no shutdown

interface GigabitEthernet0/0.100
 description RED_TRANSIT_TO_LEAF3
 encapsulation dot1Q 100
 ip address 172.16.100.2 255.255.255.252
 no shutdown

interface GigabitEthernet0/0.200
 description BLUE_TRANSIT_TO_LEAF3
 encapsulation dot1Q 200
 ip address 172.16.200.2 255.255.255.252
 no shutdown

ip route 10.10.10.0 255.255.255.0 172.16.100.1
ip route 10.20.20.0 255.255.255.0 172.16.200.1
```

## Проверка

### Leaf1 - EVPN summary

```text
Leaf1#show bgp evpn summary

Neighbor V AS           State   PfxRcd
10.0.0.0 4 65000        Estab   8
10.0.0.6 4 65001        Estab   8
```

EVPN-соседства Leaf1 со Spine1 и Spine2 находятся в состоянии `Established`.

### Leaf1 - маршруты VRF RED

```text
Leaf1#show ip route vrf RED

C        10.10.10.0/24
         directly connected, Vlan10

B E      10.20.20.0/24
         via VTEP 100.100.100.3 VNI 50100

B E      172.16.100.0/30
         via VTEP 100.100.100.3 VNI 50100
```

Leaf1 имеет локальную сеть Host1 и получает маршрут к сети Host2 через Border Leaf `Leaf3`.

### Leaf1 - VXLAN VNI

```text
Leaf1#show vxlan vni

VNI         VLAN       Source       Interface       802.1Q Tag
10010       10         static       Ethernet3       untagged
                                    Vxlan1          10

VNI         VLAN       VRF       Source
50100       4097       RED       evpn
```

На Leaf1 настроены L2VNI `10010` и L3VNI `50100`.

### Leaf2 - EVPN summary

```text
Leaf2#show bgp evpn summary

Neighbor V AS           State   PfxRcd
10.0.0.2 4 65000        Estab   8
10.0.0.8 4 65001        Estab   8
```

EVPN-соседства Leaf2 со Spine1 и Spine2 находятся в состоянии `Established`.

### Leaf2 - маршруты VRF BLUE

```text
Leaf2#show ip route vrf BLUE

B E      10.10.10.0/24
         via VTEP 100.100.100.3 VNI 50200

C        10.20.20.0/24
         directly connected, Vlan20

B E      172.16.200.0/30
         via VTEP 100.100.100.3 VNI 50200
```

Leaf2 имеет локальную сеть Host2 и получает маршрут к сети Host1 через Border Leaf `Leaf3`.

### Leaf2 - VXLAN VNI

```text
Leaf2#show vxlan vni

VNI         VLAN       Source       Interface       802.1Q Tag
10020       20         static       Ethernet3       untagged
                                    Vxlan1          20

VNI         VLAN       VRF        Source
50200       4097       BLUE       evpn
```

На Leaf2 настроены L2VNI `10020` и L3VNI `50200`.

### Leaf3 - EVPN summary

```text
Leaf3#show bgp evpn summary

Neighbor  V AS           State   PfxRcd
10.0.0.4  4 65000        Estab   8
10.0.0.10 4 65001        Estab   8
```

Leaf3 имеет рабочие EVPN-соседства со Spine1 и Spine2.

### Leaf3 - маршруты VRF RED

```text
Leaf3#show ip route vrf RED

B E      10.10.10.10/32
         via VTEP 100.100.100.1 VNI 50100

B E      10.10.10.0/24
         via VTEP 100.100.100.1 VNI 50100

S        10.20.20.0/24
         via 172.16.100.2, Vlan100

C        172.16.100.0/30
         directly connected, Vlan100
```

В VRF `RED` сеть Host1 получена через EVPN от Leaf1, а маршрут к сети BLUE указывает на R1 через `172.16.100.2`.

### Leaf3 - маршруты VRF BLUE

```text
Leaf3#show ip route vrf BLUE

S        10.10.10.0/24
         via 172.16.200.2, Vlan200

B E      10.20.20.10/32
         via VTEP 100.100.100.2 VNI 50200

B E      10.20.20.0/24
         via VTEP 100.100.100.2 VNI 50200

C        172.16.200.0/30
         directly connected, Vlan200
```

В VRF `BLUE` сеть Host2 получена через EVPN от Leaf2, а маршрут к сети RED указывает на R1 через `172.16.200.2`.

### Leaf3 - EVPN IP Prefix routes

```text
Leaf3#show bgp evpn route-type ip-prefix ipv4

* > RD: 11.11.11.11:50100 ip-prefix 10.10.10.0/24
    Next Hop: 100.100.100.1

* > RD: 33.33.33.33:50200 ip-prefix 10.10.10.0/24

* > RD: 22.22.22.22:50200 ip-prefix 10.20.20.0/24
    Next Hop: 100.100.100.2

* > RD: 33.33.33.33:50100 ip-prefix 10.20.20.0/24

* > RD: 33.33.33.33:50100 ip-prefix 172.16.100.0/30

* > RD: 33.33.33.33:50200 ip-prefix 172.16.200.0/30
```

EVPN Type-5 маршруты находятся в состоянии `valid` и `best`.

### Leaf3 - VXLAN VNI

```text
Leaf3#show vxlan vni

VNI         VLAN       VRF        Source
50100       4097       RED        evpn
50200       4098       BLUE       evpn
```

На Border Leaf настроены L3VNI для обоих VRF.

## Проверка R1

### Интерфейсы R1

```text
R1#show ip interface brief

Interface              IP-Address      Status  Protocol
GigabitEthernet0/0     unassigned      up      up
GigabitEthernet0/0.100 172.16.100.2    up      up
GigabitEthernet0/0.200 172.16.200.2    up      up
```

R1 подключен к Leaf3 через trunk. Subinterface `.100` используется для VRF `RED`, subinterface `.200` - для VRF `BLUE`.

### Маршруты R1

```text
R1#show ip route

S 10.10.10.0/24 via 172.16.100.1
S 10.20.20.0/24 via 172.16.200.1

C 172.16.100.0/30 is directly connected, GigabitEthernet0/0.100
C 172.16.200.0/30 is directly connected, GigabitEthernet0/0.200
```

R1 имеет статические маршруты к обеим клиентским сетям через соответствующие SVI Leaf3.

### Проверка R1 - Leaf3

```text
R1#ping 172.16.100.1

Success rate is 100 percent (5/5)
```

R1 успешно пингует Leaf3 в RED transit-сети.

## Проверка Host1

### IP-адресация Host1

```text
Host1

ens3:
  inet 10.10.10.10/24

default via 10.10.10.1 dev ens3
10.10.10.0/24 dev ens3
```

### Host1 - gateway RED

```text
Host1$ ping -c 5 10.10.10.1

5 packets transmitted, 5 received, 0% packet loss
```

### Host1 - R1 RED transit

```text
Host1$ ping -c 5 172.16.100.2

5 packets transmitted, 5 received, 0% packet loss
```

### Host1 - Host2

```text
Host1$ ping -c 5 10.20.20.10

5 packets transmitted, 5 received, 0% packet loss
```

Host1 успешно пингует Host2, который находится в другом VRF.

## Проверка Host2

### IP-адресация Host2

```text
Host2

ens3:
  inet 10.20.20.10/24

default via 10.20.20.1 dev ens3
10.20.20.0/24 dev ens3
```

### Host2 - gateway BLUE

```text
Host2$ ping -c 5 10.20.20.1

5 packets transmitted, 5 received, 0% packet loss
```

### Host2 - R1 BLUE transit

```text
Host2$ ping -c 5 172.16.200.2

5 packets transmitted, 5 received, 0% packet loss
```

### Host2 - Host1

```text
Host2$ ping -c 5 10.10.10.10

5 packets transmitted, 5 received, 0% packet loss
```

Host2 успешно пингует Host1, который находится в другом VRF.

## Подтверждение маршрутизации через внешнее устройство

В лабораторной работе `traceroute` не использовался как основной способ проверки. Прохождение трафика через внешний маршрутизатор подтверждается таблицами маршрутизации Leaf3 и R1.

На Leaf3:

```text
VRF RED:
S 10.20.20.0/24 via 172.16.100.2

VRF BLUE:
S 10.10.10.0/24 via 172.16.200.2
```

Это означает, что Border Leaf не выполняет прямой route leaking между `RED` и `BLUE`, а отправляет межVRF-трафик на R1.

На R1:

```text
S 10.10.10.0/24 via 172.16.100.1
S 10.20.20.0/24 via 172.16.200.1
```

R1 возвращает трафик в соответствующие VRF через transit-интерфейсы Leaf3.

End-to-end ping между Host1 и Host2 проходит успешно в обе стороны, что подтверждает корректную работу межVRF-маршрутизации через внешнее устройство.

## Итог

В результате лабораторной работы:

* настроена VXLAN EVPN фабрика на Arista vEOS;
* настроен eBGP underlay;
* настроен EVPN control-plane;
* настроены VRF `RED` и `BLUE`;
* настроены L3VNI `50100` и `50200`;
* Host1 размещен в VRF `RED`;
* Host2 размещен в VRF `BLUE`;
* Leaf3 настроен как Border Leaf;
* Cisco R1 настроен как внешнее устройство для маршрутизации между VRF;
* подтверждена связность Host1 - Host2 через внешний маршрутизатор.

Итоговая схема соответствует требованиям задания: два клиента размещены в разных VRF одной фабрики, а маршрутизация между ними выполняется через внешнее устройство.
