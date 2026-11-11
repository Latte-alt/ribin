# Лабораторная работа 1–2 — iSCSI на Debian/Ubuntu

[Методичка](../Лаб%20раб%201-2%20iSCSI.docx). Команды выполнять на учебном сервере Debian/Ubuntu. Это Linux-стенд, не Cisco Packet Tracer.

## 1. Подготовка

Для настройки цели нужен один Linux-сервер; для проверки подключения — второй компьютер (инициатор) с сетевой доступностью сервера.

- В методичке разрешён клиент **192.168.111.150**. Если у твоего клиента другой адрес, заменить его в `initiator-address`.
- IP сервера методичка не задаёт: узнать его через `ip -br address`. В дополнительной проверке ниже заменить `SERVER_IP` реальным адресом сервера.
- Создавать диск в новом файле: если `/storage/lun0.img` уже существует, команду `dd` поверх него не выполнять.

Все последующие серверные команды — от root:

```bash
sudo -i
ip -br address
apt update
apt install -y tgt
mkdir -p /storage
mkdir -p /etc/tgt/conf.d
```

## 2. Создание диска — выбрать один вариант

### Основной вариант: фиксированный диск 2 ГиБ

```bash
dd if=/dev/zero of=/storage/lun0.img bs=1M count=2048
ls -lh /storage/lun0.img
du -h /storage/lun0.img
```

### Альтернатива из методички: разреженный диск 200 ГиБ

**Выполнять вместо предыдущего `dd`, не после него.** Требуется файловая система с поддержкой разреженных файлов. Место будет расходоваться по мере записи данных.

```bash
dd if=/dev/zero of=/storage/lun0.img bs=1 count=0 seek=200G
ls -lh /storage/lun0.img
du -h /storage/lun0.img
```

Для преобразования обычного файла в разреженную копию в методичке дана команда `cp --sparse=always filename newfilename`. Это отдельная операция, для основного выполнения не нужна.

## 3. Настройка цели

Проверить, что основной файл подключает конфигурации из `conf.d`:

```bash
cat /etc/tgt/targets.conf
```

В нём должна быть активная строка:

```text
include /etc/tgt/conf.d/*.conf
```

Если её нет — добавить один раз:

```bash
printf '\ninclude /etc/tgt/conf.d/*.conf\n' >> /etc/tgt/targets.conf
```

Создать отдельную конфигурацию цели. Перед выполнением проверить IP инициатора:

```bash
cat > /etc/tgt/conf.d/debain-test.conf <<'EOF'
<target iqn.2009-02.lab.interface31:deban-test-target>
    backing-store /storage/lun0.img
    initiator-address 192.168.111.150
    incominguser andrey secretpass12
</target>
EOF

chmod 600 /etc/tgt/conf.d/debain-test.conf
service tgt restart
```

IQN и учебная учётная запись сохранены из методички. Используется имя файла **lun0.img с цифрой 0**, совпадающее с командой создания диска.

## 4. Проверка сервера — обязательная часть методички

```bash
systemctl status tgt --no-pager
tgtadm --mode target --op show
ss -lntp | grep ':3260'
```

Ожидается:

- Служба `tgt` работает, сервер слушает TCP **3260**.
- Цель **iqn.2009-02.lab.interface31:deban-test-target**, драйвер **iscsi**, состояние **ready**.
- **LUN 1** — диск с путём **/storage/lun0.img**, `Online: Yes`; для основного варианта размер около **2147 MB** (2 ГиБ).
- В списке доступа — IP клиента **192.168.111.150**, в учётных записях — **andrey**.
- **LUN 0** типа controller — служебное устройство, а не созданный диск.

При ошибке:

```bash
journalctl -u tgt -n 50 --no-pager
cat /etc/tgt/conf.d/debain-test.conf
ls -lh /storage/lun0.img
```

## 5. Дополнительно: проверка с Linux-клиента

Методичка заканчивается проверкой цели. Этот раздел позволяет подтвердить подключение инициатора. Выполнять **на втором компьютере**, адрес которого разрешён в конфигурации сервера.

```bash
sudo -i
apt update
apt install -y open-iscsi
systemctl start iscsid
ip -br address
```

Подставить реальный адрес сервера вместо `SERVER_IP` в первой строке:

```bash
ISCSI_SERVER='SERVER_IP'
ISCSI_TARGET='iqn.2009-02.lab.interface31:deban-test-target'

ping -c 4 "$ISCSI_SERVER"
iscsiadm -m discovery -t sendtargets -p "$ISCSI_SERVER:3260"

iscsiadm -m node -T "$ISCSI_TARGET" -p "$ISCSI_SERVER:3260" --op update -n node.session.auth.authmethod -v CHAP
iscsiadm -m node -T "$ISCSI_TARGET" -p "$ISCSI_SERVER:3260" --op update -n node.session.auth.username -v andrey
iscsiadm -m node -T "$ISCSI_TARGET" -p "$ISCSI_SERVER:3260" --op update -n node.session.auth.password -v secretpass12

iscsiadm -m node -T "$ISCSI_TARGET" -p "$ISCSI_SERVER:3260" --login
iscsiadm -m session
lsblk -o NAME,SIZE,TYPE,MODEL,MOUNTPOINT
ls -l /dev/disk/by-path/*iscsi*
```

Ожидается активная iSCSI-сессия и новый диск **2 ГиБ** (либо **200 ГиБ**, если выбран альтернативный вариант). Форматирование и монтирование методичкой не требуются.

На сервере после подключения повторить:

```bash
tgtadm --mode target --op show
```

В `I_T nexus information` должна появиться сессия инициатора с его IP. Если подключения нет: проверить IP в `initiator-address`, сетевой доступ к TCP 3260 и CHAP `andrey / secretpass12`. Если включён межсетевой экран, разрешить TCP 3260 от адреса клиента по правилам своего стенда.

После проверки отключить сессию **на клиенте**, пока диск не используется:

```bash
iscsiadm -m node -T "$ISCSI_TARGET" -p "$ISCSI_SERVER:3260" --logout
```

## 6. Вывод

После успешной проверки можно записать: «Настроена iSCSI-цель Linux SCSI Target. Создан файловый диск, задан разрешённый IP инициатора и настроена CHAP-аутентификация. Проверены состояние цели и параметры предоставляемого LUN».

Если выполнен раздел 5, добавить: «Клиент подключился к цели и обнаружил предоставленный диск».

Команды подготовлены по методичке; на виртуальных машинах здесь не запускались. Не подключать обычную файловую систему одного LUN одновременно на запись с нескольких независимых клиентов.

Справочники: [Debian — targets.conf](https://manpages.debian.org/testing/tgt/targets.conf.5.en.html), [Debian — open-iscsi](https://wiki.debian.org/SAN/iSCSI/open-iscsi).
