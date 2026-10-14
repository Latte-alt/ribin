# Лабораторная работа № 3

Настройка Cisco CallManager Express на Cisco 2811 в Packet Tracer.

[Задание: 3лаб.docx](https://github.com/Latte-alt/ribin/blob/main/3%D0%BB%D0%B0%D0%B1.docx).

## 1. Подключение

Устройства: **2811**, **3560-24PS**, **3 телефона 7960**. Кабель: **Copper Straight-Through**.

| Устройство | Порт коммутатора |
| --- | --- |
| Router FastEthernet0/0 | FastEthernet0/1 |
| Телефон 1, порт Switch | FastEthernet0/2 |
| Телефон 2, порт Switch | FastEthernet0/3 |
| Телефон 3, порт Switch | FastEthernet0/4 |

При отсутствии питания: телефон → **Physical** → установить **IP_PHONE_POWER_ADAPTER**.

Используется VLAN 1 по командам методички. Несмотря на подпись «Trunk» на рисунке, для данной конфигурации подключение маршрутизатора настроено как access: на маршрутизаторе нет подинтерфейсов 802.1Q.

## 2. Очистка старых настроек

**Для новых устройств пропустить. Команды удаляют прежнюю конфигурацию.**

Маршрутизатор:

```text
enable
erase startup-config
reload
```

Коммутатор:

```text
enable
erase startup-config
delete flash:vlan.dat
reload
```

Подтверждения — **Enter**. На предложение сохранить конфигурацию — **no**. Если `vlan.dat` отсутствует, продолжить. После перезагрузки на `initial configuration dialog` — **no**.

## 3. Коммутатор SwitchA

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

interface range fastEthernet0/2 - 5
switchport mode access
switchport access vlan 1
switchport voice vlan 1
spanning-tree portfast
no shutdown
exit

interface vlan 1
ip address 192.168.10.2 255.255.255.0
no shutdown
exit
ip default-gateway 192.168.10.1

end
copy running-config startup-config
```

На запрос имени файла — **Enter**. Адрес `192.168.10.2` добавлен для проверки связи и удалённого входа на маршрутизатор.

## 4. Маршрутизатор CMERouter

```text
enable
configure terminal
hostname CMERouter
no ip domain-lookup
enable secret cisco
service password-encryption

line vty 0 4
password cisco
login
logging synchronous
transport input telnet
exit

line console 0
password cisco
login
logging synchronous
exit

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
max-dn 5
max-ephones 5
ip source-address 192.168.10.1 port 2000
auto assign 1 to 5
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

ephone-dn 4
number 54004
exit

ephone-dn 5
number 54005
exit

end
copy running-config startup-config
```

На запрос имени файла — **Enter**. Пароль консоли, Telnet и режима `enable`: **cisco**.

Вместо двух диапазонов из методички используется `auto assign 1 to 5`: линия 6 не входит в `max-dn 5`.

## 5. Телефоны

1. Режим **Realtime** → дождаться загрузки телефонов; при необходимости нажать **Fast Forward Time**.
2. Телефон → **GUI** → проверить номера **54001**, **54002**, **54003**. Порядок назначения зависит от порядка регистрации.
3. Если сохранились старые настройки: при наличии соответствующего меню **Settings → Network Configuration → Erase Configuration → Save**, затем перезагрузить телефон. На физическом 7960 настройки разблокируются последовательностью `**#`; в Packet Tracer доступность меню зависит от версии.

## 6. Команды проверки

### Маршрутизатор

```text
enable
show ip interface brief
show ip dhcp binding
show ip dhcp pool
show ip route
show running-config
show ephone
```

Если поддерживается:

```text
show ephone registered
```

Ожидается: `Fa0/0` — **192.168.10.1, up/up**; DHCP выдал адреса трём телефонам; телефоны зарегистрированы. Дополнительные `ip route` для одной подключённой сети не нужны.

### Коммутатор

```text
enable
show interfaces status
show vlan brief
show interfaces fastEthernet0/2 switchport
show interfaces fastEthernet0/3 switchport
show interfaces fastEthernet0/4 switchport
show ip interface brief
ping 192.168.10.1
```

Ожидается: подключённые порты — **connected**; телефонные порты — **Access VLAN 1 / Voice VLAN 1**; `Vlan1` — **192.168.10.2, up/up**.

### Проверка Telnet с коммутатора

```text
telnet 192.168.10.1
```

На `Password:` ввести **cisco**. В открывшейся сессии:

```text
enable
```

На `Password:` снова ввести **cisco**, затем:

```text
show ip interface brief
exit
```

### Ping телефонов с маршрутизатора

```text
ping 192.168.10.11
ping 192.168.10.12
ping 192.168.10.13
```

Подставить фактические адреса телефонов из `show ip dhcp binding`.

## 7. Проверка звонков и сервисов

Телефон → **GUI** → снять трубку → набрать номер → на принимающем телефоне снять трубку.

- Проверить **54001 → 54002**, **54002 → 54003**, **54003 → 54001**.
- **Перевод:** звонок 54001 → 54002; на 54002 ответить → **Transfer** → 54003 → **Transfer** → ответить на 54003.
- **Конференция:** разговор 54001 ↔ 54002 → **Confrn** → 54003 → ответить на 54003 → снова **Confrn** на инициаторе.
- **Перехват:** 54001 звонит 54002, второй не отвечает → на 54003 **PickUp** → 54002 → при необходимости **Answer**.

Дополнительные сервисы проверять только при наличии поддержки и нужных кнопок. Базовые команды методички не гарантируют работу перевода, конференции и перехвата в каждой версии Packet Tracer. Если функция недоступна, указать это в отчёте.

## 8. Сохранение

На маршрутизаторе и коммутаторе:

```text
copy running-config startup-config
```

На запрос имени файла — **Enter**. **File → Save As → Лабораторная_3.pkt**.
