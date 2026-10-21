# Лабораторная работа №8 — удалённые IP-телефоны и EIGRP

[Методичка](../%D0%BB%D0%B0%D0%B1%208.docx). Решение для новой схемы в Cisco Packet Tracer.

**Один CME на CMERouter обслуживает все три телефона, включая удалённый.** На RemoteRouter телефонный сервис и VoIP dial-peer для внутренних звонков не нужны.

**Границы выполнения:** основные блоки настраивают VLAN, EIGRP и регистрацию удалённого телефона на центральном CME. Задание также требует выхода в PSTN и проверки перевода, конференции и перехвата. PSTN требует отдельной телефонной линии/стенда; дополнительные сервисы зависят от поддержки симулятора. Без успешной проверки этих пунктов лабораторная выполнена частично. COR в задании №8 не требуется. Файл содержит инструкции, а не результаты проверки готового `.pkt`.

## 1. Устройства и подключение

- **2 × 2811** — CMERouter и RemoteRouter; второй 2811 заменяет отсутствующий 2600.
- **2 × 3560-24PS** — CMESwitch и RemoteSwitch, как на схеме.
- **3 × IP Phone 7960** — два слева, один справа. Home VoIP не нужен.
- **2 × WIC-2T** — по одному в каждый маршрутизатор.

Physical → выключить маршрутизатор → установить WIC-2T в слот 3 → включить. В командах используется **Serial0/3/0**. Если `show ip interface brief` показывает другое имя, заменить его во всех блоках.

| Откуда | Куда | Кабель |
| --- | --- | --- |
| CMERouter Fa0/0 | CMESwitch Fa0/1 | Copper Straight-Through |
| RemoteRouter Fa0/0 | RemoteSwitch Fa0/1 | Copper Straight-Through |
| CMERouter S0/3/0 | RemoteRouter S0/3/0 | Serial DCE, первым выбрать CMERouter |
| Первый телефон, порт Switch | CMESwitch Fa0/2 | Copper Straight-Through |
| Второй телефон, порт Switch | CMESwitch Fa0/3 | Copper Straight-Through |
| Удалённый телефон, порт Switch | RemoteSwitch Fa0/2 | Copper Straight-Through |

3560-24PS подаёт питание 7960 через PoE. Использовать порт телефона **Switch**, а не PC. На вопрос `initial configuration dialog` маршрутизатора ответить **no**.

| Сегмент | VLAN | Сеть | Шлюз |
| --- | --- | --- | --- |
| Центральные ПК, если добавлены | 10 | 10.10.1.0/24 | 10.10.1.1 |
| Центральные телефоны | 20 | 10.20.1.0/24 | 10.20.1.1 |
| Удалённый телефон | 30 | 10.30.1.0/24 | 10.30.1.1 |
| Удалённые ПК, если добавлены | 40 | 10.40.1.0/24 | 10.40.1.1 |
| Serial | — | 82.115.34.208/30 | CMERouter .210, RemoteRouter .209 |

## 2. CMESwitch

```text
enable
configure terminal
hostname CMESwitch
no ip domain-lookup

vlan 10
name DATA
exit
vlan 20
name VOICE
exit

interface fastEthernet0/1
switchport trunk encapsulation dot1q
switchport mode trunk
switchport trunk allowed vlan 10,20
no shutdown
exit

interface range fastEthernet0/2 - 3
switchport mode access
switchport access vlan 10
switchport voice vlan 20
spanning-tree portfast
no shutdown
exit

end
copy running-config startup-config
```

На запрос `Destination filename [startup-config]?` — **Enter**. То же при сохранении остальных устройств.

## 3. RemoteSwitch

```text
enable
configure terminal
hostname RemoteSwitch
no ip domain-lookup

vlan 30
name VOICE
exit
vlan 40
name DATA
exit

line vty 0 4
password cisco
login
logging synchronous
exit
line console 0
password Cisco
login
logging synchronous
exit

interface fastEthernet0/1
switchport trunk encapsulation dot1q
switchport mode trunk
switchport trunk allowed vlan 30,40
no shutdown
exit

interface fastEthernet0/2
switchport mode access
switchport access vlan 40
switchport voice vlan 30
spanning-tree portfast
no shutdown
exit

end
copy running-config startup-config
```

Пароли — учебные из методички. Для входа через консоль после повторного открытия потребуется `Cisco`. Настройка VTY сама по себе не создаёт IP-адрес управления коммутатором.

## 4. CMERouter — VLAN, DHCP, EIGRP и единственный CME

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
ip address 10.10.1.1 255.255.255.0
exit
interface fastEthernet0/0.20
encapsulation dot1Q 20
ip address 10.20.1.1 255.255.255.0
exit

interface serial0/3/0
ip address 82.115.34.210 255.255.255.252
encapsulation hdlc
clock rate 2000000
bandwidth 2000
no shutdown
exit

ip dhcp excluded-address 10.10.1.1 10.10.1.10
ip dhcp excluded-address 10.20.1.1 10.20.1.10

ip dhcp pool LOCAL_DATA
network 10.10.1.0 255.255.255.0
default-router 10.10.1.1
exit
ip dhcp pool LOCAL_VOICE
network 10.20.1.0 255.255.255.0
default-router 10.20.1.1
option 150 ip 10.20.1.1
exit

router eigrp 100
network 10.10.1.0 0.0.0.255
network 10.20.1.0 0.0.0.255
network 82.115.34.208 0.0.0.3
no auto-summary
passive-interface fastEthernet0/0.10
passive-interface fastEthernet0/0.20
exit

telephony-service
max-ephones 5
max-dn 5
ip source-address 10.20.1.1 port 2000
auto assign 1 to 3
exit

ephone-dn 1
number 54001
exit
ephone-dn 2
number 54002
exit
ephone-dn 3
number 54003
exit

end
copy running-config startup-config
```

`clock rate` задаётся только на DCE. `bandwidth 2000` указывает полосу 2000 кбит/с для расчётов протоколов, но физическую скорость не меняет.

## 5. RemoteRouter — VLAN, DHCP и EIGRP

```text
enable
configure terminal
hostname RemoteRouter
no ip domain-lookup

interface fastEthernet0/0
no ip address
no shutdown
exit

interface fastEthernet0/0.30
encapsulation dot1Q 30
ip address 10.30.1.1 255.255.255.0
exit
interface fastEthernet0/0.40
encapsulation dot1Q 40
ip address 10.40.1.1 255.255.255.0
exit

interface serial0/3/0
ip address 82.115.34.209 255.255.255.252
encapsulation hdlc
bandwidth 2000
no shutdown
exit

ip dhcp excluded-address 10.30.1.1 10.30.1.10
ip dhcp excluded-address 10.40.1.1 10.40.1.10

ip dhcp pool REMOTE_VOICE
network 10.30.1.0 255.255.255.0
default-router 10.30.1.1
option 150 ip 10.20.1.1
exit
ip dhcp pool REMOTE_DATA
network 10.40.1.0 255.255.255.0
default-router 10.40.1.1
exit

router eigrp 100
network 10.30.1.0 0.0.0.255
network 10.40.1.0 0.0.0.255
network 82.115.34.208 0.0.0.3
no auto-summary
passive-interface fastEthernet0/0.30
passive-interface fastEthernet0/0.40
exit

end
copy running-config startup-config
```

У удалённого телефона **шлюз 10.30.1.1**, а **TFTP/CME — 10.20.1.1**. Это разные адреса. DHCP работает на RemoteRouter, поэтому `ip helper-address` не требуется.

## 6. Телефоны

Оставить получение адресов по **DHCP**, дождаться EIGRP и перезагрузить телефоны при необходимости. DHCP заменяет ручной ввод адресов через меню реального 7960 из методички.

| Телефоны | Адрес | Шлюз | TFTP/CME |
| --- | --- | --- | --- |
| Два центральных | 10.20.1.x/24 | 10.20.1.1 | 10.20.1.1 |
| Удалённый | 10.30.1.x/24 | 10.30.1.1 | 10.20.1.1 |

Номера **54001–54003** назначаются в порядке регистрации. Ориентироваться на экран **GUI**: удалённый аппарат может получить любой из трёх номеров. Все три регистрации должны отображаться на **CMERouter**.

Если нужен именно статический адрес удалённого аппарата и интерфейс телефона позволяет его задать: **10.30.1.2**, маска **255.255.255.0**, шлюз **10.30.1.1**, TFTP/CallManager **10.20.1.1**, VLAN **30**. Адрес `.2` уже исключён из DHCP.

## 7. Проверка

### CMERouter

```text
show ip interface brief
show controllers serial 0/3/0
show ip eigrp neighbors
show ip protocols
show ip route eigrp
show ip dhcp binding
show ephone
show running-config
ping 82.115.34.209
ping 10.30.1.1
ping 10.40.1.1
```

### RemoteRouter

```text
show ip interface brief
show controllers serial 0/3/0
show ip eigrp neighbors
show ip route eigrp
show ip dhcp binding
show running-config
ping 82.115.34.210
ping 10.20.1.1
ping 10.10.1.1
```

### Оба коммутатора

```text
show interfaces status
show vlan brief
show interfaces trunk
show interfaces fastEthernet0/2 switchport
```

Ожидается:

- Serial и используемые подинтерфейсы — **up/up**.
- На каждом маршрутизаторе — один сосед EIGRP через Serial.
- На CMERouter маршруты **D** к 10.30.1.0/24 и 10.40.1.0/24; на RemoteRouter — к 10.10.1.0/24 и 10.20.1.0/24.
- Fa0/1 обоих коммутаторов — trunk. Телефонные порты: центральные **Access 10 / Voice 20**, удалённый **Access 40 / Voice 30**.
- Два DHCP-адреса телефонов на CMERouter, один на RemoteRouter; три зарегистрированных телефона на CMERouter.

На CMERouter дополнительно выполнить `ping` на фактический IP удалённого телефона из `show ip dhcp binding` на RemoteRouter.

Телефон → **GUI** → снять трубку → набрать номер → ответить. Проверить **центральный ↔ центральный**, затем **удалённый → каждый центральный** и **каждый центральный → удалённый**. Номера брать с экранов.

Если адрес есть, но номера нет: проверить `option 150 ip 10.20.1.1`, доступность 10.20.1.1, обратный маршрут к 10.30.1.0/24 и `show ephone` на CMERouter. Не создавать второй CME как обход проблемы: это изменит задание.

## 8. Перевод, конференция, перехват

Пусть **A и B** — центральные аппараты, **C** — удалённый; использовать их фактические номера.

| Сервис | Проверка в GUI, если кнопки доступны |
| --- | --- |
| Перевод | C звонит A → A отвечает → на A **Transfer → номер B → Transfer** → B отвечает |
| Конференция | A разговаривает с C → на A **Confrn → номер B** → B отвечает → на A снова **Confrn** |
| Направленный перехват | C звонит A, A не отвечает → на B **PickUp → номер A**, при необходимости **Answer** |

Наличие номеров и обычных звонков не доказывает работу этих сервисов. Если GUI или IOS не поддерживает функцию, указать её как **не проверенную**, а не выполненную.

На полноценном IOS/CME для перевода с консультацией и программной конференции использовать двухканальные линии: **при первоначальном создании** заменить `ephone-dn 1`, `ephone-dn 2`, `ephone-dn 3` на `ephone-dn 1 dual-line`, `ephone-dn 2 dual-line`, `ephone-dn 3 dual-line`. Дополнительно на CMERouter, если команды поддерживаются:

```text
configure terminal
telephony-service
transfer-system full-consult
max-conferences 4 gain -6
create cnf-files
exit
end
copy running-config startup-config
```

После изменения конфигурации перезагрузить телефоны. Программная конференция требует подходящей версии CME и кодека G.711; доступность команд и ресурсов проверяется отдельно. Если `dual-line` или команды сервиса не поддерживаются в Packet Tracer, этот блок пропустить и зафиксировать ограничение.

## 9. PSTN — обязательный пункт для полного выполнения

В основной схеме выше **нет выхода в PSTN**. Cloud-PT и Home VoIP не заменяют телефонную линию оператора. Отсутствие COR здесь не мешает; для PSTN необходим отдельный поддерживаемый голосовой интерфейс и внешний телефонный стенд.

Для физического 2811 с совместимым **FXO-модулем**, голосовыми ресурсами и IOS/CME: подключить **FXO CMERouter к аналоговой линии оператора или FXS-порту лабораторной АТС**. Узнать фактический номер порта:

```text
show voice port summary
```

**Следующий шаблон только для такого стенда, не для основной схемы Packet Tracer.** Заменить `<FXO_PORT>` фактическим портом, например `0/0/0`, только если он действительно указан как FXO.

```text
configure terminal
dial-peer voice 90 pots
destination-pattern 9T
port <FXO_PORT>
exit
end
copy running-config startup-config
```

С удалённого телефона набрать **9 + действительный номер внешнего тестового телефона**, дождаться окончания межцифрового таймера. Фиксированная цифра `9` удаляется POTS dial-peer; остальные цифры передаются в линию. Ответить на внешнем аппарате и проверить разговор в обе стороны. Отдельный VoIP dial-peer на RemoteRouter не требуется: удалённый 7960 уже обслуживается центральным CME.

На CMERouter во время проверки, если команды доступны:

```text
show dial-peer voice summary
show voice port summary
show call active voice brief
```

Этот шаблон предполагает уже работающую внешнюю линию и её совместимость со стендом; он не создаёт PSTN. Если такой линии/стенда нет, в отчёте отметить **«выход в PSTN не выполнен»**. Настройка публичного оператора определяется его интерфейсом и параметрами, которых в методичке нет.

## 10. Сохранение

На всех четырёх сетевых устройствах:

```text
copy running-config startup-config
```

На запрос имени — **Enter**. **File → Save As → Лабораторная_8.pkt**.

Исправления методички: `network` вместо `netwok`; удалённый Serial **82.115.34.209/30**, а не `.20`; включён физический Fa0/0; явно созданы VLAN; добавлены DHCP для удалённого телефона и центральный CME с тремя номерами. EIGRP сохраняет номер автономной системы **100** и отключённое автоматическое суммирование.

[Ответы на контрольные вопросы](labs-control-answers.md#лабораторная-работа-8).

Технические справочники Cisco: [EIGRP](https://www.cisco.com/c/en/us/support/docs/ip/enhanced-interior-gateway-routing-protocol-eigrp/16406-eigrp-toc.html), [DHCP и параметры сети CME](https://www.cisco.com/c/en/us/td/docs/voice_ip_comm/cucme/admin/configuration/manual/cmeadm/cmenetwk.html), [перевод](https://www.cisco.com/c/en/us/td/docs/voice_ip_comm/cucme/admin/configuration/manual/cmeadm/cmetrans.html), [конференции](https://www.cisco.com/c/en/us/td/docs/voice_ip_comm/cucme/admin/configuration/manual/cmeadm/cmeconf.html), [выход через голосовые шлюзы](https://www.cisco.com/c/en/us/td/docs/voice_ip_comm/cucme/srnd/design/guide/cmesrnd/gatewy.html).
