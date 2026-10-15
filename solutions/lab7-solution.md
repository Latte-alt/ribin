# Лабораторная работа №7 — IP-телефония между филиалами

[Методичка](../%D0%BB%D0%B0%D0%B1%207.docx). Cisco Packet Tracer. COR здесь не требуется.

## 1. Подготовка

Устройства: **2 × Cisco 2811**, **2 × Cisco 2950T**, **6 × IP Phone 7960**, **2 × PC-PT**. Если модель коммутатора не поддерживает `switchport voice vlan`, использовать **2960-24TT**.

На каждом маршрутизаторе: **Physical → выключить питание → установить WIC-2T в слот 3 → включить питание**. В командах используется **Serial0/3/0**. Проверить имя через `show ip interface brief`; если оно другое, заменить во всех блоках.

| Откуда | Куда | Кабель |
| --- | --- | --- |
| RouterA Fa0/0 | SwitchA Fa0/1 | Copper Straight-Through |
| RouterB Fa0/0 | SwitchB Fa0/1 | Copper Straight-Through |
| RouterA S0/3/0 | RouterB S0/3/0 | Serial DCE; первым выбрать RouterA |
| Три телефона филиала A, порт Switch | SwitchA Fa0/2, Fa0/3, Fa0/4 | Copper Straight-Through |
| Три телефона филиала B, порт Switch | SwitchB Fa0/2, Fa0/3, Fa0/4 | Copper Straight-Through |
| PC0 FastEthernet0 | SwitchA Fa0/5 | Copper Straight-Through |
| PC1 FastEthernet0 | SwitchB Fa0/5 | Copper Straight-Through |

Телефоны → **Physical** → установить **IP_PHONE_POWER_ADAPTER**. ПК подключаются непосредственно к коммутаторам, как на рисунке методички.

Использовать новые устройства. Маршрутизатор → **CLI** → на вопрос `initial configuration dialog` ответить **no**.

## 2. SwitchA

```text
enable
configure terminal
hostname SwitchA
no ip domain-lookup

interface fastEthernet0/1
switchport mode access
switchport access vlan 1
no shutdown
exit

interface range fastEthernet0/2 - 4
switchport mode access
switchport access vlan 1
switchport voice vlan 1
spanning-tree portfast
no shutdown
exit

interface fastEthernet0/5
switchport mode access
switchport access vlan 1
spanning-tree portfast
no shutdown
exit

end
copy running-config startup-config
```

На запрос имени файла — **Enter**.

## 3. SwitchB

```text
enable
configure terminal
hostname SwitchB
no ip domain-lookup

interface fastEthernet0/1
switchport mode access
switchport access vlan 1
no shutdown
exit

interface range fastEthernet0/2 - 4
switchport mode access
switchport access vlan 1
switchport voice vlan 1
spanning-tree portfast
no shutdown
exit

interface fastEthernet0/5
switchport mode access
switchport access vlan 1
spanning-tree portfast
no shutdown
exit

end
copy running-config startup-config
```

## 4. RouterA

```text
enable
configure terminal
hostname RouterA
no ip domain-lookup

interface fastEthernet0/0
ip address 192.168.1.1 255.255.255.224
no shutdown
exit

interface serial0/3/0
ip address 10.0.1.1 255.255.255.252
encapsulation hdlc
clock rate 64000
no shutdown
exit

ip dhcp excluded-address 192.168.1.1 192.168.1.9
ip dhcp pool T1
network 192.168.1.0 255.255.255.224
default-router 192.168.1.1
option 150 ip 192.168.1.1
exit

router rip
version 2
no auto-summary
network 192.168.1.0
network 10.0.0.0
passive-interface fastEthernet0/0
exit

telephony-service
max-dn 5
max-ephones 5
ip source-address 192.168.1.1 port 2000
auto assign 1 to 3
exit

ephone-dn 1
number 1101
exit
ephone-dn 2
number 1102
exit
ephone-dn 3
number 1103
exit

dial-peer voice 1 voip
destination-pattern 12..
session target ipv4:10.0.1.2
exit

end
copy running-config startup-config
```

## 5. RouterB

```text
enable
configure terminal
hostname RouterB
no ip domain-lookup

interface fastEthernet0/0
ip address 172.16.1.1 255.255.255.224
no shutdown
exit

interface serial0/3/0
ip address 10.0.1.2 255.255.255.252
encapsulation hdlc
no shutdown
exit

ip dhcp excluded-address 172.16.1.1 172.16.1.9
ip dhcp pool T2
network 172.16.1.0 255.255.255.224
default-router 172.16.1.1
option 150 ip 172.16.1.1
exit

router rip
version 2
no auto-summary
network 172.16.0.0
network 10.0.0.0
passive-interface fastEthernet0/0
exit

telephony-service
max-dn 5
max-ephones 5
ip source-address 172.16.1.1 port 2000
auto assign 1 to 3
exit

ephone-dn 1
number 1201
exit
ephone-dn 2
number 1202
exit
ephone-dn 3
number 1203
exit

dial-peer voice 1 voip
destination-pattern 11..
session target ipv4:10.0.1.1
exit

end
copy running-config startup-config
```

`clock rate` задаётся только на стороне **DCE**, в этой схеме на RouterA. Если кабель установлен наоборот, перенести эту команду на фактическую сторону DCE.

## 6. ПК и телефоны

- PC0 и PC1 → **Desktop → IP Configuration → DHCP**.
- Дождаться загрузки телефонов в **Realtime**; при необходимости нажать **Fast Forward Time**.
- Открыть **GUI** телефонов и проверить номера.

| Филиал | Номера телефонов | IP телефонов и ПК | Шлюз |
| --- | --- | --- | --- |
| A | 1101, 1102, 1103 | 192.168.1.10–30, маска 255.255.255.224 | 192.168.1.1 |
| B | 1201, 1202, 1203 | 172.16.1.10–30, маска 255.255.255.224 | 172.16.1.1 |

Номера назначаются по порядку регистрации, а не расположению телефонов. MAC-адреса вручную подставлять не нужно.

## 7. Проверка сети и RIP

### RouterA

```text
show ip interface brief
show ip protocols
show ip route
show ip route rip
show ip dhcp binding
show ephone
show dial-peer voice summary
ping 10.0.1.2
ping 172.16.1.1
```

### RouterB

```text
show ip interface brief
show ip protocols
show ip route
show ip route rip
show ip dhcp binding
show ephone
show dial-peer voice summary
ping 10.0.1.1
ping 192.168.1.1
```

Ожидается:
- Fa0/0 и Serial0/3/0 — **up/up** на обоих маршрутизаторах.
- На RouterA есть RIP-маршрут **172.16.1.0/27**, на RouterB — **192.168.1.0/27**, с обозначением **R**.
- В каждом DHCP-пуле выданы адреса трём телефонам и ПК.
- Телефоны зарегистрированы на своём маршрутизаторе.

Дождаться обмена RIP-маршрутами, если они появились не сразу.

### Оба коммутатора

```text
show interfaces status
show vlan brief
show interfaces fastEthernet0/2 switchport
```

### ПК — Desktop → Command Prompt

На обоих ПК:

```text
ipconfig /all
```

На PC0:

```text
ping 192.168.1.1
ping 172.16.1.1
```

На PC1:

```text
ping 172.16.1.1
ping 192.168.1.1
```

Затем с PC0 выполнить `ping` на фактический адрес PC1 и наоборот. Адреса взять из `ipconfig`; DHCP может выдать их в разном порядке.

## 8. Проверка звонков

Телефон → **GUI** → снять трубку → набрать номер → ответить на принимающем телефоне.

Проверить по очереди, завершая каждый разговор:

| Проверка | Назначение |
| --- | --- |
| 1101 → 1102 | Внутри филиала A |
| 1201 → 1202 | Внутри филиала B |
| 1101 → 1201 | A → B |
| 1102 → 1202 | A → B |
| 1103 → 1203 | A → B |
| 1201 → 1101 | B → A |

Если локальные звонки работают, а удалённые нет: проверить RIP-маршруты, ping удалённого LAN-интерфейса, `destination-pattern` и `session target`.

Если доступны дополнительные функции:
- перевод: разговор **1101 ↔ 1201** → на 1201 **Transfer → 1202 → Transfer**;
- конференция: разговор **1101 ↔ 1201** → **Confrn → 1202 → ответ → Confrn**;
- перехват: **1101 звонит 1201**, на 1201 не отвечать → на 1202 **PickUp → 1201**, при необходимости **Answer**.

Поддержку этих функций проверять отдельно: базовые команды из методички не гарантируют их работу в каждой версии симулятора.

## 9. Если Serial не поднимается

На обоих маршрутизаторах:

```text
show controllers serial 0/3/0
show interfaces serial0/3/0
```

Проверить:
- одинаковую инкапсуляцию **HDLC**;
- адреса **10.0.1.1/30** и **10.0.1.2/30**;
- `no shutdown` с обеих сторон;
- `clock rate` на стороне DCE.

### Если нужна большая полоса

В основном решении оставлено **64000 бит/с**, как в методичке. Такая полоса ограничивает голосовой трафик: для G.711 с заголовками её недостаточно. Для G.711 или нескольких одновременных разговоров можно увеличить скорость на RouterA (DCE):

```text
configure terminal
interface serial0/3/0
clock rate 2000000
exit
end
copy running-config startup-config
```

Если требуется явно выбрать одинаковый кодек G.711, на **обоих** маршрутизаторах, после увеличения скорости:

```text
configure terminal
dial-peer voice 1 voip
codec g711ulaw
exit
end
copy running-config startup-config
```

Команда `bandwidth` сама по себе не меняет физическую скорость Serial.

## 10. Сохранение

На обоих маршрутизаторах и коммутаторах:

```text
copy running-config startup-config
```

На запрос имени файла — **Enter**. **File → Save As → Лабораторная_7.pkt**.

Исправления методички: добавлено `no auto-summary`; RIP `network` указан для основных классовых сетей; вместо пересекающихся диапазонов автоматического назначения выбран **1–3**, по числу созданных линий.

[Контрольные вопросы и ответы](labs-control-answers.md#лабораторная-работа-7).
