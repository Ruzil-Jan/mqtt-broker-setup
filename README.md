# Приложение А — Установка MQTT-брокера, настройка Flask-приложения и тестирование соединения

## Содержание

1. [Установка Mosquitto](#1-установка-mosquitto)
2. [Конфигурация брокера](#2-конфигурация-брокера)
3. [Настройка Flask-приложения](#3-настройка-flask-приложения)
4. [Тестирование MQTT-соединения](#4-тестирование-mqtt-соединения)
5. [Автозапуск системных служб](#5-автозапуск-системных-служб)
6. [Установка на Android (Termux)](#6-установка-на-android-termux)
7. [Типичные ошибки](#7-типичные-ошибки)

---

## 1. Установка Mosquitto

### Raspberry Pi OS / Debian

```bash
sudo apt update
sudo apt install -y mosquitto mosquitto-clients
```

### Arch Linux ARM

```bash
sudo pacman -S mosquitto
```

### Проверка установки

```bash
mosquitto -v
# ожидаемый вывод: mosquitto version 2.x.x
mosquitto_pub --version
mosquitto_sub --version
```

---

## 2. Конфигурация брокера

Начиная с версии 2.0, Mosquitto по умолчанию не принимает подключения с других адресов — необходимо явно настроить `listener` и `allow_anonymous`.

Создать файл конфигурации:

```bash
sudo nano /etc/mosquitto/conf.d/local.conf
```

Содержимое:

```
# Слушать порт 1883 на всех интерфейсах
listener 1883

# Разрешить подключения без пароля (локальная сеть)
allow_anonymous true

# Сохранение сообщений между перезапусками
persistence true
persistence_location /var/lib/mosquitto/

# Логирование
log_dest file /var/log/mosquitto/mosquitto.log
log_type all
connection_messages true
```

> **Примечание.** Если сервер доступен из внешней сети, рекомендуется настроить аутентификацию через `password_file` вместо `allow_anonymous true`.

Перезапустить и проверить:

```bash
sudo systemctl restart mosquitto
sudo systemctl status mosquitto
```

Ожидаемый вывод:

```
● mosquitto.service - Mosquitto MQTT Broker
     Active: active (running)
```

Проверить, что брокер слушает порт 1883:

```bash
ss -tlnp | grep 1883
# или
netstat -tlnp | grep 1883
```

---

## 3. Настройка Flask-приложения

### Переменные окружения

Приложение читает адрес брокера из переменных окружения. Создать файл `.env` в корне проекта:

```
MQTT_BROKER=localhost
MQTT_PORT=1883
```

Если брокер запущен на другом устройстве в сети:

```
MQTT_BROKER=192.168.1.100
MQTT_PORT=1883
```

В коде `app.py` переменные считываются следующим образом:

```python
MQTT_BROKER = os.getenv('MQTT_BROKER', 'localhost')
MQTT_PORT   = int(os.getenv('MQTT_PORT', 1883))
```

Значения по умолчанию (`localhost:1883`) работают без `.env`-файла, если брокер запущен на том же устройстве.

### Зависимости

```bash
# Создание виртуального окружения
python3 -m venv venv
source venv/bin/activate        # Linux / Android
# или: venv\Scripts\Activate.ps1  # Windows

# Установка зависимостей
pip install -r requirements.txt
```

Ключевые пакеты:

| Пакет | Версия | Назначение |
|---|---|---|
| `paho-mqtt` | 1.6.1 | MQTT-клиент Python |
| `Flask-SocketIO` | 5.3.4 | WebSocket поверх Flask |
| `python-socketio` | 5.9.0 | Движок SocketIO |
| `python-engineio` | 4.7.1 | Transport-слой SocketIO |
| `python-dotenv` | 1.0.0 | Загрузка `.env`-файла |

### Запуск приложения

```bash
python app.py
```

Ожидаемые строки в консоли при успешном запуске:

```
[CONFIG] Загружено 5 типов устройств
[MQTT] Конфигурация: localhost:1883
[APP] MQTT поток запущен
[MQTT] Подключено к брокеру
[APP] Запуск сервера на http://0.0.0.0:5000
```

---

## 4. Тестирование MQTT-соединения

### 4.1 Базовая проверка брокера

В первом терминале — подписаться на все топики:

```bash
mosquitto_sub -h localhost -t "#" -v
```

Флаг `-v` выводит название топика рядом с сообщением.

Во втором терминале — отправить тестовое сообщение:

```bash
mosquitto_pub -h localhost -t "test" -m "hello"
```

Ожидаемый вывод в первом терминале:

```
test hello
```

### 4.2 Имитация обновления состояния от IoT-устройства

Flask-приложение подписано на топик `home/devices/+/status`.  
При получении сообщения — обновляет БД и рассылает `device_update` через WebSocket всем браузерам.

```bash
# Устройство сообщает о своём состоянии
mosquitto_pub -h localhost -t "home/devices/1/status" \
  -m '{"state": "on", "temperature": 24.5}'

# Устройство сообщает об отключении
mosquitto_pub -h localhost -t "home/devices/1/status" \
  -m '{"state": "off"}'
```

Ожидаемый лог Flask:

```
[MQTT] Получено сообщение: home/devices/1/status — {"state": "on", "temperature": 24.5}
```

### 4.3 Перехват команды от Flask к устройству

При нажатии кнопки переключения Flask публикует в топик `home/devices/{id}/command`.

Подписаться и нажать кнопку в браузере:

```bash
mosquitto_sub -h localhost -t "home/devices/+/command" -v
```

Ожидаемый вывод после нажатия кнопки:

```
home/devices/1/command {"action": "off", "timestamp": "2024-01-01T12:00:00"}
```

### 4.4 Проверка QoS и доставки

Проверить гарантированную доставку (QoS=1 — подтверждение):

```bash
mosquitto_pub -h localhost -t "home/devices/1/status" \
  -m '{"state": "on"}' -q 1
```

Брокер должен подтвердить приём — при QoS 1 команда не потеряется при кратковременном разрыве соединения.

### 4.5 Сквозной тест через браузер

1. Открыть `http://<IP-сервера>:5000`
2. Подписаться на статусный топик:
   ```bash
   mosquitto_sub -h localhost -t "home/devices/+/status" -v
   ```
3. Отправить обновление состояния:
   ```bash
   mosquitto_pub -h localhost -t "home/devices/1/status" -m '{"state":"on"}'
   ```
4. Карточка устройства в браузере должна обновиться автоматически без перезагрузки страницы.

### 4.6 Проверка REST API

```bash
# Получить информацию об устройстве
curl http://localhost:5000/device/1

# Переключить состояние устройства
curl http://localhost:5000/toggle/1

# Добавить новое устройство
curl -X POST http://localhost:5000/add \
  -d "name=Гараж&ip=192.168.1.50&device_type=gate"

# Удалить устройство
curl -X POST http://localhost:5000/delete/1
```

---

## 5. Автозапуск системных служб

### Mosquitto (включить в автозагрузку)

```bash
sudo systemctl enable mosquitto
```

### Flask-приложение через systemd

Создать файл службы:

```bash
sudo nano /etc/systemd/system/iot-monitor.service
```

Содержимое:

```ini
[Unit]
Description=IoT Monitor Flask Application
After=network.target mosquitto.service
Requires=mosquitto.service

[Service]
User=pi
WorkingDirectory=/home/pi/local_project
EnvironmentFile=/home/pi/local_project/.env
ExecStart=/home/pi/local_project/venv/bin/python app.py
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

Активировать:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now iot-monitor
sudo systemctl status iot-monitor
```

Просмотр логов в реальном времени:

```bash
journalctl -u iot-monitor -f
```

### Порядок запуска

Флаг `Requires=mosquitto.service` гарантирует, что Flask-приложение не стартует раньше MQTT-брокера — это критично, так как `app.py` пытается подключиться к брокеру при запуске.

### Автозапуск через init.d (для систем без systemd)

```bash
sudo nano /etc/init.d/iot-monitor
```

```bash
#!/bin/sh
### BEGIN INIT INFO
# Provides:          iot-monitor
# Required-Start:    $network mosquitto
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
### END INIT INFO

APP_DIR=/home/pi/local_project
PYTHON=$APP_DIR/venv/bin/python

case "$1" in
  start)
    cd $APP_DIR
    $PYTHON app.py >> /var/log/iot-monitor.log 2>&1 &
    echo "IoT Monitor started"
    ;;
  stop)
    pkill -f "python app.py"
    echo "IoT Monitor stopped"
    ;;
  restart)
    $0 stop
    sleep 2
    $0 start
    ;;
esac
exit 0
```

```bash
sudo chmod +x /etc/init.d/iot-monitor
sudo update-rc.d iot-monitor defaults
```

---

## 6. Установка на Android (Termux)

### Установка Termux

Скачать Termux из [F-Droid](https://f-droid.org/packages/com.termux/) (не из Google Play — версия устарела).

### Установка зависимостей

```bash
pkg update && pkg upgrade -y
pkg install -y python mosquitto
```

### Конфигурация Mosquitto в Termux

```bash
mkdir -p $PREFIX/etc/mosquitto/conf.d
nano $PREFIX/etc/mosquitto/conf.d/local.conf
```

Содержимое:

```
listener 1883
allow_anonymous true
persistence false
```

### Запуск сервисов

```bash
# Запустить MQTT-брокер в фоне
mosquitto -c $PREFIX/etc/mosquitto/conf.d/local.conf -d

# Установить зависимости проекта
cd ~/local_project
pip install -r requirements.txt

# Запустить Flask
python app.py
```

### Автозапуск при загрузке Android (Termux:Boot)

Установить [Termux:Boot](https://f-droid.org/packages/com.termux.boot/) из F-Droid.

Создать скрипт автозапуска:

```bash
mkdir -p ~/.termux/boot
nano ~/.termux/boot/start-iot.sh
```

Содержимое:

```bash
#!/data/data/com.termux/files/usr/bin/bash
termux-wake-lock
mosquitto -c $PREFIX/etc/mosquitto/conf.d/local.conf -d
sleep 2
cd ~/local_project
python app.py >> ~/iot-monitor.log 2>&1 &
```

```bash
chmod +x ~/.termux/boot/start-iot.sh
```

---

## 7. Типичные ошибки

| Ошибка | Причина | Решение |
|---|---|---|
| `[Errno 111] Connection refused` на порту 1883 | Mosquitto не запущен | `sudo systemctl start mosquitto` |
| `Client <id> not allowed` (Mosquitto 2.x) | Не настроено `allow_anonymous` | Добавить в конфиг `allow_anonymous true` |
| Flask выводит `[MQTT] Ошибка подключения` | Неверный `MQTT_BROKER` в `.env` | Проверить IP брокера: `mosquitto_sub -h <IP> -t test` |
| Состояние в браузере не обновляется автоматически | Flask не запущен через `socketio.run()` | В `app.py` должно быть `socketio.run(app, ...)`, а не `app.run(...)` |
| `database is locked` | Конкурентный доступ к SQLite | WAL-режим уже включён в `db.py`; проверить, нет ли зависших процессов Python |
| MQTT-сообщение не обновляет БД | Неверный формат топика | Топик должен быть `home/devices/{id}/status`, где `{id}` — числовой ID из таблицы `bulbs` |
| Порт 5000 недоступен с других устройств | Flask слушает только `127.0.0.1` | Убедиться, что запуск: `socketio.run(app, host="0.0.0.0", port=5000)` |
