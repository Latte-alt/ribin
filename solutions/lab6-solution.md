# Лабораторная работа №6 — вариант для Packet Tracer

[Исходная методичка](../%D0%BB%D0%B0%D0%B1%206.docx).

Схема: **два Cisco 2811, два Catalyst 3560-24PS, два IP-телефона и два аналоговых телефона через Home VoIP**.

В используемом Packet Tracer отсутствует `dial-peer cor`. Здесь выполняются настройка двух узлов и проверка звонков через Serial. **COR и выход в PSTN не реализованы**, поэтому это адаптация доступной части задания, а не полная проверка ограничений.

## 1. Устройства и модули

| Название в Packet Tracer | Количество | Назначение |
| --- | --- | --- |
| 2811 | 2 | RouterA и RouterB |
| WIC-2T | 2 | По одному Serial-модулю в каждый 2811 |
| 3560-24PS | 2 | SwitchA и SwitchB |
| IP Phone 7960 | 2 | Номера 1001 и 1002 |
| Home-VoIP-PT | 2 | Адаптеры для аналоговых аппаратов |
| Analog-Phone-PT | 2 | Номера 2001 и 2002 |

Маршрутизатор → **Physical → выключить питание → установить WIC-2T в слот 3 → включить питание**.

В командах используется **Serial0/3/0**. Проверить фактическое имя:

```text
enable
show ip interface brief
```

Если модуль установлен в другой слот, заменить `serial0/3/0` во всех командах на своё имя.

**7960 получают питание PoE от 3560-24PS. Отдельные адаптеры питания им не требуются.** Home VoIP нужен для преобразования аналогового сигнала в IP и не заменяется PoE-коммутатором.

## 2. Подключение

Все Ethernet-соединения — **Copper Straight-Through**.

| Откуда | Куда |
| --- | --- |
| RouterA Fa0/0 | SwitchA Fa0/1 |
| IP-телефон 1001, порт Switch | SwitchA Fa0/2 |
| IP-телефон 1002, порт Switch | SwitchA Fa0/3 |
| RouterB Fa0/0 | SwitchB Fa0/1 |
| Home VoIP 1, порт Ethernet | SwitchB Fa0/2 |
| Home VoIP 2, порт Ethernet | SwitchB Fa0/3 |

Отдельно:

- **RouterA Serial0/3/0 ↔ RouterB Serial0/3/0** — кабель **Serial DCE**, первым выбрать RouterA.
- **Analog Phone 2001, Phone ↔ Home VoIP 1, Phone** — кабель **Phone**.
- **Analog Phone 2002, Phone ↔ Home VoIP 2, Phone** — кабель **Phone**.

**Телефоны не подключаются напрямую к маршрутизаторам.** Облако PSTN не добавлять: оно не создаёт работающую телефонную сеть.

## 3. Подготовка перед командами

Использовать новые устройства. Если уже вводилась старая конфигурация 6-й лабораторной, сохранить копию проекта и очистить настройки соответствующих устройств:

```text
enable
erase startup-config
reload
```

Подтверждение удаления — **Enter**; на предложение сохранить настройки при перезагрузке — **no**. На вопрос `initial configuration dialog` — **no**. Команды очистки удаляют прежние настройки устройства.

Узнать **Ethernet MAC обоих Home VoIP**: навести указатель на устройство или открыть сведения его Ethernet-интерфейса во вкладке **Config**. Записать, какой адаптер соединён с каким аналоговым телефоном.

В блоке RouterB заменить:
- `<MAC_HOME_2001>` — MAC первого Home VoIP;
- `<MAC_HOME_2002>` — MAC второго Home VoIP.

Формат MAC — `XXXX.XXXX.XXXX`, без угловых скобок. У аналоговых аппаратов Ethernet MAC нет.

## 4. SwitchA

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

interface range fastEthernet0/2 - 3
switchport mode access
switchport access vlan 1
switchport voice vlan 1
spanning-tree portfast
no shutdown
exit

end
copy running-config startup-config
```

На запрос имени файла — **Enter**.

## 5. SwitchB

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

interface range fastEthernet0/2 - 3
switchport mode access
switchport access vlan 1
switchport voice vlan 1
spanning-tree portfast
no shutdown
exit

end
copy running-config startup-config
```

С обеих сторон используется VLAN 1. Trunk и подинтерфейсы для этой схемы не нужны.

## 6. RouterA — IP-телефоны 1001 и 1002

```text
enable
configure terminal
hostname RouterA
no ip domain-lookup

interface fastEthernet0/0
ip address 192.168.10.1 255.255.255.0
no shutdown
exit

interface serial0/3/0
ip address 10.0.0.1 255.255.255.252
encapsulation hdlc
clock rate 2000000
no shutdown
exit

ip route 192.168.20.0 255.255.255.0 10.0.0.2

ip dhcp excluded-address 192.168.10.1 192.168.10.10
ip dhcp pool VOICE_A
network 192.168.10.0 255.255.255.0
default-router 192.168.10.1
option 150 ip 192.168.10.1
exit

telephony-service
max-dn 2
max-ephones 2
ip source-address 192.168.10.1 port 2000
auto assign 1 to 2
exit

ephone-dn 1
number 1001
exit
ephone-dn 2
number 1002
exit

dial-peer voice 20 voip
destination-pattern 200.
session target ipv4:10.0.0.2
codec g711ulaw
exit

end
copy running-config startup-config
```

Номера 1001 и 1002 назначатся IP-телефонам автоматически в порядке регистрации.

## 7. RouterB — аналоговые телефоны через Home VoIP

**Перед вставкой заменить два MAC-адреса.**

```text
enable
configure terminal
hostname RouterB
no ip domain-lookup

interface fastEthernet0/0
ip address 192.168.20.1 255.255.255.0
no shutdown
exit

interface serial0/3/0
ip address 10.0.0.2 255.255.255.252
encapsulation hdlc
no shutdown
exit

ip route 192.168.10.0 255.255.255.0 10.0.0.1

ip dhcp excluded-address 192.168.20.1 192.168.20.10
ip dhcp pool VOICE_B
network 192.168.20.0 255.255.255.0
default-router 192.168.20.1
option 150 ip 192.168.20.1
exit

telephony-service
max-dn 2
max-ephones 2
ip source-address 192.168.20.1 port 2000
exit

ephone-dn 1
number 2001
exit
ephone-dn 2
number 2002
exit

ephone 1
mac-address <MAC_HOME_2001>
type ata
button 1:1
exit

ephone 2
mac-address <MAC_HOME_2002>
type ata
button 1:2
exit

dial-peer voice 10 voip
destination-pattern 100.
session target ipv4:10.0.0.1
codec g711ulaw
exit

end
copy running-config startup-config
```

`clock rate` задаётся только на RouterA — стороне DCE. Если кабель установлен наоборот, перенести команду на фактическую DCE-сторону.

Номера аналоговых телефонов настраиваются на RouterB через регистрацию **Home VoIP как ATA**. Это не физические FXS-порты маршрутизатора.

## 8. Home VoIP и регистрация

На **обоих Home VoIP**:

1. Открыть **Config → Settings**.
2. В поле **Server Address** указать **192.168.20.1**.
3. Для Ethernet-интерфейса выбрать **DHCP**, если доступен выбор способа получения адреса.
4. Дождаться получения адреса и регистрации.

Ожидаемые параметры:

| Устройства | IP-сеть | Шлюз / сервер телефонии |
| --- | --- | --- |
| IP-телефоны 1001, 1002 | 192.168.10.0/24 | 192.168.10.1 |
| Home VoIP для 2001, 2002 | 192.168.20.0/24 | 192.168.20.1 |

Аналоговый аппарат собственного IP не получает — адрес получает его Home VoIP.

Подождать в режиме **Realtime** или нажать **Fast Forward Time**. Если телефоны включились до настройки, при необходимости перезапустить их после ввода команд.

## 9. Проверка

### RouterA

```text
show ip interface brief
show ip route
show ip dhcp binding
show ephone
show dial-peer voice summary
ping 10.0.0.2
ping 192.168.20.1
```

### RouterB

```text
show ip interface brief
show ip route
show ip dhcp binding
show ephone
show dial-peer voice summary
ping 10.0.0.1
ping 192.168.10.1
```

Ожидается:
- Fa0/0 и Serial0/3/0 — **up/up** на обоих маршрутизаторах.
- У RouterA есть статический маршрут **192.168.20.0/24**, у RouterB — **192.168.10.0/24**.
- RouterA выдал два адреса IP-телефонам.
- RouterB выдал два адреса Home VoIP.
- На RouterA зарегистрированы два 7960, на RouterB — два ATA.

### Оба коммутатора

```text
show interfaces status
show vlan brief
show interfaces fastEthernet0/2 switchport
show interfaces fastEthernet0/3 switchport
```

На SwitchA дополнительно, если команда поддерживается:

```text
show power inline
```

### Проверка звонков

Снять трубку, набрать номер, ответить на другом аппарате. После каждого теста завершить разговор.

| Вызов | Что проверяется |
| --- | --- |
| 1001 → 1002 | Локальная IP-телефония слева |
| 2001 → 2002 | Аналоговые аппараты через ATA справа |
| 1001 → 2001 | Звонок через Serial слева направо |
| 1002 → 2002 | Второй удалённый аппарат |
| 2001 → 1001 | Звонок справа налево |
| 2002 → 1002 | Обратное направление второго аппарата |

**В этой конфигурации все перечисленные звонки должны проходить: ограничений COR нет.**

## 10. Если связь не работает

- Serial красный: проверить питание, модуль WIC-2T, имя интерфейса, `no shutdown` и тактирование DCE.
- Home VoIP не регистрируется: проверить MAC, DHCP и **Server Address 192.168.20.1**.
- Работают только локальные звонки: проверить статические маршруты, ping удалённого Fa0/0 и `session target`.
- Телефон 7960 не включается: проверить, что используется **3560-24PS**, исправный Ethernet-порт и подключение к порту телефона **Switch**.
- Для диагностики конфигурации на обоих маршрутизаторах:

```text
show running-config
show controllers serial 0/3/0
show interfaces serial0/3/0
```

## 11. Дополнительные сервисы

При наличии поддержки и кнопок:
- перевод: **1001 → 1002**, ответ → на 1002 **Transfer → 2001 → Transfer**;
- конференция: разговор **1001 ↔ 1002** → **Confrn → 2001 → ответ → Confrn**;
- перехват: **2001 звонит 1001**, на 1001 не отвечать → на 1002 **PickUp → 1001**, при необходимости **Answer**.

Проверять эти функции отдельно: базовая конфигурация и данная версия Packet Tracer могут их не поддерживать.

## 12. Что не выполнено из исходного задания

- Ограничения по классам COR: в используемой модели нет `dial-peer cor`.
- Звонки в PSTN и запрет PSTN → аналоговый аппарат: внешняя телефонная сеть и FXO здесь отсутствуют.

Неудачный звонок без настроенной PSTN не подтверждает работу ограничения. Для полной исходной лабораторной нужен стенд с соответствующими функциями.

Фраза для отчёта после успешной проверки связи: «Настроена связь двух узлов через Serial, регистрация IP-телефонов и аналоговых аппаратов через Home VoIP. Проверены локальные и межузловые звонки. COR и PSTN не проверены из-за отсутствия необходимых функций в используемом симуляторе».

## 13. Сохранение

На обоих маршрутизаторах и коммутаторах:

```text
copy running-config startup-config
```

На запрос имени файла — **Enter**. **File → Save As → Лабораторная_6.pkt**.

[Контрольные вопросы и ответы](labs-control-answers.md#лабораторная-работа-6).
