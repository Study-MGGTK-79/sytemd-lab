# Лабораторная работа: Архитектура и администрирование Linux systemd

**Цель работы:** 
Изучить устройство и принципы работы подсистемы инициализации `systemd`, научиться создавать и настраивать пользовательские сервисы (`.service`), настраивать периодические задачи с помощью таймеров (`.timer`), управлять зависимостями, накладывать ограничения на ресурсы (cgroups) и работать с журналом событий (`journalctl`).

**Требования к окружению:**
* ОС семейства Linux с systemd (Ubuntu 20.04+, Debian 11+, Rocky Linux 8+, Fedora, Arch и др.).
* Права администратора (`sudo`).

---

## ЧАСТЬ 1. Теоретическая справка (Лекция)

### 1. Что такое systemd?
`systemd` — это системный менеджер и система инициализации в современных дистрибутивах Linux (занимает PID 1). Он пришел на смену классической системе `SysVinit` и `Upstart`.

**Ключевые преимущества systemd:**
* **Параллелизация запуска:** службы стартуют параллельно на основе сокетов и шины D-Bus, что ускоряет загрузку ОС.
* **Юниты (Units):** единая абстракция для управления процессами, точками монтирования, сокетами, таймерами.
* **Контроль процессов через cgroups:** дочерние процессы отслеживаются через cgroups ядра, поэтому «демонизация» (двойной `fork`) не позволяет процессу ускользнуть от супервизора.
* **Централизованный журнал:** `systemd-journald` агрегирует stdout, stderr, syslog в единый бинарный индексированный лог.

---

### 2. Типы юнитов (Unit Types)
Каждый ресурс описывается конфигурационным файлом с соответствующим расширением:
* `.service` — системная служба (демон, приложение).
* `.timer` — таймер, альтернатива классическому `cron`.
* `.target` — группа юнитов, определяющая состояние системы (аналог runlevel, например `multi-user.target`, `graphical.target`).
* `.socket` — сокет IPC/сети для отложенной активации сервисов.
* `.mount` / `.automount` — управление точками монтирования файловых систем.
* `.path` — отслеживание событий файловой системы (in-otify) для запуска юнитов.
* `.slice` — иерархический узел cgroups для разделения ресурсов.

**Пути хранения unit-файлов (по приоритету загрузки):**
1. `/etc/systemd/system/` — пользовательские и локально переопределенные юниты (наивысший приоритет).
2. `/run/systemd/system/` — временные юниты, созданные во время работы системы.
3. `/usr/lib/systemd/system/` (или `/lib/...`) — системные юниты, установленные пакетным менеджером (не редактируются вручную!).

---

### 3. Анатомия Unit-файла (`.service`)

Файл службы обычно состоит из трех ключевых секций:

```ini
[Unit]
Description=My Custom Application Service
Documentation=https://example.com/docs
After=network.target network-online.target
Wants=network-online.target

[Service]
Type=simple
User=appuser
Group=appuser
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/python3 /opt/myapp/app.py
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5s
Environment="PORT=8080" "ENV=production"
EnvironmentFile=-/etc/default/myapp

[Install]
WantedBy=multi-user.target
```

#### Разбор секций:
* **`[Unit]`** — метаданные и граф зависимостей:
  * `Description` — краткое человекочитаемое описание.
  * `After=` / `Before=` — порядок запуска (не задает жесткую зависимость, только очередность).
  * `Requires=` — жесткая зависимость (если целевой юнит упал или не запустился, текущий завершится).
  * `Wants=` — слабая зависимость (запустить юнит вместе с данным, но ошибка не приведет к падению текущего).
* **`[Service]`** — параметры запуска процесса:
  * `Type=`:
    * `simple` (по умолчанию) — процесс, запущенный в `ExecStart`, является основным.
    * `exec` — аналогичен `simple`, но systemd ждет завершения системного вызова `execve()`.
    * `forking` — для традиционных демонов, которые форкаются в фон (`PIDFile=` обязателен).
    * `oneshot` — для скриптов, выполняющих одну задачу и завершающихся.
    * `notify` — процесс отправляет сигнал готовности через сокет `sd_notify()`.
  * `ExecStart=` — абсолютный путь к бинарнику/скрипту и его аргументы.
  * `Restart=` — условие перезапуска (`no`, `always`, `on-failure`, `on-abort`).
  * `User=` / `Group=` — запуск под непривилегированным пользователем.
* **`[Install]`** — поведение при включении автозагрузки (`systemctl enable`):
  * `WantedBy=multi-user.target` — создает симлинк в `/etc/systemd/system/multi-user.target.wants/`.

---

### 4. Шпаргалка по утилитам `systemctl` и `journalctl`

#### Управление состоянием:
```bash
sudo systemctl start <unit>       # Запустить
sudo systemctl stop <unit>        # Остановить
sudo systemctl restart <unit>     # Перезапустить
sudo systemctl reload <unit>      # Мягкая перезагрузка конфигурации (без даунтайма)
sudo systemctl status <unit>      # Текущий статус и последние строки логов
```

#### Автозагрузка:
```bash
sudo systemctl enable <unit>          # Включить автозапуск (создает симлинк)
sudo systemctl disable <unit>         # Выключить автозапуск
sudo systemctl enable --now <unit>    # Включить и сразу запустить
sudo systemctl is-enabled <unit>      # Проверить статус автозапуска
```

#### Служебные команды:
```bash
sudo systemctl daemon-reload          # Перечитать конфигурации всех юнитов с диска
systemctl list-units --type=service   # Список активных служб
systemctl list-timers                 # Список активных таймеров
sudo systemctl mask <unit>            # Заблокировать юнит (связывает с /dev/null)
```

#### Просмотр логов (`journalctl`):
```bash
journalctl -u <unit> -f               # Логи конкретного юнита в реальном времени (follow)
journalctl -u <unit> -b               # Логи только с текущей загрузки ОС
journalctl -u <unit> --since "1h ago" # Логи за последний час
journalctl -p err..emerg              # Ошибки уровня от Error до Emergency
```

---

## ЧАСТЬ 2. Практические задания

### Задание 1. Разработка и автодемонизация скрипта
1. Написать bash-скрипт `/usr/local/bin/status-monitor.sh`, который каждые 3 секунды дописывает в файл `/var/log/status-monitor.log` текущую дату, uptime и объем свободной оперативной памяти.
2. Создать пользователя `monitor-user` без домашней директории и права входа по паролю.
3. Описать unit-файл `/etc/systemd/system/status-monitor.service`, запускающий данный скрипт от имени `monitor-user`.
4. Настроить политику автоперезапуска `Restart=always` с задержкой 5 секунд.
5. Протестировать работу: запустить службу, включить в автозагрузку, сымитировать аварийное завершение процесса (`kill -9`) и проверить, перезапустил ли его systemd.

---

### Задание 2. Создание systemd-таймера (Альтернатива cron)
1. Написать скрипт `/usr/local/bin/log-cleaner.sh`, который удаляет строки из `/var/log/status-monitor.log`, если размер файла превышает 50 КБ.
2. Создать `oneshot` сервис `/etc/systemd/system/log-cleaner.service`.
3. Создать юнит таймера `/etc/systemd/system/log-cleaner.timer`, настроенный на запуск каждые 2 минуты, а также через 1 минуту после старта ОС.
4. Активировать таймер и проверить его через `systemctl list-timers`.

---

### Задание 3. Ограничение ресурсов (cgroups в systemd)
1. Написать скрипт `/usr/local/bin/mem-stress.sh`, который аллоцирует массив в оперативной памяти.
2. Создать сервис `/etc/systemd/system/mem-stress.service`.
3. Ограничить максимальный объем потребляемой памяти до **64M** (`MemoryMax=64M`) и настроить сброс по OOM-killer.
4. Запустить юнит и с помощью `journalctl` убедиться, что процесс был принудительно остановлен ядром по причине превышения лимита памяти.

---

## ЧАСТЬ 3. Решения и эталонный ход выполнения

### Решение Задания 1: Разработка и автодемонизация скрипта

#### Шаг 1.1: Создание системного пользователя
```bash
sudo useradd -r -s /usr/sbin/nologin monitor-user
```
*Флаг `-r` создает системного пользователя (с UID < 1000), `-s /usr/sbin/nologin` запрещает интерактивный вход.*

#### Шаг 1.2: Создание скрипта
Создадим файл `/usr/local/bin/status-monitor.sh`:
```bash
sudo tee /usr/local/bin/status-monitor.sh > /dev/null << 'EOF'
#!/bin/bash
LOGFILE="/var/log/status-monitor.log"

while true; do
    TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
    LOAD=$(cat /proc/loadavg | awk '{print $1, $2, $3}')
    FREE_MEM=$(free -m | awk '/Mem:/ {print $4}')
    echo "[${TIMESTAMP}] LOAD: ${LOAD} | FREE_MEM: ${FREE_MEM}MB" >> "${LOGFILE}"
    sleep 3
done
EOF
```

Сделаем скрипт исполняемым:
```bash
sudo chmod +x /usr/local/bin/status-monitor.sh
```

Подготовим лог-файл и выдадим права на запись пользователю `monitor-user`:
```bash
sudo touch /var/log/status-monitor.log
sudo chown monitor-user:monitor-user /var/log/status-monitor.log
```

#### Шаг 1.3: Описание unit-файла
Создадим файл `/etc/systemd/system/status-monitor.service`:
```bash
sudo tee /etc/systemd/system/status-monitor.service > /dev/null << 'EOF'
[Unit]
Description=Status Monitor Daemon
After=network.target

[Service]
Type=simple
User=monitor-user
Group=monitor-user
ExecStart=/usr/local/bin/status-monitor.sh
Restart=always
RestartSec=5s

# Стандартный вывод направляем в journald
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF
```

#### Шаг 1.4: Регистрация и запуск сервиса
```bash
# Обязательно перечитываем конфигурации
sudo systemctl daemon-reload

# Включаем автозапуск и сразу стартуем
sudo systemctl enable --now status-monitor.service

# Проверяем статус
sudo systemctl status status-monitor.service
```

#### Шаг 1.5: Проверка автоперезапуска
Найдем PID процесса и принудительно завершим его:
```bash
PID=$(pgrep -f status-monitor.sh)
echo "Current PID: $PID"
sudo kill -9 $PID

# Смотрим статус сразу и через 6 секунд
sleep 1
sudo systemctl status status-monitor.service --no-pager
sleep 5
sudo systemctl status status-monitor.service --no-pager
```
*В выводе `systemctl status` будет видно сообщение: `Scheduled restart job, restart counter is at 1.` и появится новый PID.*

---

### Решение Задания 2: Создание systemd-таймера

#### Шаг 2.1: Создание скрипта очистки
Создадим скрипт `/usr/local/bin/log-cleaner.sh`:
```bash
sudo tee /usr/local/bin/log-cleaner.sh > /dev/null << 'EOF'
#!/bin/bash
TARGET="/var/log/status-monitor.log"
MAX_SIZE=51200 # 50 KB в байтах

if [ -f "$TARGET" ]; then
    SIZE=$(stat -c%s "$TARGET")
    if [ "$SIZE" -gt "$MAX_SIZE" ]; then
        echo "Log size ($SIZE bytes) exceeds limit. Truncating..."
        tail -n 200 "$TARGET" > "${TARGET}.tmp" && mv "${TARGET}.tmp" "$TARGET"
    else
        echo "Log size ($SIZE bytes) is within normal range."
    fi
fi
EOF

sudo chmod +x /usr/local/bin/log-cleaner.sh
```

#### Шаг 2.2: Создание сервиса для таймера
Таймер запускает юнит с **тем же именем**, но расширением `.service`. 
Создадим `/etc/systemd/system/log-cleaner.service`:
```bash
sudo tee /etc/systemd/system/log-cleaner.service > /dev/null << 'EOF'
[Unit]
Description=Clean status monitor log file
Documentation=man:systemd.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/log-cleaner.sh
EOF
```
*Обратите внимание: секция `[Install]` здесь не нужна, так как сервис будет запускаться исключительно по сигналу от таймера, а не напрямую через symlink.*

#### Шаг 2.3: Создание unit-файла таймера
Создадим файл `/etc/systemd/system/log-cleaner.timer`:
```bash
sudo tee /etc/systemd/system/log-cleaner.timer > /dev/null << 'EOF'
[Unit]
Description=Run log-cleaner every 2 minutes
RefuseManualStop=no

[Timer]
# Запуск через 1 минуту после старта системы
OnBootSec=1min
# Запуск каждые 2 минуты
OnUnitActiveSec=2min
# Если система была выключена в момент запуска, выполнить при включении
Persistent=true

[Install]
WantedBy=timers.target
EOF
```

#### Шаг 2.4: Активация и верификация
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now log-cleaner.timer

# Проверяем таймеры
systemctl list-timers log-cleaner.timer
```
Вывод покажет время следующего срабатывания (`NEXT`), время последнего выполнения (`LAST`) и имя связанного сервиса (`ACTIVATES: log-cleaner.service`).

---

### Решение Задания 3: Ограничение ресурсов (cgroups в systemd)

#### Шаг 3.1: Скрипт активного выделения памяти
Создадим скрипт на Python `/usr/local/bin/mem-stress.py`, который динамически выделяет 120 МБ памяти:
```bash
sudo tee /usr/local/bin/mem-stress.py > /dev/null << 'EOF'
#!/usr/bin/env python3
import time
import sys

print("Allocating memory...", flush=True)
try:
    # Выделяем ~120MB памяти в буфер
    data = bytearray(120 * 1024 * 1024)
    # Заполняем данными, чтобы ядро выделило реальные физические страницы
    for i in range(len(data)):
        data[i] = 1
    print("Memory allocated successfully. Sleeping...", flush=True)
    time.sleep(30)
except MemoryError:
    print("MemoryError encountered!", file=sys.stderr)
EOF

sudo chmod +x /usr/local/bin/mem-stress.py
```

#### Шаг 3.2: Создание сервиса с лимитом памяти
Создадим юнит `/etc/systemd/system/mem-stress.service` с лимитом `MemoryMax=64M`:
```bash
sudo tee /etc/systemd/system/mem-stress.service > /dev/null << 'EOF'
[Unit]
Description=Memory Stress Test with cgroups limit

[Service]
Type=simple
ExecStart=/usr/bin/python3 /usr/local/bin/mem-stress.py

# Ограничение cgroups v2
MemoryMax=64M
MemoryHigh=50M

# Запретить свопирование для гарантированного вызова OOM
MemorySwapMax=0

[Install]
WantedBy=multi-user.target
EOF
```

#### Шаг 3.3: Запуск и анализ реакции ядра
```bash
sudo systemctl daemon-reload
sudo systemctl start mem-stress.service

# Ожидаем завершения и проверяем статус
sudo systemctl status mem-stress.service --no-pager
```

Примерный вывод статуса:
```text
● mem-stress.service - Memory Stress Test with cgroups limit
     Loaded: loaded (/etc/systemd/system/mem-stress.service; static)
     Active: failed (Result: oom-kill) since ...
    Process: 12345 ExecStart=/usr/bin/python3 /usr/local/bin/mem-stress.py (code=killed, signal=KILL)
   Main PID: 12345 (code=killed, signal=KILL)
```

Детальный лог в `journalctl`:
```bash
journalctl -u mem-stress.service -n 20 --no-pager
```
*В журнале будет четко зафиксировано событие: `oom-kill` или `Killed process ... (mem-stress.py) total-vm:... anon-rss:...`.*
