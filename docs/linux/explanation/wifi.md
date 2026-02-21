---
title: "Беспроводная сеть Ethernet (Wi-Fi)"
type: explanation
tags: [linux, networking, wifi, wireless, 802.11, wpa, iw, wpa-supplicant, channels]
sources:
  docs: "https://wireless.wiki.kernel.org/en/users/documentation"
  book: "Внутреннее устройство Linux — Брайан Уорд, Глава 9.27"
related:
  - "[[linux/explanation/networking]]"
  - "[[linux/explanation/arp-ndp]]"
  - "[[linux/how-to/configure-network]]"
  - "[[linux/reference/cheatsheet]]"
---

# Беспроводная сеть Ethernet (Wi-Fi)

> **TL;DR:** Wi-Fi — это Ethernet по радио. На уровне IP и выше разницы нет: те же адреса, маршруты, DNS. Отличия — на канальном уровне (L2): общая среда передачи (радиоэфир), необходимость аутентификации (WPA2/WPA3), управление каналами и мощностью. В Linux Wi-Fi управляется подсистемой ядра `cfg80211`/`mac80211`, а в userspace — через `iw` (низкий уровень), `wpa_supplicant` (аутентификация) и NetworkManager (высокий уровень).

## Зачем это знать

- Wi-Fi используется на ноутбуках, IoT, серверах с out-of-band management — понимание протокола помогает при диагностике
- «Подключился, но интернета нет» — без понимания этапов (ассоциация → аутентификация → DHCP) невозможно локализовать проблему
- Безопасность: открытая сеть, WPA2-Personal, WPA2-Enterprise, WPA3 — разные модели угроз
- В контексте DevOps: настройка серверов с Wi-Fi-интерфейсом (edge-устройства, Raspberry Pi, точки доступа)

## Чем Wi-Fi отличается от проводного Ethernet

В принципе, беспроводные сети (Wi-Fi) незначительно отличаются от проводных с точки зрения верхних уровней стека. Разница — на физическом и канальном уровне:

| Аспект | Проводной Ethernet | Wi-Fi |
|--------|-------------------|-------|
| Среда передачи | Кабель (выделенная линия) | Радиоэфир (общая среда) |
| Коллизии | Полный дуплекс, коллизий нет | Полудуплекс, коллизии возможны |
| Механизм доступа | Не нужен (point-to-point) | CSMA/CA (Carrier Sense Multiple Access / Collision Avoidance) |
| Аутентификация | Физический доступ к порту | WPA2/WPA3 (обязательна для безопасности) |
| Скорость | Стабильная (1/10/25 Gbps) | Переменная, зависит от расстояния и помех |
| MTU | 1500 байт | 1500 байт (тот же) |
| Адресация (L2) | MAC-адреса | MAC-адреса (те же) |
| Адресация (L3+) | IP, TCP, UDP | IP, TCP, UDP (без изменений) |

### CSMA/CA — доступ к общей среде

Радиоэфир — общий ресурс. Два устройства не могут передавать одновременно на одном канале. Wi-Fi использует CSMA/CA:

1. **Слушать** эфир (Carrier Sense) — занят ли канал?
2. **Ждать** случайный интервал (backoff), если канал занят
3. **Передать** кадр
4. **Получить** ACK от получателя (в отличие от проводного Ethernet, где ACK на L2 не нужен)

> **Проблема скрытого узла:** устройства A и C оба в зоне действия точки доступа B, но не слышат друг друга. Оба считают канал свободным → коллизия на B. Решение — механизм RTS/CTS (Request to Send / Clear to Send).

## Стандарты 802.11

| Стандарт | Маркетинговое имя | Частоты | Макс. скорость | Год |
|----------|-------------------|---------|---------------|-----|
| 802.11b | — | 2.4 GHz | 11 Mbps | 1999 |
| 802.11a | — | 5 GHz | 54 Mbps | 1999 |
| 802.11g | — | 2.4 GHz | 54 Mbps | 2003 |
| 802.11n | Wi-Fi 4 | 2.4 / 5 GHz | 600 Mbps | 2009 |
| 802.11ac | Wi-Fi 5 | 5 GHz | 6.9 Gbps | 2013 |
| 802.11ax | Wi-Fi 6/6E | 2.4 / 5 / 6 GHz | 9.6 Gbps | 2020 |

### Частоты и каналы

```
2.4 GHz (каналы 1–13):
  + Лучше проникает через стены
  − Только 3 неперекрывающихся канала (1, 6, 11)
  − Перегружен: микроволновки, Bluetooth, соседи

5 GHz (каналы 36–165):
  + 25+ неперекрывающихся каналов
  + Меньше помех
  − Хуже проникает через стены, меньше радиус

6 GHz (Wi-Fi 6E):
  + Ещё больше каналов, минимум помех
  − Требует новое оборудование
```

```bash
# Посмотреть доступные частоты и каналы адаптера
iw phy phy0 channels

# Текущий канал интерфейса
iw dev wlan0 info
# ...
# channel 6 (2437 MHz), width: 20 MHz
```

## Безопасность Wi-Fi

### Эволюция протоколов

| Протокол | Шифрование | Статус |
|----------|-----------|--------|
| Open (нет) | Нет | Трафик виден всем в радиусе |
| WEP | RC4 (40/104 бит) | Взломан за минуты. Не использовать |
| WPA | TKIP (RC4 с улучшениями) | Устарел, уязвим |
| **WPA2** | **CCMP (AES-128)** | **Стандарт де-факто** |
| **WPA3** | **SAE + GCMP-256** | **Рекомендуется** для нового оборудования |

### WPA2-Personal vs Enterprise

| | WPA2-Personal (PSK) | WPA2-Enterprise (802.1X) |
|---|---------------------|--------------------------|
| Аутентификация | Общий пароль (Pre-Shared Key) | Индивидуальные учётные данные (RADIUS) |
| Применение | Дом, малый офис | Корпоративная сеть |
| Риск | Пароль знают все → компрометация одного = компрометация всех | Отзыв учётки одного сотрудника не затрагивает других |

### 4-way handshake (WPA2)

После ассоциации с точкой доступа клиент и AP выполняют 4-стороннее рукопожатие для генерации сессионных ключей:

```
1. AP → Client:  ANonce (случайное число AP)
2. Client → AP:  SNonce + MIC (клиент вычислил PTK из PSK + ANonce + SNonce)
3. AP → Client:  GTK (групповой ключ для broadcast/multicast) + MIC
4. Client → AP:  ACK

Результат: PTK (Pairwise Transient Key) — уникальный ключ сессии
           GTK (Group Temporal Key) — общий ключ для broadcast
```

> **Важно:** WPA2-Personal уязвим к офлайн-брутфорсу: перехватив handshake, атакующий может подбирать PSK без подключения к сети. Используй длинные пароли (12+ символов). WPA3 (SAE) устраняет эту уязвимость.

## Wi-Fi в Linux — архитектура

```
┌─────────────────────────────────────┐
│         NetworkManager / systemd-networkd   ← высокий уровень
│              (nmcli, nmtui)                  (автоподключение, профили)
├─────────────────────────────────────┤
│         wpa_supplicant / iwd                ← аутентификация
│         (WPA2/WPA3 handshake)                (userspace daemon)
├─────────────────────────────────────┤
│         iw / nl80211                        ← управление
│         (сканирование, каналы, мощность)     (netlink API)
├─────────────────────────────────────┤
│         cfg80211 / mac80211                 ← ядро
│         (общий Wi-Fi framework)              (управление кадрами)
├─────────────────────────────────────┤
│         Драйвер (iwlwifi, ath9k, etc.)      ← hardware
└─────────────────────────────────────┘
```

- **cfg80211** — ядерный API конфигурации Wi-Fi (замена старого wireless-extensions)
- **mac80211** — фреймворк для «софтовых» Wi-Fi-драйверов (большинство современных)
- **wpa_supplicant** — демон аутентификации (WPA/WPA2/WPA3). NetworkManager общается с ним
- **iwd** (Intel Wireless Daemon) — современная альтернатива wpa_supplicant (Arch по умолчанию)

## Инструменты

### iw — низкоуровневое управление

```bash
# Информация об адаптере
iw phy phy0 info
# Supported interface modes: managed, AP, monitor, ...
# Bands: 2.4 GHz, 5 GHz

# Информация о подключении
iw dev wlan0 info
# Interface wlan0
#   type managed
#   channel 36 (5180 MHz), width: 80 MHz
#   ssid MyNetwork
#   txpower 20.00 dBm

# Сканирование сетей
sudo iw dev wlan0 scan | grep -E "SSID|signal|freq"
# SSID: MyNetwork
# signal: -45.00 dBm    ← чем ближе к 0, тем лучше
# freq: 5180

# Статистика соединения
iw dev wlan0 station dump
# Station bb:bb:bb:bb:bb:bb
#   signal: -52 dBm
#   tx bitrate: 866.7 MBit/s
#   rx bitrate: 650.0 MBit/s
```

### wpa_supplicant — аутентификация

```bash
# Конфигурация WPA2-Personal
# /etc/wpa_supplicant/wpa_supplicant-wlan0.conf
ctrl_interface=/run/wpa_supplicant
update_config=1

network={
    ssid="MyNetwork"
    psk="my-secret-password"
    # или хешированный пароль (безопаснее):
    # psk=a1b2c3d4...  (вывод wpa_passphrase)
}

# Генерировать хеш пароля (не хранить plaintext)
wpa_passphrase "MyNetwork" "my-secret-password"

# Запустить вручную (обычно через NetworkManager)
sudo wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant-wlan0.conf
sudo dhclient wlan0    # получить IP
```

### NetworkManager (nmcli) — высокий уровень

Подробнее: → [[linux/how-to/configure-network]]

```bash
# Сканировать и подключиться
nmcli device wifi list
nmcli device wifi connect "MyNetwork" password "my-secret-password"

# Сохранённые профили
nmcli connection show
nmcli connection up "MyNetwork"
nmcli connection down "MyNetwork"

# Wi-Fi вкл/выкл
nmcli radio wifi on
nmcli radio wifi off
```

## Диагностика Wi-Fi

```bash
# 1. Адаптер виден ядром?
iw dev
# Нет вывода → драйвер не загружен. lspci | grep -i wireless, dmesg | grep -i wifi

# 2. Интерфейс поднят?
ip link show wlan0
# state DOWN → sudo ip link set wlan0 up

# 3. Сети видны?
nmcli device wifi list
# Пусто → sudo iw dev wlan0 scan (более низкоуровневый)

# 4. Подключён к AP?
iw dev wlan0 link
# Connected to aa:bb:cc:dd:ee:ff (on wlan0)
# SSID: MyNetwork
# signal: -55 dBm

# 5. IP получен?
ip addr show wlan0
# Нет inet → проблема DHCP

# 6. Уровень сигнала
iw dev wlan0 station dump | grep signal
# signal: -55 dBm   (хорошо: -30..-60, плохо: -70..-90)

# 7. Непрерывный мониторинг
watch -n 1 "iw dev wlan0 link"
```

## Подводные камни

| Проблема | Симптом | Решение |
|----------|---------|---------|
| Драйвер не загружен | `iw dev` пусто, `ip link` без wlan | `lspci`, `dmesg \| grep firmware`, установить firmware (`linux-firmware`) |
| Заблокирован rfkill | Адаптер виден, но не подключается | `rfkill list` → `rfkill unblock wifi` |
| Слабый сигнал | Частые дисконнекты, низкая скорость | `iw dev wlan0 station dump`, переключить на 2.4 GHz (лучше проникает) |
| Перегруженный канал | Подключён, но скорость очень низкая | `iw dev wlan0 scan \| grep -E "freq\|signal"`, сменить канал на AP |
| wpa_supplicant конфликтует с NetworkManager | Подключение не работает | Использовать что-то одно. NM сам управляет wpa_supplicant |
| Country code не установлен | Не видны 5 GHz каналы | `iw reg get`, `sudo iw reg set RU` (или через crda) |

## Связанные материалы

- [[linux/explanation/networking]] — стек TCP/IP, IP, подсети, маршрутизация (одинаковый для Wi-Fi и проводного)
- [[linux/explanation/arp-ndp]] — ARP/NDP работает одинаково поверх Wi-Fi и Ethernet
- [[linux/how-to/configure-network]] — nmcli, netplan, настройка подключений (включая Wi-Fi)
- [[linux/reference/cheatsheet]] — iw, nmcli, rfkill
