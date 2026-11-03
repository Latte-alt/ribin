# Лабораторная №9 — VoIP в сети с VLAN

[Методичка](../%D0%BB%D0%B0%D0%B1%209.docx). Cisco Packet Tracer. Использовать новую схему.

## 1. Устройства и подключение

**2811 — 1 шт., 3560-24PS — 1 шт., Server-PT — 1 шт., IP Phone 7960 — 3 шт., PC-PT — 3 шт.** Дополнительные модули не нужны. Телефоны получают питание от 3560-24PS через PoE.

Все соединения — **Copper Straight-Through**:

| Откуда | Куда |
| --- | --- |
| Router Fa0/0 | Switch Gi0/1 |
| Server FastEthernet0 | Switch Fa0/5 |
| Первый телефон, порт Switch | Switch Fa0/1 |
| PC0 FastEthernet0 | Первый телефон, порт PC |
| Второй телефон, порт Switch | Switch Fa0/2 |
| PC1 FastEthernet0 | Второй телефон, порт PC |
| Третий телефон, порт Switch | Switch Fa0/6 |
| PC2 FastEthernet0 | Switch Fa0/4 |

На вопрос маршрутизатора `initial configuration dialog` — **no**.

## 2. Коммутатор

```text
enable
configure terminal
hostname VoiceSwitch
no ip domain-lookup

vlan 100
name DATA
exit
vlan 110
name VOICE
exit
vlan 120
name SERVER
exit

interface gigabitEthernet0/1
switchport trunk encapsulation dot1q
switchport mode trunk
switchport trunk allowed vlan 100,110,120
no shutdown
exit

interface range fastEthernet0/1 - 2
switchport mode access
switchport access vlan 100
switchport voice vlan 110
spanning-tree portfast
no shutdown
exit

interface fastEthernet0/6
switchport mode access
switchport access vlan 100
switchport voice vlan 110
spanning-tree portfast
no shutdown
exit

interface fastEthernet0/4
switchport mode access
switchport access vlan 100
spanning-tree portfast
no shutdown
exit

interface fastEthernet0/5
switchport mode access
switchport access vlan 120
spanning-tree portfast
no shutdown
exit

end
copy running-config startup-config
```

На запрос имени файла — **Enter**. Если модель не принимает `switchport trunk encapsulation dot1q`, пропустить только эту команду и проверить trunk.

## 3. Маршрутизатор — VLAN, DHCP relay, CME

```text
enable
configure terminal
hostname VoiceRouter
no ip domain-lookup

interface fastEthernet0/0
no ip address
no shutdown
exit

interface fastEthernet0/0.100
encapsulation dot1Q 100
ip address 10.10.100.1 255.255.255.0
ip helper-address 10.10.120.100
exit

interface fastEthernet0/0.110
encapsulation dot1Q 110
ip address 10.10.110.1 255.255.255.0
ip helper-address 10.10.120.100
exit

interface fastEthernet0/0.120
encapsulation dot1Q 120
ip address 10.10.120.1 255.255.255.0
exit

telephony-service
max-ephones 5
max-dn 5
ip source-address 10.10.110.1 port 2000
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

На запрос имени файла — **Enter**. DHCP-пулы на маршрутизаторе **не создавать**: адреса выдаёт отдельный Server-PT. Статические маршруты не нужны — все три сети подключены непосредственно.

## 4. Server-PT — IP и DHCP

**Desktop → IP Configuration → Static:**

| Поле | Значение |
| --- | --- |
| IP Address | 10.10.120.100 |
| Subnet Mask | 255.255.255.0 |
| Default Gateway | 10.10.120.1 |
| DNS Server | 0.0.0.0 |

**Services → DHCP → On.** Существующий `serverPool` изменить и нажать **Save**; пулы `Data` и `Voice` создать через **Add**.

| Поле | serverPool | Data | Voice |
| --- | --- | --- | --- |
| Default Gateway | 10.10.120.1 | 10.10.100.1 | 10.10.110.1 |
| DNS Server | 0.0.0.0 | 0.0.0.0 | 0.0.0.0 |
| Start IP Address | 10.10.120.110 | 10.10.100.10 | 10.10.110.10 |
| Subnet Mask | 255.255.255.0 | 255.255.255.0 | 255.255.255.0 |
| Maximum Users | 100 | 200 | 200 |
| TFTP Server | 0.0.0.0 | 0.0.0.0 | **10.10.110.1** |

**DHCP-сервер — 10.10.120.100; TFTP/CME — 10.10.110.1.** Адреса `.0` из скриншота методички не использовать как начало выдачи: это адреса сетей /24.

## 5. ПК и телефоны

- На всех ПК: **Desktop → IP Configuration → DHCP**.
- На телефонах оставить **DHCP**, дождаться загрузки и появления номеров. При необходимости перезагрузить телефоны после настройки сервера и CME.
- ПК должны получить **10.10.100.10–209/24**, шлюз **10.10.100.1**.
- Телефоны — **10.10.110.10–209/24**, шлюз и TFTP **10.10.110.1**.
- Номера **54001–54003** назначаются по порядку регистрации. Проверять номер на экране, а не имя устройства на схеме.

## 6. Проверка

### Коммутатор

```text
enable
show vlan brief
show interfaces trunk
show interfaces status
show interfaces fastEthernet0/1 switchport
show interfaces fastEthernet0/2 switchport
show interfaces fastEthernet0/6 switchport
```

Ожидается: Gi0/1 — trunk с VLAN **100,110,120**; Fa0/1,2,6 — **Access VLAN 100 / Voice VLAN 110**; Fa0/4 — access 100; Fa0/5 — access 120. Voice VLAN проверять через `show interfaces ... switchport`.

### Маршрутизатор

```text
enable
show ip interface brief
show ip route
show ip interface fastEthernet0/0.100
show ip interface fastEthernet0/0.110
show ephone
show running-config
ping 10.10.120.100
```

Ожидается: Fa0/0 и подинтерфейсы — **up/up**, три подключённые сети, helper **10.10.120.100** на .100 и .110, три зарегистрированных телефона. `show ip dhcp binding` на маршрутизаторе не показывает аренды отдельного Server-PT.

### Сервер — Desktop → Command Prompt

```text
ping 10.10.120.1
ping 10.10.110.1
ping 10.10.100.1
```

### Каждый ПК — Desktop → Command Prompt

```text
ipconfig /all
ping 10.10.100.1
ping 10.10.110.1
ping 10.10.120.1
ping 10.10.120.100
```

Также выполнить ping между фактическими адресами ПК из `ipconfig`. Для проверки выдачи адресов отдельным сервером: **Simulation → фильтр DHCP → повторный запрос DHCP на ПК**; проследить запрос через маршрутизатор к Server-PT.

### Звонки

**GUI телефона → снять трубку → набрать номер → ответить на втором телефоне.** После каждой проверки завершать разговор.

- **54001 → 54002** и обратно.
- **54002 → 54003** и обратно.
- **54003 → 54001** и обратно.

Если IP есть, но номера нет: проверить **Voice → TFTP Server 10.10.110.1**, адрес CME и наличие трёх `ephone-dn`; перезагрузить телефоны. Если IP нет: проверить VLAN, trunk, DHCP **On**, соответствующий пул и `ip helper-address`.

## 7. Сохранение

На коммутаторе и маршрутизаторе:

```text
copy running-config startup-config
```

На запрос имени — **Enter**. **File → Save As → Лабораторная_9.pkt**.

Цель достигнута после получения ПК и телефонами адресов **от Server-PT**, успешной связи между VLAN и звонков между всеми тремя номерами. PSTN, COR, EIGRP и Serial в этой лабораторной не требуются. Конфигурация в Packet Tracer здесь не запускалась; выше указаны ожидаемые результаты.

Исправления методички: начальные DHCP-адреса заменены на допустимые; явно настроены relay и все телефонные порты; для трёх телефонов достаточно ёмкости CME 5/5 вместо потенциально неподдерживаемых 42/144.

Справочники Cisco: [DHCP relay](https://www.cisco.com/c/en/us/support/docs/ip/dynamic-address-allocation-resolution/27470-100.html), [сетевые параметры CME и option 150](https://www.cisco.com/c/en/us/td/docs/voice_ip_comm/cucme/admin/configuration/manual/cmeadm/cmenetwk.html).
