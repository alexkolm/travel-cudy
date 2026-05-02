# travel-cudy

🇬🇧 English version: [README.md](README.md)

Набор скриптов и конфигураций для OpenWrt-роутера, реализующий переключение трафика между прямым подключением и туннелем через WireGuard + VK TURN (через Turnable).

---

## 🚀 Возможности

- Переключение режимов:
  - **tunnel** — весь LAN-трафик через WireGuard + TURN relay
  - **direct** — прямой выход через WAN (mwan3)
- Управление через физическую кнопку (hotplug)
- Поддержка multi-WAN через mwan3 (wan / wan24 / wan5)
- Автоматическое восстановление маршрутов и состояния
- Защита от утечек трафика при переключении
- Очистка conntrack при смене режима
- Индикация состояния через LED:
  - белый = tunnel
  - красный = direct

---

## 🧠 Архитектура

LAN (192.168.99.0/24)
        │
        ▼
 policy routing (ip rule 700)
        │
        ▼
     wg0 (WireGuard)
        │
        ▼
127.0.0.1:51820 (Turnable)
        │
        ▼
   VK TURN relay
        │
        ▼
        VPS
        │
        ▼
     Интернет

---

## 📁 Структура проекта

bin/
  tunnel-on
  tunnel-off
  vk-turn-wrapper

init/
  vk-turn
  tunnel-switch-boot

hotplug/
  button/
    10-tunnel-switch
  iface/
    99-wg-routing

etc-config/
  network
  dhcp
  firewall
  mwan3

---

## ⚙️ Как это работает

### ▶️ tunnel-on

- блокирует mwan3 для LAN (через iptables PREROUTING)
- очищает conntrack
- запускает turnable-client
- поднимает wg0
- настраивает маршрутизацию:
  - ip rule 700 → table wg
  - ip rule 750 → unreachable
- сбрасывает route cache
- включает белый LED

---

### ⏹ tunnel-off

- ставит временное правило unreachable (защита от утечек)
- удаляет правила 700/750
- опускает wg0
- останавливает turnable-client
- возвращает управление mwan3
- очищает conntrack
- сбрасывает route cache
- включает красный LED

---

### 🔘 hotplug

- кнопка управляет режимом:
  - released → tunnel
  - pressed → direct
- состояние сохраняется:
  /etc/tunnel-switch.position
- при загрузке применяется последний режим

---

## 📡 Wi-Fi fallback watchdog

Реализован watchdog (`wifi-fallback`), обеспечивающий стабильную работу Wi-Fi в travel-router сценарии.

---

### 🎯 Назначение

- автоматическое подключение к Wi-Fi uplink (STA)
- сохранение доступности точек доступа (AP)
- устойчивость при пропадании uplink

---

### 🧠 Логика работы

Для каждого диапазона (2.4 GHz / 5 GHz):

**AP (Access Point):**
- всегда включён
- работает независимо от uplink
- автоматически восстанавливается при сбоях

**STA (Client / uplink):**
- подключается к заданному SSID
- отключается при отсутствии сигнала
- периодически пытается восстановить соединение

---

### 🔄 Поведение

- uplink доступен → STA подключается, AP работает
- uplink пропал → STA отключается, AP остаётся
- uplink появился → STA подключается автоматически
- нет Wi-Fi uplink → используется WAN (mwan3)

---

### 🔍 Обнаружение сети

- используется passive scan через AP-интерфейс:
  iw dev phyX-ap0 scan
- не влияет на клиентов
- не требует перезапуска Wi-Fi

---

### 🛡 Стабильность

- минимизировано использование wifi reload
- STA не "флапает"
- AP не перезапускается без необходимости
- используется state machine:

init → trying → connected → off

---

### ⚙️ Конфигурация

CHECK_INTERVAL=10
PROBE_INTERVAL=60
ASSOC_TIMEOUT=60

---

### 📂 Файлы

/usr/bin/wifi-fallback-watchdog
/etc/init.d/wifi-fallback

---

### 🧪 Диагностика

logread -e wifi-fallback
iw dev
iw dev phy0-sta0 link
iw dev phy1-sta0 link
ifstatus wan24
ifstatus wan5

---

### ⚠️ Особенности

- wifi reload используется только для восстановления AP
- passive scan может не работать на некоторых драйверах
- поведение зависит от конфигурации wireless и mwan3

---

## ⚠️ Взаимодействие с mwan3

В режиме tunnel:

- mwan3 НЕ участвует в маршрутизации LAN-трафика
- достигается через:
  iptables -t mangle -I PREROUTING -i br-lan -j RETURN

В режиме direct:

- mwan3 полностью управляет трафиком
- работает failover между WAN-интерфейсами

---

## 🔐 Безопасность

- используется временное правило:
  priority 699 → unreachable
- conntrack очищается при смене режима

---

## 📦 Требования

- OpenWrt (procd, hotplug, UCI)
- WireGuard
- mwan3 (рекомендуется)
- настроенный интерфейс wg0
- установленный turnable-client

---

## 📥 Установка

Скопировать файлы:

/usr/bin/
/etc/init.d/
/etc/hotplug.d/
/etc/config/

Выдать права:

chmod +x /usr/bin/*
chmod +x /etc/init.d/vk-turn

---

## 🧪 Диагностика

wg show
ip rule
ip route show table wg
iptables -t mangle -L -n
logread -e tunnel-switch
ps | grep turnable

Проверка маршрута:

ip route get 8.8.8.8 from 192.168.99.2 iif br-lan

---

## 🐛 Известные особенности

- скорость ограничена TURN relay (может быть 0.5–2 Mbps)
- чувствительно к MTU (обычно 1280–1420)
- зависит от доступности VK TURN серверов
- при переключении возможны кратковременные потери пакетов

---

## ❗ Важно

- Секреты (ключи, токены) удалены
- Бинарник turnable-client не включён
- Требуется настройка WG на стороне VPS

---

## 📌 Примечание

Проект ориентирован на travel-router сценарии:

- обход ограничений
- работа через relay
- использование нескольких источников интернета (WAN/WiFi/LTE)

Может требовать адаптации под конкретную конфигурацию.

