Схема: [Твой ПК] ➔ [RU-Сервер (xxx.xxx.xxx.xxx)] ➔ [AWG-Туннель] ➔ [VPS (yyy.yyy.yyy.yyy)] ➔ [Internet]

---

1. Подготовка (Оба сервера)

🇷🇺 🌍 Выполнить на **RU** и **VPS**:

```
sudo apt update && sudo apt upgrade -y
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

🇷🇺 **На RU-сервере (Приемник):**

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

🌍 **На VPS-сервере (Инициатор):**

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

**Запуск (на обоих):** `sudo awg-quick up awg0`. Проверка связи с VPS: `ping 10.8.0.1`.

Посмотреть в терминале config AmneziaWG:
  
```
sudo awg show
```
---

3. 🇷🇺 Установка Gost (Только RU)

Скачать архив:
```
   https://github.com/go-gost/gost/releases/download/v3.2.7-nightly.20251122/gost_3.2.7-nightly.20251122_linux_amd64.tar.gz
```

Используя любой FTP-клиент, подключиться к серверу:

https://filezilla-project.org/download.php?platform=win32
https://filezilla-project.org/download.php?platform=osx
https://filezilla-project.org/download.php?platform=macos-arm64

https://filezilla-project.org/download.php?type=client

Положить архив в корень и выполнить:

```
tar -xvf gost_3.2.7-nightly.20251122_linux_amd64.tar
sudo mv gost /usr/bin/gost && sudo chmod +x /usr/bin/gost
```

Создать сервис `sudo nano /etc/systemd/system/gost.service`:

```
[Service]
ExecStart=/usr/bin/gost -L tcp://:443 -F forward://10.8.0.2:443
Restart=always
AmbientCapabilities=CAP_NET_BIND_SERVICE
[Install]
WantedBy=multi-user.target
```

```
sudo systemctl enable --now gost
```

  

Если что-то пошло не так:

Удалит интерфейс, созданный вручную

Это освободит имя `awg0` для системной службы:

```
sudo ip link delete dev awg0
```

2. Запустите службу заново

Теперь, когда имя свободно, служба должна успешно стартовать:

```
sudo systemctl start awg-quick@awg0
```

3. Проверьте статус

```
sudo systemctl status awg-quick@awg0
```



4. 🌍 Установка 3x-ui (Только VPS)

```
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
```

1. Зайти: `http://yyy.yyy.yyy.yyy:port`.
2. Создать **Inbound**: VLESS, Порт **443**, Reality **ON**.
3. **Listen IP** оставить ПУСТЫМ.
4. **SNI/Dest**: `asus.com` или `://google.com`.

5. 🇷🇺 🌍 Оптимизация Скорости (Оба сервера)

Чтобы YouTube не тормозил (BBR):

```
echo "net.core.default_qdisc=fq" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

6. Настройка клиента (Nekobox/v2rayN):
  
7. Импортируй ссылку VLESS из панели VPS.
  
8. Смени **Address** на 🇷🇺`xxx.xxx.xxx.xxx` (RU).
  
9. **Port**: 443.
  
10. **Fingerprint**: `chrome`.
  
11. **Flow**: `xtls-rprx-vision`.
