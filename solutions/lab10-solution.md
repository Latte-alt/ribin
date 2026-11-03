# Лабораторная №10 — базовая IP-телефония

[Методичка](../%D0%BB%D0%B0%D0%B1%2010.docx). Cisco Packet Tracer. Использовать новую схему.

## 1. Устройства и подключение

**2811 — 1 шт., 3560-24PS — 1 шт., IP Phone 7960 — 3 шт.** Дополнительные модули не нужны. 3560-24PS заменяет 2950 из методички и питает телефоны через PoE.

Кабели — **Copper Straight-Through**:

| Откуда | Куда |
| --- | --- |
| Router FastEthernet0/0 | Switch FastEthernet0/24 |
| Первый телефон, порт Switch | Switch FastEthernet0/1 |
| Второй телефон, порт Switch | Switch FastEthernet0/2 |
| Третий телефон, порт Switch | Switch FastEthernet0/3 |

На телефонах выбирать **Switch**, а не PC. При использовании исходного **2950** каждому телефону установить **Physical → IP_PHONE_POWER_ADAPTER**.

Маршрутизатор → **CLI** → на вопрос `initial configuration dialog` ответить **no**.

## 2. Коммутатор

```text
enable
configure terminal
hostname VoiceSwitch
no ip domain-lookup

interface fastEthernet0/24
switchport mode access
switchport access vlan 1
no shutdown
exit

interface range fastEthernet0/1 - 3
switchport mode access
switchport access vlan 1
switchport voice vlan 1
spanning-tree portfast
no shutdown
exit

end
copy running-config startup-config
```

На запрос имени файла — **Enter**. VLAN 1 уже существует; trunk и подинтерфейсы здесь не нужны.

## 3. Маршрутизатор — DHCP и CME

```text
enable
configure terminal
hostname CMERouter
no ip domain-lookup

interface fastEthernet0/0
ip address 192.168.0.1 255.255.255.0
no shutdown
exit

ip dhcp excluded-address 192.168.0.1 192.168.0.10

ip dhcp pool VOICE
network 192.168.0.0 255.255.255.0
default-router 192.168.0.1
option 150 ip 192.168.0.1
exit

telephony-service
max-dn 15
max-ephones 15
ip source-address 192.168.0.1 port 2000
auto assign 1 to 3
exit

ephone-dn 1
number 1111
exit
ephone-dn 2
number 2222
exit
ephone-dn 3
number 3333
exit

end
copy running-config startup-config
```

На запрос имени файла — **Enter**. Отдельный Server-PT и `ip helper-address` не нужны: DHCP работает на самом маршрутизаторе.

## 4. Телефоны

Оставить получение адресов по **DHCP**. Дождаться загрузки в **Realtime**, при необходимости нажать **Fast Forward Time**. Если телефоны включились до настройки CME, перезагрузить их после настройки.

Ожидается:

- IP-адреса — **192.168.0.11–254**, маска **255.255.255.0**.
- Шлюз и TFTP/CME — **192.168.0.1**.
- На экранах **GUI** — номера **1111, 2222, 3333**.

Номера назначаются в порядке регистрации; расположение телефонов на схеме не определяет их номер.

## 5. Проверка

### Маршрутизатор

```text
enable
show ip interface brief
show ip dhcp pool
show ip dhcp binding
show ip route
show ephone
show running-config
```

Ожидается: Fa0/0 — **192.168.0.1, up/up**; три DHCP-аренды и три зарегистрированных телефона; подключённая сеть **192.168.0.0/24**.

Дополнительно выполнить `ping` на каждый фактический IP телефона из `show ip dhcp binding`. Например, если выдан адрес `.11`:

```text
ping 192.168.0.11
```

### Коммутатор

```text
enable
show interfaces status
show vlan brief
show interfaces fastEthernet0/1 switchport
show interfaces fastEthernet0/2 switchport
show interfaces fastEthernet0/3 switchport
```

Fa0/1–3 и Fa0/24 — **connected**. Телефонные порты — **Access VLAN 1 / Voice VLAN 1**.

### Звонки

Телефон → **GUI → снять трубку → набрать номер → ответить на втором телефоне**. После каждого разговора положить трубки.

- **1111 → 2222** и обратно.
- **2222 → 3333** и обратно.
- **3333 → 1111** и обратно.

Если IP нет — проверить питание, порт Switch телефона, VLAN 1, `no shutdown` и DHCP. Если IP есть, но номера нет — проверить `option 150`, адрес CME, порт **2000**, три `ephone-dn` и `auto assign 1 to 3`; затем перезагрузить телефоны.

## 6. Сохранение

На маршрутизаторе и коммутаторе:

```text
copy running-config startup-config
```

На запрос имени — **Enter**. **File → Save As → Лабораторная_10.pkt**.

Исправления методички: для совместимости используется стандартный порт SCCP **2000** вместо **3100**; `auto assign 1 to 19` заменён на **1 to 3** по числу созданных линий; первый номер приведён к **1111**, как в примере звонка, вместо противоречащего ему `11111`; адрес маршрутизатора исключён из DHCP. Ёмкость CME **15/15** сохранена.

Цель выполнена после получения адресов и номеров всеми тремя телефонами и успешных звонков. Конфигурация здесь в Packet Tracer не запускалась; указаны ожидаемые результаты.

[Cisco — настройка телефонов и автоматическое назначение линий CME](https://www.cisco.com/c/en/us/td/docs/voice_ip_comm/cucme/admin/configuration/manual/cmeadm/cmebasic.html).
