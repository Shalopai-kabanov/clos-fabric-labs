# Лабораторная работа №3 - IS-IS Underlay

## Цель работы

Настроить IS-IS underlay сеть в топологии Spine-Leaf для обеспечения IP связности между всеми сетевыми устройствами.

---

## План работы

1. Построить топологию Spine-Leaf
2. Настроить L3 интерфейсы между устройствами
3. Назначить IP адресацию
4. Включить IP routing
5. Настроить IS-IS
6. Проверить IS-IS соседства
7. Проверить IP связность между Loopback интерфейсами

---

## Схема сети

![Topology](Topology.jpg)

---

## Адресное пространство

### Loopback интерфейсы

| Устройство | Интерфейс | IP адрес |
|---|---|---|
| Spine1 | Loopback0 | 1.1.1.1/32 |
| Spine2 | Loopback0 | 2.2.2.2/32 |
| Leaf1 | Loopback0 | 11.11.11.11/32 |
| Leaf2 | Loopback0 | 22.22.22.22/32 |
| Leaf3 | Loopback0 | 33.33.33.33/32 |
| Leaf4 | Loopback0 | 44.44.44.44/32 |

---

### P2P соединения

| Устройство | Интерфейс | IP адрес |
|---|---|---|
| Spine1 | Ethernet1 | 10.0.0.0/31 |
| Leaf1 | Ethernet1 | 10.0.0.1/31 |
| Spine1 | Ethernet2 | 10.0.0.2/31 |
| Leaf2 | Ethernet1 | 10.0.0.3/31 |
| Spine1 | Ethernet3 | 10.0.0.4/31 |
| Leaf3 | Ethernet1 | 10.0.0.5/31 |
| Spine1 | Ethernet4 | 10.0.0.6/31 |
| Leaf4 | Ethernet1 | 10.0.0.7/31 |
| Spine2 | Ethernet1 | 10.0.0.8/31 |
| Leaf1 | Ethernet2 | 10.0.0.9/31 |
| Spine2 | Ethernet2 | 10.0.0.10/31 |
| Leaf2 | Ethernet2 | 10.0.0.11/31 |
| Spine2 | Ethernet3 | 10.0.0.12/31 |
| Leaf3 | Ethernet2 | 10.0.0.13/31 |
| Spine2 | Ethernet4 | 10.0.0.14/31 |
| Leaf4 | Ethernet2 | 10.0.0.15/31 |

---

## NET адреса IS-IS

| Устройство | NET |
|---|---|
| Spine1 | 49.0001.0000.0000.0001.00 |
| Spine2 | 49.0001.0000.0000.0002.00 |
| Leaf1 | 49.0001.0000.0000.0011.00 |
| Leaf2 | 49.0001.0000.0000.0022.00 |
| Leaf3 | 49.0001.0000.0000.0033.00 |
| Leaf4 | 49.0001.0000.0000.0044.00 |

---

## Конфигурация устройств

Конфигурации устройств расположены в директории:

```text
configs/
```

### Список конфигураций

- Spine1.cfg
- Spine2.cfg
- Leaf1.cfg
- Leaf2.cfg
- Leaf3.cfg
- Leaf4.cfg

---

## Проверка IS-IS соседства

### Spine1

```bash
Spine1#show isis neighbors

Instance VRF      System Id  Type Interface  SNPA State Hold time Circuit Id
FABRIC  default  Leaf1       L2   Ethernet1  P2P  UP    25        17
FABRIC  default  Leaf2       L2   Ethernet2  P2P  UP    24        15
FABRIC  default  Leaf3       L2   Ethernet3  P2P  UP    23        17
FABRIC  default  Leaf4       L2   Ethernet4  P2P  UP    23        19
```

---

## Проверка таблицы маршрутизации

```bash
Spine1#show ip route isis
```

Пример ECMP маршрута:

```bash
I L2 2.2.2.2/32 [115/30]
 via 10.0.0.1, Ethernet1
 via 10.0.0.3, Ethernet2
 via 10.0.0.5, Ethernet3
 via 10.0.0.7, Ethernet4
```

---

## Проверка IP связности

### Ping между Loopback интерфейсами

```bash
Leaf2#ping 44.44.44.44 source loopback0

PING 44.44.44.44 (44.44.44.44) from 22.22.22.22 : 72(100) bytes of data.
80 bytes from 44.44.44.44: icmp_seq=1 ttl=62 time=1.23 ms
80 bytes from 44.44.44.44: icmp_seq=2 ttl=62 time=1.11 ms
80 bytes from 44.44.44.44: icmp_seq=3 ttl=62 time=0.98 ms

--- 44.44.44.44 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss
```

---

## Вывод

В рамках лабораторной работы была построена Spine-Leaf топология и настроен IS-IS underlay.

Между всеми устройствами установлены IS-IS соседства уровня L2. Таблица маршрутизации содержит маршруты ко всем Loopback интерфейсам, а также ECMP маршруты с несколькими равнозначными next-hop.

В отличие от OSPF, IS-IS активируется непосредственно на интерфейсах и использует NET address для идентификации маршрутизатора в домене IS-IS.
