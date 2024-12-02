---
title: "Авторотация ssl-сертификата"
tags:
  - nginx
  - ssl
---
<style>body {text-align: justify}</style>

Короткая заметка, скорее даже для себя, т.к. постов на эту тему написано немало.  

Let's encrypt позволяет выпускать сертификат со сроком жизни не более 3 месяцев.
Далее требуется его повторный перевыпуск. Умные и хитрые люди давно автоматизировали этот процесс в виде проекта [Certbot](https://certbot.eff.org/). В результате установка и настройка происходит буквально в три команды. На моей Fedora это:
```bash
sudo dnf update #Выполнять с осторожностью
sudo dnf install certbot python3-certbot-nginx -y
```

Проверка правил firewall, должны быть открыты и 80, и 443 порт сервера. Как-то сузить список адресов, с которых осуществляется проверка, по-простому не удастся, сервер должен быть доступен по обоим портам. 
```bash
sudo firewall-cmd --list-all
```

Единственная команда, которая выполняет все необходимые действия: 
- Определяет где-что стоит, пути до сертификатов. 
- Подготовливает запрос для проверки, размещает необходимые чанки с проверочными секретами 
- Выпускает сертификаты, прописывает их путь в nginx.conf
- Добавляет сервис systemd и ставит в cron расписание обновления!

```bash
sudo certbot --nginx
```

Результат работы будет на экране, если-что-то не так, путь до логов. Проверить работу без перевыпуска сертификатов можно ключом --dry-run.

```bash
sudo certbot renew --dry-run
```

И заодно убедиться в наличии cron:
```bash
systemctl list-timers
systemctl status certbot-renew.timer
```

Осталось обновить страницу браузера и увидеть зеленый замочек в адресной строке!