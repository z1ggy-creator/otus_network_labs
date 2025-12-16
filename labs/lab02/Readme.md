# Underlay. OSPF
## Настроить OSPF для Underlay сети.

---

## 1. План работ  

### Этап 1: Настройка Underlay сети на OSPF
- [ ] Настройка OSPF  
- [ ] Настройка BFD  
- [ ] Проверка смежности OSPF  

### Этап 2: Тестирование и проверка  
- [ ] Проверка связности между всеми Loopback адресами   
- [ ] Проверка таблиц маршрутизации  
- [ ] Тестирование отказоустойчивости  
- [ ] Документирование конфигураций  

---

## 2. Адресное пространство  

### 2.1. Loopback интерфейсы  
**Формат:** `10.255.255.X/32`  

| Устройство | IP-адрес       | Router ID      |
|------------|----------------|----------------|
| Spine-1    | 10.255.255.1/32  | 10.255.255.1    |
| Spine-2    | 10.255.255.2/32  | 10.255.255.2    |
| Leaf-1     | 10.255.255.11/32 | 10.255.255.11   |
| Leaf-2     | 10.255.255.12/32 | 10.255.255.12   |
| Leaf-3     | 10.255.255.13/32 | 10.255.255.13   |

### 2.2. Point-to-Point линки  
**Подсеть:** `10.0.0.0/24`  
**Маска:** `/31`  

| Соединение        | Подсеть       | Устройство | IP-адрес     |
|-------------------|---------------|------------|--------------|
| Spine-1 ↔ Leaf-1  | 10.0.0.0/31   | Spine-1    | 10.0.0.0/31  |
|                   |               | Leaf-1     | 10.0.0.1/31  |
| Spine-1 ↔ Leaf-2  | 10.0.0.2/31   | Spine-1    | 10.0.0.2/31  |
|                   |               | Leaf-2     | 10.0.0.3/31  |
| Spine-1 ↔ Leaf-3  | 10.0.0.4/31   | Spine-1    | 10.0.0.4/31  |
|                   |               | Leaf-3     | 10.0.0.5/31  |
| Spine-2 ↔ Leaf-1  | 10.0.0.6/31   | Spine-2    | 10.0.0.6/31  |
|                   |               | Leaf-1     | 10.0.0.7/31  |
| Spine-2 ↔ Leaf-2  | 10.0.0.8/31   | Spine-2    | 10.0.0.8/31  |
|                   |               | Leaf-2     | 10.0.0.9/31  |
| Spine-2 ↔ Leaf-3  | 10.0.0.10/31  | Spine-2    | 10.0.0.10/31 |
|                   |               | Leaf-3     | 10.0.0.11/31 |

---

## 3. Схема Underlay сети на OSPF  

### 3.1. Топология  

![Topology_lab02.png](Topology_lab02.png)


### 3.2. Параметры OSPF 

#### Общие настройки:  
- **OSPF Process:** `1`  
- **Router ID:** Соответствует Loopback адресу  
- **Все интерфейсы:** Area 0 (Backbone)  
- **Таймеры:**  
  - Hello-interval: `1 сек` (для Point-to-Point)  
  - Dead-interval: `4 сек`  
  - BFD: Включен 
- **Все линки Spine-Leaf:** Point-to-Point  
- **Loopback интерфейсы:** Passive  

### 3.3. Таблица интерфейсов и OSPF настроек  

| Устройство | Интерфейс | Назначение | IP адрес       | OSPF настройки                    |
|------------|-----------|------------|----------------|-----------------------------------|
| **Spine-1**| Eth1/1    | К Leaf-1   | 10.0.0.0/31    | OSPF area 0, network point-to-point |
|            | Eth1/2    | К Leaf-2   | 10.0.0.2/31    | OSPF area 0, network point-to-point |
|            | Eth1/3    | К Leaf-3   | 10.0.0.4/31    | OSPF area 0, network point-to-point |
|            | Lo0       | Loopback   | 10.255.255.1/32| OSPF area 0, passive               |
| **Spine-2**| Eth1/1    | К Leaf-1   | 10.0.0.6/31    | OSPF area 0, network point-to-point |
|            | Eth1/2    | К Leaf-2   | 10.0.0.8/31    | OSPF area 0, network point-to-point |
|            | Eth1/3    | К Leaf-3   | 10.0.0.10/31   | OSPF area 0, network point-to-point |
|            | Lo0       | Loopback   | 10.255.255.2/32| OSPF area 0, passive               |
| **Leaf-1** | Eth1/1    | К Spine-1  | 10.0.0.1/31    | OSPF area 0, network point-to-point |
|            | Eth1/2    | К Spine-2  | 10.0.0.7/31    | OSPF area 0, network point-to-point |
|            | Lo0       | Loopback   | 10.255.255.11/32| OSPF area 0, passive              |
| **Leaf-2** | Eth1/1    | К Spine-1  | 10.0.0.3/31    | OSPF area 0, network point-to-point |
|            | Eth1/2    | К Spine-2  | 10.0.0.9/31    | OSPF area 0, network point-to-point |
|            | Lo0       | Loopback   | 10.255.255.12/32| OSPF area 0, passive              |
| **Leaf-3** | Eth1/1    | К Spine-1  | 10.0.0.5/31    | OSPF area 0, network point-to-point |
|            | Eth1/2    | К Spine-2  | 10.0.0.11/31   | OSPF area 0, network point-to-point |
|            | Lo0       | Loopback   | 10.255.255.13/32| OSPF area 0, passive              |

### 3.4. Дополнительные параметры  
- **Cost метрика:**   
- **Fast Convergence:**  
  - BFD с интервалом 50ms x 3  
  - OSPF LSA throttling отключить  

---
### 4. Конфигурация OSPF на устройствах 

### 4.1. SPINE-1 
```
SPINE-1#show running-config section ospf
interface Ethernet1
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
interface Ethernet2
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
interface Ethernet3
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
interface Loopback0
   ip ospf area 0.0.0.0
router ospf 1
   router-id 10.255.255.1
   bfd default
   passive-interface Loopback0
   max-lsa 12000
```
### 4.2. SPINE-2
```


```
### 4.3. LEAF-1
```


```
### 4.4. LEAF-2
```


```
### 4.5. LEAF-3
```


```





### 5. Проверка состояния OSPF 
