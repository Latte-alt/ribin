# Лабораторная работа №6 — COR между двумя узлами

[Методичка](../%D0%BB%D0%B0%D0%B1%206.docx).

**В твоём Packet Tracer отсутствует `dial-peer cor`: полную проверку ограничений выполнить в нём нельзя.** Ниже приведено решение для Cisco IOS с поддержкой CME/COR и голосовых портов. В Packet Tracer можно выполнить сетевую часть и базовую IP-телефонию; физические FXS/FXO и ограничения нельзя считать проверенными без соответствующего оборудования и поддержки команд.

## 1. Подготовка и номера

- **R1:** Cisco 2811 с CME, Serial и FXO.
- **R2:** Cisco 2600 с Serial и двумя FXS; конкретная модель и IOS должны поддерживать голосовые функции.
- **SwitchA:** Catalyst 3560.
- **Два IP-телефона 7960**, два аналоговых телефона.
- Для PSTN — действующая учебная линия внешней АТС и внешний тестовый телефон.

| Соединение | Кабель |
| --- | --- |
| R1 Fa0/0 → SwitchA Fa0/1 | Copper Straight-Through |
| IP-телефон 1001, Switch → SwitchA Fa0/2 | Copper Straight-Through |
| IP-телефон 1002, Switch → SwitchA Fa0/3 | Copper Straight-Through |
| R1 Serial → R2 Serial | Serial DCE/DTE; DCE на R1 |
| Аналоговый 2001 → FXS №1 на R2 | Phone |
| Аналоговый 2002 → FXS №2 на R2 | Phone |
| FXO на R1 → линия внешней АТС | Phone |

Аналоговые телефоны подключаются к **FXS**, а не непосредственно к Ethernet-коммутатору. Правый коммутатор на рисунке методички для такого подключения не нужен. Подпись Trunk слева не требует trunk в решении с одной VLAN.

Номера и адреса для этой схемы выбраны явно, поскольку в методичке полный план не задан:

| Назначение | Значение |
| --- | --- |
| LAN R1 | 192.168.10.0/24 |
| R1 Fa0/0 | 192.168.10.1 |
| Serial R1 / R2 | 10.0.0.1 / 10.0.0.2, маска /30 |
| IP-телефоны | 1001, 1002 |
| Аналоговые телефоны на удалённом узле | 2001, 2002 |
| Выход в PSTN | 9 + внешний номер |

Перед вставкой заменить:
- `<WAN_R1>`, `<WAN_R2>` — реальные Serial-интерфейсы, например `Serial0/0/0` и `Serial0/0`.
- `<FXO_R1>`, `<FXS_A_R2>`, `<FXS_B_R2>` — реальные голосовые порты.
- `<MAC_1001>`, `<MAC_1002>` — MAC IP-телефонов в формате `XXXX.XXXX.XXXX`.

Узнать интерфейсы и голосовые порты:

```text
enable
show ip interface brief
show voice port summary
```

Работать на новых или очищенных лабораторных устройствах. На вопрос `initial configuration dialog` — **no**.

## 2. Коммутатор SwitchA

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

На запрос имени файла — **Enter**. Телефонам обеспечить PoE или установить адаптеры питания.

## 3. R1 — сеть, DHCP и CME

```text
enable
configure terminal
hostname CMERouter
no ip domain-lookup

interface fastEthernet0/0
ip address 192.168.10.1 255.255.255.0
no shutdown
exit

interface <WAN_R1>
ip address 10.0.0.1 255.255.255.252
encapsulation hdlc
clock rate 2000000
no shutdown
exit

ip dhcp excluded-address 192.168.10.1 192.168.10.10
ip dhcp pool VOICE
network 192.168.10.0 255.255.255.0
default-router 192.168.10.1
option 150 ip 192.168.10.1
exit

telephony-service
max-dn 2
max-ephones 2
ip source-address 192.168.10.1 port 2000
exit

ephone-dn 1
number 1001
exit
ephone-dn 2
number 1002
exit

ephone 1
mac-address <MAC_1001>
type 7960
button 1:1
exit
ephone 2
mac-address <MAC_1002>
type 7960
button 1:2
exit

end
copy running-config startup-config
```

На полноценном IOS, если требуется генерация файлов конфигурации телефонов:

```text
configure terminal
telephony-service
create cnf-files
exit
end
```

Физическим телефонам также требуется совместимая SCCP-прошивка и её штатное предоставление через TFTP, если она ещё не установлена. Не задавать имя файла прошивки наугад.

## 4. R2 — Serial, FXS и маршруты звонков

```text
enable
configure terminal
hostname AnalogRouter
no ip domain-lookup

interface <WAN_R2>
ip address 10.0.0.2 255.255.255.252
encapsulation hdlc
no shutdown
exit

ip route 192.168.10.0 255.255.255.0 10.0.0.1

voice-port <FXS_A_R2>
signal loop-start
station-id number 2001
no shutdown
exit
voice-port <FXS_B_R2>
signal loop-start
station-id number 2002
no shutdown
exit

dial-peer voice 21 pots
destination-pattern 2001
port <FXS_A_R2>
exit
dial-peer voice 22 pots
destination-pattern 2002
port <FXS_B_R2>
exit

dial-peer voice 10 voip
destination-pattern 100.
session target ipv4:10.0.0.1
codec g711ulaw
no vad
exit
dial-peer voice 90 voip
destination-pattern 9T
session target ipv4:10.0.0.1
codec g711ulaw
no vad
exit

end
copy running-config startup-config
```

На R2 `clock rate` не задаётся: эта сторона DTE. Точный тип кабеля проверить через `show controllers serial` с номером своего интерфейса.

## 5. R1 — COR и внешние направления

**Выполнять только в IOS с поддержкой COR.** В Packet Tracer, где доступен лишь `dial-peer voice`, этот блок полностью не выполнится.

Права соответствуют заданию:
- **1001:** локальные и удалённые звонки через IP/WAN, без PSTN.
- **1002:** все настроенные направления.
- **2001/2002:** локальные IP-номера и PSTN.
- **Входящий PSTN:** разрешены локальные IP-номера, запрещены удалённые аналоговые.

```text
configure terminal

dial-peer cor custom
name PRI
name Analog
name PSTN
exit

dial-peer cor list callPRI
member PRI
exit
dial-peer cor list callAnalog
member Analog
exit
dial-peer cor list callPSTN
member PSTN
exit

dial-peer cor list Type1
member PRI
member Analog
exit
dial-peer cor list Type2
member PRI
member Analog
member PSTN
exit
dial-peer cor list AnalogUser
member PRI
member PSTN
exit
dial-peer cor list FromPSTN
member PRI
exit

ephone-dn 1
corlist incoming Type1
corlist outgoing callPRI
exit
ephone-dn 2
corlist incoming Type2
corlist outgoing callPRI
exit

dial-peer voice 20 voip
incoming called-number .
destination-pattern 200.
session target ipv4:10.0.0.2
codec g711ulaw
no vad
corlist incoming AnalogUser
corlist outgoing callAnalog
exit

voice-port <FXO_R1>
signal loop-start
connection plar 1002
no shutdown
exit

dial-peer voice 6 pots
destination-pattern 9T
port <FXO_R1>
corlist incoming FromPSTN
corlist outgoing callPSTN
exit

end
copy running-config startup-config
```

`PRI` здесь — имя класса внутренних IP-звонков, не настройка физического PRI. Класс `Analog` разрешает направление к удалённым номерам 200x по WAN.

В VoIP dial-peer цифры не отрезаются. В POTS dial-peer 6 явно заданная ведущая **9** удаляется перед передачей внешнего номера в FXO. Сигнализация loop-start должна соответствовать учебной АТС.

**Отличие от ошибочного примера методички:** в Type1 нет PSTN. Иначе телефон 1001 получил бы именно тот доступ, который задание требует запретить.

## 6. Проверки

Сначала проверить питание, регистрацию и базовые звонки **до назначения COR**. Затем применить раздел 5 и сравнить результаты.

### Сеть

R1:

```text
show ip interface brief
show ip route
show ip dhcp binding
show ephone
ping 10.0.0.2
```

R2:

```text
show ip interface brief
show ip route
show voice port summary
ping 10.0.0.1
ping 192.168.10.1
```

SwitchA:

```text
show interfaces status
show vlan brief
show interfaces fastEthernet0/2 switchport
```

### Голос и ограничения — R1

```text
show dial-peer cor
show dial-peer voice summary
show voice port summary
show ephone-dn summary
show telephony-service dial-peer
show call active voice brief
show running-config
```

Если конкретная команда `show` отсутствует в выбранном IOS, использовать поддерживаемые команды и `show running-config`.

### Звонки

| Проверка | Ожидаемый результат |
| --- | --- |
| 1001 → 1002 | Разрешено |
| 1001 → 2001 / 2002 по WAN | Разрешено |
| 1001 → 9 + внешний номер | Запрещено COR |
| 1002 → 1001 / 2001 / 2002 | Разрешено |
| 1002 → 9 + внешний номер | Разрешено |
| 2001 / 2002 → 1001 / 1002 | Разрешено |
| 2001 / 2002 → 9 + внешний номер | Разрешено |
| PSTN → 1002 через PLAR | Разрешено |
| PSTN → 2001 / 2002 | Запрещено COR |

Для PSTN нужен реально достижимый номер учебной АТС. Отсутствие линии не является доказательством запрета COR. Проверять новые вызовы в каждом направлении, завершать предыдущий разговор.

### Проверка запрета PSTN → аналоговый телефон

По умолчанию FXO направляет входящий вызов на 1002. Чтобы проверить именно запрет к аналоговому телефону, временно изменить назначение PLAR на R1:

```text
configure terminal
voice-port <FXO_R1>
no connection plar
connection plar 2001
exit
end
```

Позвонить с внешнего телефона на учебную линию FXO: **2001 не должен звонить**. Для контрольного сравнения временно разрешить это направление:

```text
configure terminal
dial-peer cor list FromPSTN
member Analog
exit
end
```

Повторить тот же вызов: **2001 должен звонить**. Затем восстановить ограничения и входящий номер:

```text
configure terminal
dial-peer cor list FromPSTN
no member Analog
exit
voice-port <FXO_R1>
no connection plar
connection plar 1002
exit
end
copy running-config startup-config
```

## 7. Дополнительные сервисы

При поддержке функций и наличии нужных кнопок:
- Перевод: **1001 → 1002**, ответ → на 1002 **Transfer → 2001 → Transfer**.
- Конференция: разговор **1001 ↔ 1002** → **Confrn → 2001 → ответ → Confrn**.
- Перехват: **2001 звонит 1001**, на 1001 не отвечать → на 1002 **PickUp → 1001**, при необходимости **Answer**.

Базовая конфигурация CME не гарантирует поддержку этих функций на любом образе или симуляторе. Не отмечать их как выполненные без успешной проверки.

## 8. Что выполнить в твоём Packet Tracer

1. Можно настроить коммутатор, DHCP, CME для двух IP-телефонов и Serial-сеть. Для обоих маршрутизаторов можно использовать 2811 с Serial-модулями; имена интерфейсов заменить фактическими.
2. Проверить регистрацию, звонки **1001 ↔ 1002**, IP-связь между маршрутизаторами.
3. Не вводить блоки физических FXS/FXO, если соответствующих портов нет.
4. **Раздел COR не выполнен:** команда `dial-peer cor` в используемой модели отсутствует. `dial-peer voice` не заменяет COR, а Home VoIP не создаёт физический FXO-порт на маршрутизаторе.

Фраза для отчёта: «Сетевая часть и базовая IP-телефония настроены. Ограничения COR и физические FXS/FXO не проверены из-за отсутствия необходимых функций в используемом симуляторе».

## 9. Сохранение

На всех устройствах IOS:

```text
copy running-config startup-config
```

На запрос имени файла — **Enter**. Проект Packet Tracer: **File → Save As → Лабораторная_6.pkt**.

[Контрольные вопросы и ответы](labs-control-answers.md#лабораторная-работа-6).

Источник по логике ограничений: [Cisco — Configure Class of Restrictions](https://www.cisco.com/c/en/us/support/docs/voice/call-routing-dial-plans/42720-configuring-cor.html).

Команды и ожидаемые результаты подготовлены по методичке; фактические звонки на оборудовании в рамках подготовки файла не выполнялись.
