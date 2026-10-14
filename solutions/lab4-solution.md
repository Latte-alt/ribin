# Лабораторная работа №4 — VoIP и VLAN

[Методичка](../%D0%BB%D0%B0%D0%B1%204.docx). Cisco Packet Tracer.

## 1. Подготовка

Добавить **2811**, **2960-24TT**, **3 телефона 7960**, **3 PC-PT**. Использовать новые устройства или предварительно очистить их прежние настройки.

Кабели — **Copper Straight-Through**:

| Откуда | Куда |
| --- | --- |
| Router0 Fa0/0 | Switch0 Fa0/1 |
| Телефон 101, порт Switch | Switch0 Fa0/2 |
| Телефон 102, порт Switch | Switch0 Fa0/3 |
| Телефон 103, порт Switch | Switch0 Fa0/4 |
| PC0 FastEthernet0 | Телефон 101, порт PC |
| PC1 FastEthernet0 | Телефон 102, порт PC |
| PC2 FastEthernet0 | Телефон 103, порт PC |

Каждый телефон → **Physical** → установить **IP_PHONE_POWER_ADAPTER**.

Узнать MAC каждого телефона: навести указатель на телефон и посмотреть MAC сетевого интерфейса либо открыть **Config** и сведения о его интерфейсе. Записать адреса именно телефонов, не ПК. Формат для команды: `XXXX.XXXX.XXXX` — 12 шестнадцатеричных цифр.

В блоке маршрутизатора заменить **`<MAC_PHONE_101>`**, **`<MAC_PHONE_102>`**, **`<MAC_PHONE_103>`** фактическими адресами, без угловых скобок. Адреса в методичке относятся к её схеме и не подходят для новых устройств.

Маршрутизатор → **CLI** → на вопрос `initial configuration dialog` ответить **no**.

## 2. Коммутатор

```text
enable
configure terminal
hostname SwitchA
no ip domain-lookup

vlan 10
name Data
exit
vlan 20
name Voice
exit
vlan 99
name Management
exit

interface vlan 99
ip address 192.168.99.10 255.255.255.0
no shutdown
exit
ip default-gateway 192.168.99.1

interface fastEthernet0/1
switchport mode trunk
switchport trunk native vlan 99
switchport trunk allowed vlan 10,20,99
no shutdown
exit

interface range fastEthernet0/2 - 4
switchport mode access
switchport access vlan 10
switchport voice vlan 20
spanning-tree portfast
no shutdown
exit

end
copy running-config startup-config
```

На запрос имени файла — **Enter**. Для 2960 команда `switchport trunk encapsulation dot1q` не нужна.

## 3. Маршрутизатор

**Перед вставкой заменить три MAC-адреса.**

```text
enable
configure terminal
hostname CMERouter
no ip domain-lookup

interface fastEthernet0/0
no ip address
no shutdown
exit

interface fastEthernet0/0.10
encapsulation dot1Q 10
ip address 192.168.10.1 255.255.255.0
no shutdown
exit

interface fastEthernet0/0.20
encapsulation dot1Q 20
ip address 192.168.20.1 255.255.255.0
no shutdown
exit

interface fastEthernet0/0.99
encapsulation dot1Q 99 native
ip address 192.168.99.1 255.255.255.0
no shutdown
exit

ip dhcp excluded-address 192.168.10.1 192.168.10.9
ip dhcp excluded-address 192.168.20.1 192.168.20.9

ip dhcp pool Data
network 192.168.10.0 255.255.255.0
default-router 192.168.10.1
exit

ip dhcp pool Voice
network 192.168.20.0 255.255.255.0
default-router 192.168.20.1
option 150 ip 192.168.20.1
exit

telephony-service
max-dn 3
max-ephones 3
ip source-address 192.168.20.1 port 2000
exit

ephone-dn 1
number 101
exit
ephone-dn 2
number 102
exit
ephone-dn 3
number 103
exit

ephone 1
mac-address <MAC_PHONE_101>
type 7960
button 1:1
exit

ephone 2
mac-address <MAC_PHONE_102>
type 7960
button 1:2
exit

ephone 3
mac-address <MAC_PHONE_103>
type 7960
button 1:3
exit

end
copy running-config startup-config
```

На запрос имени файла — **Enter**. Номера привязаны вручную, команда `auto assign` не требуется.

## 4. ПК и телефоны

На каждом ПК: **Desktop → IP Configuration → DHCP**.

Дождаться регистрации телефонов в режиме **Realtime**; при необходимости нажать **Fast Forward Time**. Если номера не появились после настройки, переподключить питание телефонов.

Ожидается:

| Устройства | Адреса / номера |
| --- | --- |
| ПК | 192.168.10.x, шлюз 192.168.10.1 |
| Телефоны | 192.168.20.x, шлюз 192.168.20.1; номера 101, 102, 103 |
| Коммутатор | 192.168.99.10, шлюз 192.168.99.1 |

## 5. Проверка

### Коммутатор

```text
enable
show vlan brief
show interfaces trunk
show interfaces fastEthernet0/2 switchport
show interfaces fastEthernet0/3 switchport
show interfaces fastEthernet0/4 switchport
show ip interface brief
ping 192.168.99.1
ping 192.168.10.1
ping 192.168.20.1
```

Fa0/1 — trunk, native VLAN 99. На Fa0/2–4: **Access VLAN 10**, **Voice VLAN 20**. Интерфейс Vlan99 — **up/up**.

### Маршрутизатор

```text
enable
show ip interface brief
show ip route
show ip dhcp pool
show ip dhcp binding
show ephone
show running-config
ping 192.168.99.10
```

Если поддерживается:

```text
show ephone registered
```

Подинтерфейсы Fa0/0.10, Fa0/0.20 и Fa0/0.99 — **up/up**. В таблице маршрутизации присутствуют три подключённые сети; в DHCP — адреса ПК и телефонов.

### На каждом ПК — Desktop → Command Prompt

```text
ipconfig /all
ping 192.168.10.1
ping 192.168.20.1
ping 192.168.99.10
```

Также выполнить `ping` на фактический адрес другого ПК, узнав его через `ipconfig`. Для проверки телефонов с маршрутизатора использовать их фактические IP из `show ip dhcp binding`.

### Звонки — GUI телефонов

- Снять трубку на 101 → набрать **102** → ответить на 102.
- Проверить **102 → 103** и **103 → 101**.
- Если поддерживаются дополнительные сервисы: перевод — **Transfer → номер → Transfer**; конференция — **Confrn → номер третьего участника → ответ → Confrn**; направленный перехват — **PickUp → номер вызываемого телефона**.

Если нужные кнопки или функции недоступны в используемой версии Packet Tracer, отметить это в отчёте. Базовая конфигурация методички не гарантирует поддержку дополнительных сервисов.

## 6. Сохранение

На маршрутизаторе и коммутаторе:

```text
copy running-config startup-config
```

На запрос имени файла — **Enter**. **File → Save As → Лабораторная_4.pkt**.

## Исправления методички

- Для телефонов задан `switchport voice vlan 20`, а для ПК — `switchport access vlan 10`. Две команды `switchport access vlan` подряд не разделяют голос и данные.
- На подинтерфейсах сначала задаётся `encapsulation dot1Q`, затем IP-адрес.
- Номера задаются командами `number 101`, `number 102`, `number 103`.
- Используются полные фактические MAC-адреса телефонов.
- `ip default-gateway` задаёт шлюз управления L2-коммутатора. Маршрутизатору для связи между тремя непосредственно подключёнными сетями маршрут по умолчанию не нужен.

[Ответы на контрольные вопросы](labs-control-answers.md#лабораторная-работа-4).
