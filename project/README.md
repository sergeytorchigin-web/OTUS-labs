# 1 Цель проектной работы

Спроектировать геораспределённую сеть двух центров обработки данных, расположенных в разных городах. Каждый ЦОД представляет собой независимую Leaf-Spine EVPN-VXLAN-фабрику. Между площадками организовать отказоустойчивое L3 DCI-соединение через два независимых маршрутизатора. Обеспечить L2-связность внутри сегментов, межсегментную маршрутизацию внутри POD и IP-маршрутизацию между двумя POD.

Результат работы

Внутри каждого POD должны работать:

eBGP Underlay;
ECMP между Leaf и Spine;
eBGP EVPN Overlay;
VXLAN;
L2VNI;
L3VNI;
Anycast Gateway;
EVPN Route Type 2, 3 и 5.

Между POD должны работать:

два независимых L3 DCI-пути;
IPv4 eBGP в VRF TENANT;
передача только клиентских IP-префиксов;
отсутствие растягивания VLAN между городами;
автоматическое переключение при отказе одного DCI-пути.

# 2. Архитектура сети

Используется двухуровневая CLOS архитектура:
<img width="2416" height="869" alt="image" src="https://github.com/user-attachments/assets/903f4d35-4400-415e-a96c-8716312eeafe" />

## 2.1 Функции устройств

| Устройство          | Назначение                                                   |
| ------------------- | ------------------------------------------------------------ |
| **Spine**           | Underlay-маршрутизация и EVPN Route Server                   |
| **Leaf**            | VTEP, шлюз VLAN, поддержка L2VNI и L3VNI                     |
| **Leaf1 / Leaf2**   | Дополнительно выполняют роль Border Leaf для организации DCI |
| **DCI-R1 / DCI-R2** | Обычная IPv4-маршрутизация между городами                    |
| **Host**            | Клиентские устройства                                        |

## 2.2 Роль DCI-R1 и DCI-R2

Маршрутизаторы **DCI-R1** и **DCI-R2**:

* не являются VTEP;
* не имеют интерфейса `Vxlan1`;
* не участвуют в EVPN;
* не передают MAC-маршруты;
* передают только IPv4-префиксы VRF `TENANT`.

## 2.3 Overlay и клиентские сети

### Общие параметры

| Параметр                | Значение            |
| ----------------------- | ------------------- |
| **VRF**                 | `TENANT`            |
| **L3VNI**               | `50000`             |
| **Anycast Gateway MAC** | `00:1c:73:00:00:01` |

### POD1

| VLAN |   L2VNI | Клиентская сеть  | Шлюз          | Расположение  |
| ---: | ------: | ---------------- | ------------- | ------------- |
| `10` | `10010` | `192.168.1.0/24` | `192.168.1.1` | Leaf1 и Leaf3 |
| `20` | `10020` | `192.168.2.0/24` | `192.168.2.1` | Leaf2         |
| `30` | `10030` | `192.168.3.0/24` | `192.168.3.1` | Leaf3         |

VLAN `10` размещён одновременно на **Leaf1** и **Leaf3**. Это позволяет отдельно проверить передачу L2-трафика через VXLAN между двумя VTEP в пределах одного POD.

### POD2

| VLAN |   L2VNI | Клиентская сеть  | Шлюз          | Расположение        |
| ---: | ------: | ---------------- | ------------- | ------------------- |
| `40` | `10040` | `192.168.4.0/24` | `192.168.4.1` | Leaf1_P2 и Leaf3_P2 |
| `50` | `10050` | `192.168.5.0/24` | `192.168.5.1` | Leaf2_P2            |
| `60` | `10060` | `192.168.6.0/24` | `192.168.6.1` | Leaf3_P2            |

VLAN `40` размещён одновременно на **Leaf1_P2** и **Leaf3_P2** для проверки L2 VXLAN-связности между Leaf-коммутаторами второго POD.
 
## 3. План работ
В рамках работы предстоит:

- Создать физическую топологию и проверить состояние Ethernet-интерфейсов.
- Настроить Loopback0 и eBGP Underlay.
- Проверить доступность Loopback всех Leaf и Spine внутри каждого POD.
- Настроить eBGP EVPN Overlay.
- Настроить VRF, VLAN, L2VNI, L3VNI и Anycast Gateway.
- Настроить клиентские порты и адреса хостов.
- Проверить L2 и L3-связность внутри каждого POD.
- Настроить DCI-R1 и DCI-R2.
- Настроить eBGP в VRF TENANT.
- Oграничить маршруты prefix-list и route-map.
- Проверить межгородскую связность.
- Проверить отказ DCI-R1, DCI-R2, одного Spine и одного Underlay-линка.


## 3.1 Этап 1. Underlay

На каждом Leaf:

show ip bgp summary
show ip route

## 3.2 Этап 2. EVPN
show bgp evpn summary

На каждом Leaf должны быть две EVPN-сессии — к двум Spine своего POD.

## 3.3 Этап 3. VXLAN

show interfaces Vxlan1
show vxlan vtep
show vxlan vni
show vxlan config-sanity detail

## 3.4 Этап 4. L2VNI

С Host1:

ping 192.168.1.20

Трафик должен пройти через VNI 10010 от Leaf1 к Leaf3.

Проверка маршрутов:

show bgp evpn route-type mac-ip
show vxlan address-table

## Этап 3.5. L3VNI внутри POD

С Host1:

ping 192.168.2.10
ping 192.168.3.10

Проверка:

show ip route vrf TENANT
show bgp evpn route-type ip-prefix ipv4

## 3.6 Этап 6. DCI BGP

На Leaf1 и Leaf2:

show ip bgp vrf TENANT summary

На DCI-R1 и DCI-R2:

show ip bgp vrf TENANT summary
show ip bgp vrf TENANT
show ip route vrf TENANT

# 4 Проверка eBGP Underlay

Leaf 1

<img width="908" height="132" alt="image" src="https://github.com/user-attachments/assets/015ef83d-2cb8-4480-af19-afb50a0e48c8" />

Leaf 2

<img width="923" height="130" alt="image" src="https://github.com/user-attachments/assets/69089e02-5ea6-4fe7-8e8a-aa69e7d5f848" />


Leaf 3

<img width="924" height="130" alt="image" src="https://github.com/user-attachments/assets/d56081db-d508-4ad3-b83c-ffb54466ba88" />

Leaf 1 POD2

<img width="699" height="134" alt="image" src="https://github.com/user-attachments/assets/746e0e10-ea58-4323-bd6d-c953b40d6874" />

Leaf 2 POD2

<img width="820" height="137" alt="image" src="https://github.com/user-attachments/assets/099e75ed-33fc-41f6-bbf1-dd198105ec5f" />

Leaf 3 POD2

<img width="761" height="131" alt="image" src="https://github.com/user-attachments/assets/e45a6dd7-5807-44d3-89df-ccc0035fb131" />

# 5 Проверка MP-BGP EVPN Overlay

Leaf 1

<img width="921" height="135" alt="image" src="https://github.com/user-attachments/assets/fd266354-270b-4237-9d71-713af1810069" />

Leaf 2

<img width="934" height="124" alt="image" src="https://github.com/user-attachments/assets/b0204395-d49b-4165-a3d9-f9ec2c864cbc" />

Leaf 3

<img width="947" height="130" alt="image" src="https://github.com/user-attachments/assets/664f63a2-a507-4f33-b266-4a77390e4df9" />

Leaf 1 POD2

<img width="714" height="130" alt="image" src="https://github.com/user-attachments/assets/c2ecff57-39a5-41fc-8eda-729c396dba3c" />


Leaf 2 POD2

<img width="728" height="131" alt="image" src="https://github.com/user-attachments/assets/519e209f-66f6-4a11-a909-5c70aa6c568b" />

Leaf 3 POD2

<img width="754" height="138" alt="image" src="https://github.com/user-attachments/assets/2dce7c01-1505-414f-bae8-d4c18653f8b5" />

# 6 Проверка EVPN-соседств на Spine

Spine 1 

<img width="884" height="142" alt="image" src="https://github.com/user-attachments/assets/c4648387-26d2-44d9-8728-dfd9835db0e7" />

Spine 2

<img width="894" height="152" alt="image" src="https://github.com/user-attachments/assets/30258bb9-fb07-4fbc-8291-1c0cfd9e5f00" />

Spine 1 POD2

<img width="887" height="149" alt="image" src="https://github.com/user-attachments/assets/aff9a7fc-19dc-4ba2-97af-db5653685d22" />

Spine 2 POD2 

<img width="700" height="157" alt="image" src="https://github.com/user-attachments/assets/cdb27eab-cf7c-428f-9324-35e2fe23ece6" />

# 7 Проверка типов EVPN-маршрутов

Leaf 1

<img width="787" height="518" alt="image" src="https://github.com/user-attachments/assets/6fff0dc0-a6b2-47be-925e-3dcf86c832a2" />

Leaf 2

<img width="798" height="508" alt="image" src="https://github.com/user-attachments/assets/374131e7-a0de-4b25-87b5-9d3706ada36e" />

Leaf 3

<img width="812" height="417" alt="image" src="https://github.com/user-attachments/assets/449f31c1-5959-44cc-9333-06390d440ddd" />

Leaf 1 POD2

<img width="745" height="371" alt="image" src="https://github.com/user-attachments/assets/3090391d-c9e2-4966-9cf7-775d92dd10e6" />

Leaf 2 POD2

<img width="781" height="394" alt="image" src="https://github.com/user-attachments/assets/811ae7b9-2609-4f08-8cde-181f03ac9a8a" />

Leaf 3 POD2

<img width="759" height="354" alt="image" src="https://github.com/user-attachments/assets/bfeb4e8a-cdd0-4479-b8ef-fb6839d1f44f" />

# 8 EVPN Route Type 5 — IP Prefix

Leaf 1

<img width="763" height="376" alt="image" src="https://github.com/user-attachments/assets/55a723c5-bfad-4de5-aaf0-32f1c43906ea" />

Leaf 2

<img width="775" height="375" alt="image" src="https://github.com/user-attachments/assets/7e15a03e-cd35-4cb1-8e4b-806aef9753cc" />

Leaf 3

<img width="798" height="339" alt="image" src="https://github.com/user-attachments/assets/3c25adb6-2e9c-4655-bd95-10f13350102c" />

Leaf 1 POD2

<img width="799" height="374" alt="image" src="https://github.com/user-attachments/assets/22a83515-9af4-468e-9ace-6ad97131ae15" />

Leaf 2 POD2

<img width="903" height="395" alt="image" src="https://github.com/user-attachments/assets/2e06b4cd-04cd-428c-9889-15e362a708e9" />

Leaf 3 POD2

# 9 DCI-R1 + ping

<img width="976" height="1141" alt="image" src="https://github.com/user-attachments/assets/871d85e4-20cf-497e-befa-8d902191fb07" />

# 10 DCI-R2 + ping

<img width="961" height="1263" alt="image" src="https://github.com/user-attachments/assets/7a70a330-6cc0-4bb3-b66a-1dcf026e41d0" />

# 11 Проверка POD1 → POD2

С Host1 POD1 проверить все хосты POD2

Исходный хост:

Host1 POD1
IP: 192.168.1.10
VLAN 10
Leaf1

Выполнить на Host1:

ping -c 4 192.168.4.10

ping -c 4 192.168.5.10

ping -c 4 192.168.4.20

ping -c 4 192.168.6.10

<img width="597" height="651" alt="image" src="https://github.com/user-attachments/assets/d3a5ae74-e80a-4142-b277-7e10b21de138" />

# 12 Проверка POD2 → POD1

Исходный хост:

Host1_P2
IP: 192.168.4.10
VLAN 40
Leaf1_P2

Выполнить на Host1_P2:

ping -c 4 192.168.1.10

ping -c 4 192.168.2.10

ping -c 4 192.168.1.20

ping -c 4 192.168.3.10

<img width="550" height="492" alt="image" src="https://github.com/user-attachments/assets/bcbea583-1110-441f-a9fc-4e994b11a65e" />


# 13 Вывод

С Host1 первого POD была подтверждена доступность всех четырёх клиентских хостов второго POD. В обратном направлении с Host1_P2 подтверждена доступность всех клиентских хостов первого POD. Успешное прохождение ICMP-трафика до хостов, подключённых к непограничным Leaf3 и Leaf3_P2, подтверждает передачу трафика через локальный EVPN-VXLAN Overlay, Border Leaf, L3 DCI и Overlay удалённого POD. Двусторонняя маршрутизация между всеми клиентскими подсетями работает корректно.
