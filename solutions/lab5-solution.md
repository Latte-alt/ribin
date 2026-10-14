# Лабораторная работа №5 — ограничения звонков COR

[Методичка](../%D0%BB%D0%B0%D0%B1%205.docx). Настройка и проверка.

## 1. Подготовка

Схема из методички: **2811**, **2960-24TT**, **2 IP Phone 7960**, **Home-VoIP-PT**, **Analog-Phone-PT**.

| Откуда | Куда | Кабель |
| --- | --- | --- |
| Router Fa0/0 | Switch Fa0/1 | Copper Straight-Through |
| IP-телефон 1001, порт Switch | Switch Fa0/2 | Copper Straight-Through |
| IP-телефон 1002, порт Switch | Switch Fa0/3 | Copper Straight-Through |
| Home VoIP, Ethernet | Switch Fa0/4 | Copper Straight-Through |
| Analog Phone, Phone | Home VoIP, Phone | Phone |

На IP-телефонах установить адаптеры питания во вкладке **Physical**. Использовать новые устройства. На вопрос маршрутизатора `initial configuration dialog` ответить **no**.

Записать MAC двух IP-телефонов и Ethernet MAC **Home VoIP**, наведя указатель на устройство или открыв настройки его интерфейса. Подставить их вместо `<MAC_IP_1001>`, `<MAC_IP_1002>`, `<MAC_HOME_VOIP>`. Формат: `XXXX.XXXX.XXXX`, без угловых скобок. У аналогового аппарата отдельный Ethernet MAC не требуется.

**Ограничение схемы:** Home VoIP подключается по Ethernet и регистрируется как ATA. Это не физический FXS-порт маршрутизатора. PSTN и FXO на рисунке отсутствуют. Блок для физических портов приведён отдельно в разделе 7.

## 2. Коммутатор

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

end
copy running-config startup-config
```

На запрос имени файла — **Enter**.

## 3. Маршрутизатор — сеть и телефоны

Заменить три MAC-адреса перед вставкой.

```text
enable
configure terminal
hostname CMERouter
no ip domain-lookup

interface fastEthernet0/0
ip address 192.168.10.1 255.255.255.0
no shutdown
exit

ip dhcp excluded-address 192.168.10.1 192.168.10.10
ip dhcp pool VOICE
network 192.168.10.0 255.255.255.0
default-router 192.168.10.1
option 150 ip 192.168.10.1
exit

telephony-service
max-dn 3
max-ephones 3
ip source-address 192.168.10.1 port 2000
exit

ephone-dn 1
number 1001
exit
ephone-dn 2
number 1002
exit
ephone-dn 3
number 2001
exit

ephone 1
mac-address <MAC_IP_1001>
type 7960
button 1:1
exit
ephone 2
mac-address <MAC_IP_1002>
type 7960
button 1:2
exit
ephone 3
mac-address <MAC_HOME_VOIP>
type ata
button 1:3
exit

end
copy running-config startup-config
```

Home VoIP → **Config → Settings → Server Address: 192.168.10.1**. Для Ethernet выбрать DHCP, если доступно. Дождаться регистрации. Если необходимо, перезапустить телефоны и Home VoIP.

**До применения COR** проверить все направления между **1001**, **1002**, **2001**. Если звонки не работают уже сейчас, сначала исправить регистрацию, питание и DHCP.

## 4. Проверка поддержки COR

На маршрутизаторе:

```text
enable
configure terminal
dial-peer cor ?
```

Если `custom` и `list` доступны, выполнить раздел 5. Если команда не поддерживается, эта версия Packet Tracer не позволяет выполнить COR-часть лабораторной. Для неё нужен IOS с CME/COR на подходящем оборудовании или в совместимой лабораторной среде. Не считать обычный неудачный звонок доказательством работы COR.

## 5. Классы ограничений

На маршрутизаторе, при поддержке COR:

```text
end
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
member PSTN
exit
dial-peer cor list Type2
member PSTN
member Analog
exit
dial-peer cor list Type3
member PRI
member Analog
member PSTN
exit

ephone-dn 1
corlist incoming Type1
corlist outgoing callPRI
exit
ephone-dn 2
corlist incoming Type3
corlist outgoing callPRI
exit
ephone-dn 3
corlist incoming Type2
corlist outgoing callAnalog
exit

end
copy running-config startup-config
```

`Type1` и `Type2` сохранены как в методичке. `Type3` добавлен для второго IP-телефона, чтобы были разрешённые и запрещённые вызовы для сравнения.

`PRI` здесь — название класса внутренних звонков, а не настройка физического интерфейса PRI. Вызов разрешён, если incoming-список источника содержит все элементы outgoing-списка назначения.

## 6. Проверка ограничений

### Команды маршрутизатора

```text
show ip interface brief
show ip dhcp binding
show ephone
show dial-peer cor
show dial-peer voice summary
show running-config
```

Дополнительные команды из методички — если поддерживаются:

```text
show ephone-dn summary
show telephony-service ephone-dn
show telephony-service dial-peer
show telephony-service all
```

Коммутатор:

```text
show interfaces status
show vlan brief
show interfaces fastEthernet0/2 switchport
```

### Звонки после применения COR

| Вызов | Ожидаемый результат |
| --- | --- |
| 1001 → 1002 | Запрещён: в Type1 нет PRI |
| 1001 → 2001 | Запрещён: в Type1 нет Analog |
| 1002 → 1001 | Разрешён: в Type3 есть PRI |
| 1002 → 2001 | Разрешён: в Type3 есть Analog |
| 2001 → 1001 | Запрещён: в Type2 нет PRI |
| 2001 → 1002 | Запрещён: в Type2 нет PRI |

Для каждого теста снять трубку, набрать номер и ответить на принимающем телефоне, если вызов разрешён. Проверять именно инициирование новых звонков: запрет исходящего вызова не запрещает принимать разрешённый входящий вызов.

## 7. Физические FXS/FXO и PSTN — вариант для IOS

Этот раздел выполняется **вместо Home VoIP**, если есть маршрутизатор с голосовыми модулями и IOS с CME/COR. FXS подключается к аналоговому телефону, FXO — к линии АТС/PSTN. Одних команд без портов и работающей линии недостаточно.

Узнать номера физических портов:

```text
show voice port summary
```

Подставить фактические порты вместо `<FXS_PORT>` и `<FXO_PORT>`. Для примера внешние номера набираются с префиксом **9**; план внешней АТС должен быть настроен отдельно.

Сначала выполнить разделы 2–3 только для двух IP-телефонов и раздел 5 только для `ephone-dn 1` и `ephone-dn 2`. **Не создавать `ephone 3` и `ephone-dn 3`**: номер 2001 в этом варианте обслуживает физический FXS.

```text
enable
configure terminal

voice-port <FXS_PORT>
signal loop-start
station-id number 2001
no shutdown
exit

voice-port <FXO_PORT>
signal loop-start
connection plar 1002
no shutdown
exit

dial-peer voice 2 pots
destination-pattern 2001
port <FXS_PORT>
corlist incoming Type2
corlist outgoing callAnalog
exit

dial-peer voice 6 pots
destination-pattern 9T
port <FXO_PORT>
corlist incoming Type3
corlist outgoing callPSTN
exit

end
copy running-config startup-config
```

POTS dial-peer удаляет явно заданную ведущую цифру **9** при отправке номера в FXO. `connection plar 1002` направляет входящий вызов с FXO на IP-телефон 1002. Конкретную сигнализацию и план набора согласовать с внешней АТС; показан вариант loop-start.

### Проверка физических портов и PSTN

```text
show voice port summary
show dial-peer voice summary
show dial-peer cor
show call active voice brief
```

- Повторить внутренние и аналоговые звонки из таблицы раздела 6.
- С 1001, 1002 и 2001 набрать **9 + рабочий номер внешней АТС**: COR разрешает направление PSTN всем трём классам.
- С внешнего телефона позвонить на линию FXO: вызов должен поступить на 1002.

## 8. Дополнительные сервисы

Для проверки перевода и конференции сначала временно дать всем трём источникам полный класс. Для схемы с Home VoIP:

```text
configure terminal
ephone-dn 1
corlist incoming Type3
exit
ephone-dn 3
corlist incoming Type3
exit
end
```

В варианте с физическим FXS вместо изменения `ephone-dn 3` изменить его входящий dial-peer:

```text
configure terminal
dial-peer voice 2 pots
corlist incoming Type3
exit
end
```

Если доступны функции и кнопки:

- **Перевод:** 1001 звонит 1002 → ответ → на 1002 **Transfer → 2001 → Transfer**.
- **Конференция:** разговор 1001 ↔ 1002 → **Confrn → 2001 → ответ → Confrn**.
- **Перехват:** 2001 звонит 1001, не отвечать → на 1002 **PickUp → 1001**, затем при необходимости **Answer**.

Если функции не поддерживаются симулятором или базовой конфигурацией CME, отметить это. После проверки восстановить исходные классы.

Для Home VoIP:

```text
configure terminal
ephone-dn 1
corlist incoming Type1
exit
ephone-dn 3
corlist incoming Type2
exit
end
copy running-config startup-config
```

Для физического FXS:

```text
configure terminal
ephone-dn 1
corlist incoming Type1
exit
dial-peer voice 2 pots
corlist incoming Type2
exit
end
copy running-config startup-config
```

## 9. Сохранение

На запрос имени файла — **Enter**. В Packet Tracer: **File → Save As → Лабораторная_5.pkt**.

## Исправления и уточнения

- В методичке не заданы `destination-pattern`, физические `port` или `session target` для ряда dial-peer; одних COR-списков недостаточно для маршрутизации вызова.
- Для IP-телефонов outgoing COR назначен их `ephone-dn`; произвольный POTS dial-peer без связи с номером не ограничивает внутренние звонки.
- Для физического FXO используется POTS dial-peer. VoIP dial-peer нужен для IP-шлюза оператора и требует его реального адреса.
- Отсутствующий incoming COR не означает запрет: без него ограничения можно обойти. Поэтому каждому источнику задан класс.

[Ответы на контрольные вопросы](labs-control-answers.md#лабораторная-работа-5).

Источники: [Cisco — COR](https://www.cisco.com/c/en/us/support/docs/voice/call-routing-dial-plans/42720-configuring-cor.html), [команда corlist](https://www.cisco.com/c/en/us/td/docs/voice_ip_comm/cucme/command/reference/cme_cr/cme_cr_chapter_010.html), [FXS/FXO](https://www.cisco.com/c/en/us/support/docs/voice/analog-signaling-e-m-did-fxs-fxo/214636-troubleshoot-and-monitor-analog-ports.html).
