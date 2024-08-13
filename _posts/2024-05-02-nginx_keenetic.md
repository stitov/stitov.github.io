---
title: "Настройка nginx на роутере keenetic"
tags:
  - nginx
  - keenetic
---
<style>body {text-align: justify}</style>

Статья описывает установку веб-сервера nginx на роутерах keenetic. Для чего это может быть нужно?
Если собственный блог страсть как хочется, публичным хостерам типа github pages вы не доверяете, своей "железной" виртуалки дома у вас нет, а облачному провайдеру платить не хочется, 
то такой вариант вполне возможен.   

Помимо установки затрагиваются вопросы настройки nginx для ограниченного доступа к приватным разделам сайта.

## Настройка keenetic
Вопрос использовать http или https уже давно не стоит, а значит для полноценного сайта вам потребуется освободить 443 порт, который по умолчанию занят админкой keenetic. 
Необходимо включить в UI админке доступ посредством telnet, и с помощью любого shell клиента попасть на роутер:
```bash
telnet 192.168.1.1 # IP адрес роутера
Login:
Password:
# Переназначение порта, пусть будет 4433, 80 порт также остается
ip http ssl port 4433
system configuration save
```

## Установка nginx

Там же, в админке установите дополнительные компоненты системы - OPKG, еще я добавил "Модули ядра для поддержки файловых систем". Описанное верно для версии ядра KeeneticOS 4.1.6.
Подробная инструкция по [Установке OPKG Entware на встроенную память роутера](https://help.keenetic.ru/hc/ru/articles/360021888880.html?utm_source=webhelp&utm_campaign=4.00.C.7.0-0&utm_medium=ui_notes&utm_content=controlpanel/opkg) лежит в официальной документации на сайте Keenetic.

Перейдем к установке пакетов. Также, в консоли, открыв ssh до только что установленного линукса на роутере, обновим репо и поставим необходимое.
```bash
opkg update
opkg install nginx-ssl
opkg install bash ca-certificates coreutils-mktemp curl diffutils grep openssl-util openssh-sftp-server
```

Готово! Осталось скопировать ваш nginx.conf в /opt/etc/nginx/nginx.conf. На сайте DigitalOcean есть удобный мастер по подготовке файла конфигурации с вашими требованиями, 
ссылки в конце статьи. 

Проверим конфигурацию и запустим сервис.

```bash
# Проверка конфига
nginx -t
# Старт сервиса
/opt/etc/init.d/S80nginx start
# Еще пригодится релоад и рестарт
/opt/etc/init.d/S80nginx reload
/opt/etc/init.d/S80nginx restart
```

## Базовая аутентификация

Базовая аутентифкация входит в стандартную поставку nginx и позволяет закрыть определенные раздела сайта парой логин/пароль. 
Для чего - решать вам. 

Создадим файл с логином и паролем.
```bash
mkdir -p /opt/root/ca/private/
cd /opt/root/ca/private/
touch htpasswd
# Безопасность превыше всего!
chmod 400 htpasswd
chown -R nobody:root /opt/root/ca/private/
# Собственно сама пара, вместо my_login подставить свой логин
echo 'my_login:' >> /opt/root/ca/private/htpasswd
# Укажем ключ -5 чтобы использовать хэш-функцию sha-256
openssl passwd -5 >> /opt/root/ca/private/htpasswd
```

Теперь можно добавить в ваш nginx.conf в интересующий раздел location строки:
```vim
auth_basic           "Private zone";
auth_basic_user_file /opt/root/ca/private/htpasswd;
```
Например, это будет location /private/, тогда при переходе на www.your.domain.com/private вы получите предложение ввести логин и пароль для открытия страницы.


## Клиентские сертификаты

Для абсолютного большинства публичных сайтов используется ssl-сертификат сервера, который позволяет через цепочку доверия
убедиться в подлинности сайта. Да, это не идеально, и зеленый замочек в вашем браузере говорит лишь о том, что сайт принадлежит его владельцу, 
а что за "Рога И Копыта" им владеют, честная это компания или пирамида-однодневка, система, да и вся отрасль сертификатов вам ответ не дадут.

В корпоративной среде, помимо проверки валидности серверного сертификата клиентом, распространена обратная аутентификация клиента сервером, или, проще говоря,
вход на веб-ресурс по ssl-сертификату. Для построения такой схемы используются собственные центры сертификации по аналогии с self-signed сертификатами без привлечения
третьях лиц, ~~которые забили место под солнцем,~~ или доверенных центров сертификации.

Выдачей клиентских сертификатов мы сейчас и займемся. 

```bash
 # Подготовим структуру каталогов
 cd /opt/root/
 mkdir ca
 cd ca/
 mkdir certs csr newcerts private
 chmod 700 private
 # Каждый клиентский сертификат будет иметь свой номер, здесь определим номерную серию
 touch index.txt
 echo 1000 > serial
```
**CA profile.**  
Подготовим конфигурационный файл conf/ca.conf.
```vim
[ ca ]
default_ca = CA_default

[ CA_default ]
# Directory and file locations.
dir               = /opt/root/ca
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/private/.rand

# The root key and root certificate.  
private_key       = $dir/private/ca.key  
certificate       = $dir/certs/ca.crt

name_opt          = ca_default  
cert_opt          = ca_default  
default_days      = 365  
default_md        = sha256
preserve          = no  
policy            = policy

[ policy ]
# The root CA should only sign intermediate certificates that match.
# See the POLICY FORMAT section of man ca.
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional


[ req ]
default_md = sha256
prompt = no
string_mask = utf8only
x509_extensions = v3_ca
distinguished_name = req_distinguished_name

[ req_distinguished_name ]
commonName = SetiGateCA
countryName = RU
stateOrProvinceName = Moscow
organizationName = SetiGate

[ v3_ca ]
basicConstraints = critical,CA:true
keyUsage=critical,digitalSignature
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer

[ server_cert ]
basicConstraints=CA:false
extendedKeyUsage=serverAuth
nsCertType = server
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always

[ user_cert ]
basicConstraints=CA:false
extendedKeyUsage=clientAuth
nsCertType = client
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
```
Генерация ключей и сертификатов. Создаем приватный ключ, csr и выпускам сертификат от собственного Certificate Authority (CA).
```bash
# Конечно, на эллиптических кривых
openssl ecparam -genkey -name secp384r1 > private/ca.key
chmod 400 ca.key
# Срок действия сертификата 10 лет
openssl req -config conf/ca.conf -key private/ca.key -new -x509 -days 3650 -out certs/ca.crt
openssl x509 -in certs/ca.crt -text
```
**Server Profile.**   
В зависимости от ваших требований этот пункт можно пропустить. На серверный сертификат, полученный таким образом, браузер будет ругаться как на самоподписанный.
Чтобы сторонние пользователи сайта не видели угрожающих надписей, то можно выпустить серверный сертификат в Let's Encrypt. 

Подготовим конфигурационный файл conf/server.conf.
```vim
[ req ]
default_md = sha256
prompt = no
days = 365
string_mask = utf8only
req_extensions = req_ext
distinguished_name = req_distinguished_name

[ req_distinguished_name ]
commonName = setigate.ru
countryName = RU
stateOrProvinceName = Moscow
organizationName = SetiGate

[ req_ext ]
#extendedKeyUsage=serverAuth
#basicConstraints=CA:false
#nsCertType = server
#subjectKeyIdentifier = hash
##authorityKeyIdentifier = keyid,issuer:always
subjectAltName = @alt_names

[ alt_names ]
DNS.0 = setigate.ru
DNS.1 = www.setigate.ru
```
Генерация ключей и сертификатов. Создаем приватный ключ, csr и выпускам сертификат от собственного CA.
```bash
# Приватный ключ
openssl ecparam -genkey -name secp384r1 > private/server.key
chmod 400 server.key
# Запрос на выпуск сертификата (CSR)
openssl req -config conf/server.conf -key private/server.key -new -out csr/server.csr
# Подписываем CSR нашим CA
openssl ca -config conf/ca.conf -extensions server_cert -in csr/server.csr -out certs/server.crt
```

**Client profile.**  
Выпуск  клиентского сертификата, по которому будет осуществляться вход на защищенный ресурс.
Подготовим конфигурационный файл для выпуска сертификата conf/client.conf.
```vim
[ req ]
default_md = sha256
prompt = no
days = 365
string_mask = utf8only
req_extensions = req_ext
distinguished_name = req_distinguished_name

[ req_distinguished_name ]
commonName = Sergei TItov
countryName = RU
stateOrProvinceName = Moscow
organizationName = SetiGate

[ req_ext ]
#extendedKeyUsage=clientAuth
#basicConstraints=CA:false
#nsCertType = client
#subjectKeyIdentifier = hash
#authorityKeyIdentifier = keyid,issuer
subjectAltName = @alt_names

[ alt_names ]
DNS.0 = setigate.ru
DNS.1 = www.setigate.ru
```
Генерация ключей и сертификатов. Создаем приватный ключ, csr и выпускам сертификат от собственного CA.
```bash
# Приватный ключ
openssl ecparam -genkey -name secp384r1 > /opt/root/ca/private/client.key
chmod 400 client.key
# Запрос на выпуск сертификата (CSR)
openssl req -config conf/client.conf  -key private/client.key  -new -out csr/client.csr
# Подписываем CSR нашим CA
openssl ca -config conf/ca.conf -extensions user_cert -in csr/client.csr -out certs/client.crt
# Преобразуем сертификат и приватный ключ в формат контейнера pkcs12
openssl pkcs12 -export -in certs/client.crt -inkey private/client.key -out client.p12 -passout pass:123
```
> Примечание.  
> Изначально пробовал использовать известную кривую ed25519 для создания ключа, но оказалось, что она не сильно приветствуется, и большинство браузеров не поддерживает этот тип кривой. Алгоритм ed25519 создан энтузиастом, и, видимо, явных дыр в нем нет, в отличие от того, что одобрил [NIST](https://www.nist.gov/). Но это не должо волновать простого владельца личного бложика. Если вам тоже захочеться поэкспериментировать, то вызов openssl немного другой:  
`openssl genpkey -algorithm ed25519 -out test.key`


Полученный сертификат в формате pkcs12 необходимо скачать на компьютер и загрузить в хранилище сертификатов браузера. Теперь можно проверить подключение.
При открытии защищенной ссылки браузер предложит выбрать для аутентификации загруженный клиентский сертификат.
Также можно проверить подключение через curl.

```bash
openssl s_client -connect your.domain.com
curl -vk --key private/client.key --cert certs/client.crt https://your.domain.com/private/
```

## Полезные ссылки

Чтобы разобраться в области ssl-сертификатов, понять процесс их выдачи, тонкой конфигурации и, особенно, как создать свой собственный удостоверяющий центр, мне помогли следующие ссылки:

- [Онлайн мастер NginxСonfig от DigitalOcean](https://www.digitalocean.com/community/tools/nginx?global.app.lang=ru)
- [Create ED25519 certificates for TLS with OpenSSL](https://blog.pinterjann.is/ed25519-certificates.html)
- [Статья на Хабре. Авторизация клиентов в nginx посредством SSL сертификатов](https://habr.com/ru/articles/213741/)
- [На английском. How to setup your own CA with OpenSSL](https://gist.github.com/Soarez/9688998)
- [Сформировать CSR на CertificateTools.com X509 Certificate Generator](https://certificatetools.com/)
- [Официальная документация OpenSSL x509v3_config по формату сертификатов](https://www.openssl.org/docs/man3.0/man5/x509v3_config.html)
- [Инструкция от IBM OpenSSL configuration examples](https://www.ibm.com/docs/en/hpvs/1.2.x?topic=reference-openssl-configuration-examples)
- [Инструкция от IBM Creating CA signed certificates for the monitoring infrastructure](https://www.ibm.com/docs/en/hpvs/1.2.x?topic=servers-creating-ca-signed-certificates-monitoring-infrastructure)
- [ЛУЧШАЯ инструкция на русском с сайта Сергея Голубева](https://sgolubev.ru/openssl-ca/)
- [Проксирование cockpit через Nginx](https://garrett.github.io/cockpit-project.github.io/external/wiki/Proxying-Cockpit-over-NGINX)
