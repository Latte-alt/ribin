# Лабораторная №11 — VoIP на проводных и Wi-Fi устройствах

[Методичка](../%D0%BB%D0%B0%D0%B1%2011.docx). Cisco Packet Tracer. Использовать новую схему.

## 1. Устройства и подключение

По одному: **2811, 3560-24PS, Home-VoIP-PT, Analog-Phone-PT, IP Phone 7960, PC-PT, AccessPoint-PT, Laptop-PT, SMARTPHONE-PT, TabletPC-PT**.

3560-24PS заменяет коммутатор 2960 из методички и питает 7960 через PoE. Home VoIP нужен для преобразования сигнала аналогового телефона. Модули Serial и FXS/FXO в маршрутизатор не устанавливать.

| Откуда | Куда | Кабель |
| --- | --- | --- |
| Router Fa0/0 | Switch Fa0/24 | Copper Straight-Through |
| Home VoIP Ethernet | Switch Fa0/1 | Copper Straight-Through |
| Analog Phone Phone | Home VoIP Phone | Phone |
| IP Phone 7960, порт Switch | Switch Fa0/2 | Copper Straight-Through |
| PC FastEthernet0 | Switch Fa0/3 | Copper Straight-Through |
| AccessPoint, проводной порт Port 0 | Switch Fa0/4 | Copper Straight-Through |

Ноутбук, смартфон и планшет подключаются к точке доступа **по Wi-Fi**. На 2811 при первом запуске ответить **no** на `initial configuration dialog`.

Если у Laptop-PT нет Wireless0: **Physical → выключить питание → убрать Ethernet-модуль → установить WPC300N → включить**. Сначала установить Wi-Fi-модуль, затем записывать его MAC.

## 2. Коммутатор

```text
enable
configure terminal
hostname VoiceSwitch
no ip domain-lookup

interface range fastEthernet0/1 - 4
switchport mode access
switchport access vlan 1
spanning-tree portfast
no shutdown
exit

interface fastEthernet0/2
switchport voice vlan 1
exit

interface fastEthernet0/24
switchport mode access
switchport access vlan 1
no shutdown
exit

end
copy running-config startup-config
```

На запрос имени файла — **Enter**. Все устройства находятся в VLAN 1 и одной сети; trunk не требуется.

## 3. Маршрутизатор — DHCP, CME и номера

```text
enable
configure terminal
hostname CMERouter
no ip domain-lookup

interface fastEthernet0/0
ip address 10.3.0.1 255.255.255.0
no shutdown
exit

ip dhcp excluded-address 10.3.0.1 10.3.0.10

ip dhcp pool Telephony
network 10.3.0.0 255.255.255.0
default-router 10.3.0.1
option 150 ip 10.3.0.1
exit

telephony-service
max-ephones 10
max-dn 10
ip source-address 10.3.0.1 port 2000
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
ephone-dn 4
number 104
exit
ephone-dn 5
number 105
exit
ephone-dn 6
number 106
exit

end
copy running-config startup-config
```

Автоматическое назначение `auto assign` здесь не используется: все шесть номеров ниже закрепляются вручную через `button`.

## 4. Home VoIP, Wi-Fi и получение адресов

**Home VoIP → Config → Server Address: `10.3.0.1`.** Сам аналоговый телефон IP-адрес не получает — в IP-сети работает Home VoIP.

**AccessPoint → Config → беспроводной Port 1:**

| Параметр | Значение |
| --- | --- |
| Port Status | On |
| SSID | wifi |
| Channel | 6 |
| Authentication | WPA2-PSK |
| Encryption, если выбирается | AES |
| PSK Pass Phrase | cisco123 |

На **смартфоне, планшете и ноутбуке**: **Config → Wireless0 → On**, задать SSID **wifi**, WPA2-PSK, AES и ключ **cisco123**. Если поле канала доступно — **6**; иначе канал определяется при подключении к точке. В IP Configuration выбрать **DHCP**. На ноутбуке для подключения также можно использовать **Desktop → PC Wireless**.

На проводном **PC → Desktop → IP Configuration → DHCP**. На 7960 оставить автоматическое получение адреса.

Ожидаемые адреса клиентов: **10.3.0.11–254/24**, шлюз и TFTP/CME — **10.3.0.1**. Для клиентов используется один DHCP-сервер на 2811. Точка доступа работает как мост к этой сети.

## 5. Записать MAC-адреса

На PC и ноутбуке: `ipconfig /all`; также можно открыть **Config → сетевой интерфейс**. На смартфоне и планшете — **Config → Wireless0**. Для 7960 и Home VoIP навести указатель на устройство и посмотреть MAC в подсказке.

| Номер | Устройство | Какой MAC взять | Замена в командах |
| --- | --- | --- | --- |
| 101 | Analog Phone через Home VoIP | Ethernet MAC **Home VoIP** | `<MAC_ATA>` |
| 102 | IP Phone 7960 | MAC телефона | `<MAC_7960>` |
| 103 | PC | FastEthernet0 | `<MAC_PC>` |
| 104 | Смартфон | Wireless0 | `<MAC_SMARTPHONE>` |
| 105 | Планшет | Wireless0 | `<MAC_TABLET>` |
| 106 | Ноутбук | **Wireless0**, а не Ethernet | `<MAC_LAPTOP>` |

**Перед вставкой следующего блока заменить все шесть обозначений вместе с угловыми скобками.** Формат Cisco: `XXXX.XXXX.XXXX` — 12 шестнадцатеричных цифр. Адреса из методички принадлежат её устройствам, копировать их нельзя.

## 6. Привязка устройств к номерам на CMERouter

```text
enable
configure terminal

ephone 1
mac-address <MAC_ATA>
type ata
button 1:1
exit

ephone 2
mac-address <MAC_7960>
type 7960
button 1:2
exit

ephone 3
mac-address <MAC_PC>
type CIPC
button 1:3
exit

ephone 4
mac-address <MAC_SMARTPHONE>
type CIPC
button 1:4
exit

ephone 5
mac-address <MAC_TABLET>
type CIPC
button 1:5
exit

ephone 6
mac-address <MAC_LAPTOP>
type CIPC
button 1:6
exit

end
copy running-config startup-config
```

Команда `device-security-mode none` из методички в базовом стенде не требуется и в основной блок не включена.

## 7. Запуск Cisco IP Communicator

На **PC, смартфоне, планшете и ноутбуке** открыть **Desktop → Cisco IP Communicator** после получения IP-адреса. Клиент может зарегистрироваться только после запуска приложения.

Проверить номера: **PC — 103, смартфон — 104, планшет — 105, ноутбук — 106**. В GUI аналогового телефона должен появиться **101**, на 7960 — **102**.

Если CIPC позволяет задать TFTP вручную через **Preferences → Network**, указать **10.3.0.1**. Сначала проверить DHCP option 150 и доступность маршрутизатора.

Если на устройстве **нет Cisco IP Communicator**, эта модель/версия симулятора не позволяет выполнить соответствующую часть инструкции. Не заменять смартфон или планшет обычным телефоном и не отмечать отсутствующую проверку выполненной.

## 8. Проверки

### CMERouter

```text
enable
show ip interface brief
show ip dhcp binding
show ephone
show running-config
```

Ожидается: Fa0/0 — **10.3.0.1, up/up**; адреса выданы Home VoIP, 7960 и четырём устройствам с CIPC; зарегистрированы шесть телефонных клиентов с номерами **101–106**. Аналоговый аппарат не создаёт отдельную DHCP-аренду.

Выполнить `ping` на фактические адреса клиентов из `show ip dhcp binding`. Например, если выдан `.11`:

```text
ping 10.3.0.11
```

### VoiceSwitch

```text
enable
show interfaces status
show vlan brief
show interfaces fastEthernet0/2 switchport
```

Fa0/1–4 и Fa0/24 — **connected**. Все проводные порты — access VLAN 1; порт 7960 дополнительно имеет voice VLAN 1.

### PC и ноутбук — Desktop → Command Prompt

```text
ipconfig /all
ping 10.3.0.1
```

На смартфоне и планшете проверить **Wireless0: соединение с wifi, IP 10.3.0.x/24 и шлюз 10.3.0.1**, затем номер в CIPC.

### Звонки

На телефонах — **GUI**, на остальных устройствах — **Cisco IP Communicator**: набрать номер и снять трубку; на принимающем устройстве ответить. Каждый разговор завершать перед следующим.

| Вызов | Что проверяется |
| --- | --- |
| **105 → 101**, затем **101 → 105** | Планшет ↔ аналоговый телефон |
| **101 → 102**, затем **102 → 101** | Аналоговый телефон ↔ 7960 |
| **103 → 104**, затем **104 → 103** | Проводной PC ↔ смартфон |
| **106 → 102**, затем **102 → 106** | Ноутбук ↔ 7960 |

Так проверяются все шесть клиентов, включая проводную, беспроводную и аналоговую части.

Если IP нет — проверить VLAN 1, кабели, DHCP, SSID и ключ Wi-Fi. Если IP есть, но номера нет — проверить MAC активного интерфейса, `type`, `button`, Server Address Home VoIP и запуск CIPC. После исправления перезапустить клиент/телефон.

## 9. Сохранение

На маршрутизаторе и коммутаторе:

```text
copy running-config startup-config
```

На запрос имени — **Enter**. **File → Save As → Лабораторная_11.pkt**.

Цель достигнута после регистрации шести клиентов под своими номерами и успешной проверки звонков. Конфигурация здесь в Packet Tracer не запускалась; приведены ожидаемые результаты.

Исправления методички: при ручной привязке убрано автоматическое назначение линий; MAC заменяются фактическими; **планшет → аналоговый телефон — 105 → 101**, обратный вызов — **101 → 105**.

Справочники: [Cisco CME — подключение ATA и IP Communicator](https://www.cisco.com/c/en/us/td/docs/voice_ip_comm/cucme/admin/configuration/manual/cmeadm/cmebasic.html), [справка Packet Tracer — Home VoIP и телефоны](https://tutorials.ptnetacad.net/help/default/config_others.htm).
