---
title: "Запуск Telegram бота на ВМ"
tags:
  - bot
  - vm
---
<style>body {text-align: justify}</style>

И-за того, что Youtube стал блочить запросы к его API из РФ, [youtube_transcript_bot](https://t.me/youtube_transcript_bot) сломался. Рабочим вариантом в процессе отладки оказалось банальное использование VPN. После некоторых попыток прикрутить vpn к контейнеру было решено перенести приложение на обычную виртуальную машину (ВМ).  

Далее, что было сделано.   

Поставим питон на ВМ
```bash
sudo dnf update
sudo dnf install -y python3.12
```

Так как в быту я пользуюсь adGuard VPN, и у них есть клиент для linux, то далее установим adGuard for linux по официальной инструкции.
```bash
sudo -s
curl -fsSL https://raw.githubusercontent.com/AdguardTeam/AdGuardVPNCLI/master/scripts/release/install.sh -o /root/adguard.sh
chmod +x /root/adguard.sh
/root/adguard.sh
```
Скрипт чуть-чуть пошуршит, после установки бинарник AdGuard VPN CLI будет доступен в /opt/adguardvpn_cli/adguardvpn-cli   

Бот будет запускаться под отдельным пользователем, создаем его, настраиваем виртуальное окружение и подгружаем необходимые библиотеки для работы.
```bash
sudo useradd -m -s /bin/bash youtube_transcript_bot
su youtube_transcript_bot
python3.12 -m venv ~/venv
venv/bin/pip install --no-cache-dir -f requirements.txt
```
Создаем в домашнем каталоге бота папку app и копируем в нее файлы приложения, не забывая выставить соответствующие права (chown) на файлы.

Создаем два системных сервиса. Первый для VPN, второй - запуск приложения.   
Сервис /etc/systemd/system/adguard-vpn.service. В /etc/adguard.conf хранится логин и пароль для авторизации. Вторая команда поднимает туннель до ближайшего сервера с минимальным latency.
```bash
[Unit]
Description=AdGuard VPN Connection
After=network.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'source /etc/adguard.conf \
    && /opt/adguardvpn_cli/adguardvpn-cli login -u "$VPN_USER" -p "$VPN_PASS" \
    && /opt/adguardvpn_cli/adguardvpn-cli connect -y'
ExecStop=/opt/adguardvpn_cli/adguardvpn-cli disconnect
RemainAfterExit=true

[Install]
WantedBy=multi-user.target
```
Сервис /etc/systemd/system/youtube-transcript-bot.service. Зависит и запускается после adguard-vpn.service. Если упал, поднимается всегда.
```bash
[Unit]
Description=Youtube Transcript Bot
After=adguard-vpn.service
Requires=adguard-vpn.service

[Service]
User=youtube_transcript_bot
WorkingDirectory=/home/youtube_transcript_bot/app
ExecStart=/bin/bash -c "source /home/youtube_transcript_bot/venv/bin/activate && exec python telegram_bot.py"
Restart=always

[Install]
WantedBy=multi-user.target
```

И стандартный набор для автозапуска
```bash
sudo systemctl daemon-reload
sudo systemctl enable adguard-vpn
sudo systemctl start adguard-vpn
```
Аналогичные команды с youtube-transcript-bot.service. Готово!