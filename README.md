# Установка и настройка реверсивного туннеля с обфускацией

> [!IMPORTANT]
> Этот проект предназначен только для **личного использования**.  
> Данный материал создан *исключительно в образовательных целях*.  
> Автор не несёт ответственности за возможные последствия использования.  
> Применяйте только в рамках законного тестирования и легальных задач.

---

Схема: [Твой ПК] ➔ [RU-Сервер (xxx.xxx.xxx.xxx)] ➔ [AWG-Туннель] ➔ [VPS (yyy.yyy.yyy.yyy)] ➔ [Internet]


1. Подготовка (Оба сервера)

## ⚙️ Выполнить на **RU** и **VPS**:

```
sudo apt update && sudo apt upgrade -y
sudo apt install linux-headers-generic
# Важно: заголовки ядра нужны для корректной сборки модуля AWG
sudo apt install curl wget nano software-properties-common linux-headers-$(uname -r) -y
sudo add-apt-repository ppa:amnezia/ppa -y
sudo apt update && sudo apt install amneziawg -y

# Генерация ключей (сделай на каждом и запиши результат)
awg genkey | tee privatekey | awg pubkey > publickey
```

Посмотреть приватный ключ:

```
cat privatekey
```

Посмотреть публичный ключ:

```
cat publickey
```

---

2. Настройка AmneziaWG (Туннель)

Создать файл `sudo nano /etc/amnezia/amneziawg/awg0.conf`:

## **На RU-сервере (Приемник):**

```
[Interface]
PrivateKey = <ТВОЙ_PRIV_RU>
Address = 10.8.0.1/24
ListenPort = 51820
MTU = 1100
Jc = 4
Jmin = 40
Jmax = 70
S1 = 15
S2 = 24
H1 = 1
H2 = 2
H3 = 3
H4 = 4

[Peer]
PublicKey = <ПУБЛИЧНЫЙ_KEY_ОТ_VPS>
AllowedIPs = 10.8.0.2/32
```

## **На VPS-сервере (Инициатор):**

```
[Interface]
PrivateKey = <ТВОЙ_PRIV_KEY_VPS>
Address = 10.8.0.2/24
ListenPort = 51820
MTU = 1100
Jc = 4
Jmin = 40
Jmax = 70
S1 = 15
S2 = 24
H1 = 1
H2 = 2
H3 = 3
H4 = 4

[Peer]
PublicKey = <ПУБЛИЧНЫЙ_KEY_ОТ_RU>
Endpoint = xxx.xxx.xxx.xxx:51820
AllowedIPs = 10.8.0.0/24
PersistentKeepalive = 20
```

**Запуск (на обоих):**.
```
sudo awg-quick up awg0
```
Проверка связи с VPS:

```
ping 10.8.0.1
```
---

## 3. Установка Gost (Только RU)

Скачать архив:
```
   https://github.com/go-gost/gost/releases/download/v3.2.7-nightly.20251122/gost_3.2.7-nightly.20251122_linux_amd64.tar.gz
```

Используя любой FTP-клиент, подключиться к серверу:




https://filezilla-project.org/download.php?platform=win32 <img width="48" height="48" alt="icons8-windows-10-48" src="https://github.com/user-attachments/assets/1dead14d-1fec-41f3-b653-a74020ab12e8" /> 


https://filezilla-project.org/download.php?platform=osx <img width="50" height="50" alt="icons8-mac-os-50" src="https://github.com/user-attachments/assets/e70c4553-5d64-4431-8048-ce7c89b89484" />


(https://filezilla-project.org/download.php?platform=macos-arm64 <img width="48" height="48" alt="icons8-mac-os-48" src="https://github.com/user-attachments/assets/edb97cde-4b6f-42fa-a574-c233e38da9c9" />


Положить архив в корень и выполнить:

```
tar -xvf gost_3.2.7-nightly.20251122_linux_amd64.tar.gz
sudo mv gost /usr/bin/gost && sudo chmod +x /usr/bin/gost
```

Создать сервис
```
sudo nano /etc/systemd/system/gost.service
```

```
[Service]
ExecStart=/usr/bin/gost \
  -L tcp://:443 \
  -L udp://:443 \
  -F forward://10.8.0.2:443
Restart=always
AmbientCapabilities=CAP_NET_BIND_SERVICE
[Install]
WantedBy=multi-user.target
```

Чтобы UDP-пакеты «летали» без задержек, добавь правило для MSS (размера пакетов) специально для UDP-туннелирования. Выполни на VPS:

```
sudo iptables -t mangle -A FORWARD -p udp -j TEE --gateway 10.8.0.1
```

> [!NOTE]
> Если есть второй VPS и хочешь чтобы работал в режиме балансировщика:
>```
>[Service]
>ExecStart=/usr/bin/gost \
>  -L tcp://:443 \
>  -L udp://:443 \
>  -F forward://10.8.0.2:443?priority=10 \
>  -F forward://10.8.0.3:443?priority=5 \
>  --strategy=failover
>Restart=always
>AmbientCapabilities=CAP_NET_BIND_SERVICE
>[Install]
>WantedBy=multi-user.target
>```


Выолнить:
```
sudo systemctl enable --now gost
```

## 4. Установка 3x-ui (Только VPS)

```
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
```

1. Зайти: `http://yyy.yyy.yyy.yyy:port`.
2. Создать **Inbound**: VLESS, Порт **443**, Reality **ON**.
3. **Listen IP** оставить ПУСТЫМ.
4. **SNI/Dest**: `asus.com` или `google.com`.

## 5. Оптимизация Скорости (Оба сервера)

```
nano /etc/sysctl.conf
```

```
#Тюнинг скорости (BBR + FQ + FastOpen)

net.core.default_qdisc = fq

net.ipv4.tcp_congestion_control = bbr

net.ipv4.tcp_fastopen = 3

#Оптимизация буферов

net.core.rmem_max = 67108864

net.core.wmem_max = 67108864

net.ipv4.tcp_rmem = 4096 87380 67108864

net.ipv4.tcp_wmem = 4096 65536 67108864

#Лимиты очередей

net.core.somaxconn = 4096

#Увеличение очереди (помогает при нагрузках)

net.core.netdev_max_backlog = 5000

net.ipv4.tcp_max_syn_backlog = 4096

#Безопасность

net.ipv4.conf.all.rp_filter = 1

net.ipv4.conf.default.rp_filter = 1

net.ipv4.icmp_echo_ignore_broadcasts = 1

net.ipv4.tcp_syncookies = 1

#DPI-friendly

net.ipv4.tcp_mtu_probing = 1

#Стабильность соединений

net.ipv4.tcp_keepalive_time = 600

net.ipv4.tcp_keepalive_intvl = 60

net.ipv4.tcp_keepalive_probes = 3

#TIME_WAIT и FIN

net.ipv4.tcp_fin_timeout = 30

#Отключение ipv6

net.ipv6.conf.all.disable_ipv6 = 1

net.ipv6.conf.default.disable_ipv6 = 1

net.ipv6.conf.lo.disable_ipv6 = 1

#Защита от определения аптайма и ОС

net.ipv4.tcp_timestamps = 0

#Ограничение ответов на ICMP (чтобы сложнее было сканировать)

net.ipv4.icmp_echo_ignore_all = 1
```

Применяем изменения

```
sudo sysctl -p
```

Настройка MTU (самое важное для скрытия туннелей)
Нестандартный MTU (например, 1420 или 1280) — это главный признак VPN.
Хотя мы ставили 1180 для стабильности, сканеры видят это. Чтобы это скрыть, можно использовать правило MSS Clamping в iptables на RU-сервере:
```
sudo iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```

Ограничение размера логов в Systemd
По умолчанию Linux может хранить гигабайты логов. Мы это исправим.
Открой конфиг journald:

```
nano /etc/systemd/journald.conf
```
Найди или добавь в секцию [Journal] следующие строки:

```
#Максимальный размер логов на диске

SystemMaxUse=100M

#Хранить логи не дольше 1 дня

MaxRetentionSec=1day
```

Перезапусти службу логов:

```
sudo systemctl restart systemd-journald
```

6. Импортируй ссылку VLESS из панели VPS.
  
7. Смени **Address** на 🇷🇺`xxx.xxx.xxx.xxx` (RU).


> [!NOTE]
>
> **Если что-то пошло не так:**  
> Удалит интерфейс, созданный вручную
>```
>sudo ip link delete dev awg0
>```
> Запустит службу заново
>```
>sudo systemctl start awg-quick@awg0
>```
>
> Проверить статус службы
>```
>sudo systemctl status awg-quick@awg0
>```
> Посмотреть в терминале config AmneziaWG:
>```
>sudo awg show
>```
