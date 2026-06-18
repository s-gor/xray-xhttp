![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04%20%7C%2026.04-E95420?logo=ubuntu&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-%20Proxy-009639?logo=nginx&logoColor=white)
![Xray](https://img.shields.io/badge/Xray-XHTTP-2F80ED)
![Let's Encrypt](https://img.shields.io/badge/Let's%20Encrypt-TLS-003A70?logo=letsencrypt&logoColor=white)


# Xray + XHTTP + Nginx + Let's Encrypt

Пошаговая настройка Xray XHTTP за Nginx с TLS-сертификатом Let's Encrypt.

Итоговая схема:

```text
Client -> domain:443 -> Nginx -> 127.0.0.1:8443 -> Xray XHTTP
```

Снаружи открыты только:

```text
80/tcp
443/tcp
```

Xray слушает только локально:

```text
127.0.0.1:8443
```

Порт `8443` наружу открывать не нужно.

---

## Для кого этот гайд

Этот гайд не самый короткий. Он рассчитан не на профессионалов, которые каждый день работают с Nginx, Xray, TLS-сертификатами и Linux-сервисами.

Цель — пройти настройку спокойно и последовательно: шаг за шагом, с проверками после каждого важного действия.

Здесь специально показано:

* зачем выполняется каждый этап;
* какую команду запускать;
* что должно быть видно, если всё прошло успешно;
* где находятся важные файлы.

Если вы уже опытный администратор, часть пояснений может показаться очевидной.

Если вы только только в начале пути, не спешите: лучше внимательно пройти все этапы по порядку, чем пропустить проверку и потом искать ошибку в конце.

---

## Как выполнять команды

Все команды выполняются на сервере от `root`.

Короткие команды в инструкции даны одной строкой. Их можно копировать и выполнять целиком.

Большие блоки с `cat <<EOF` создают файлы. Их нужно копировать целиком, включая последнюю строку `EOF`.

В инструкции не используется `nano`. Конфигурационные файлы создаются командами, чтобы избежать ручных ошибок при копировании домена, UUID и XHTTP path.

После проверочных команд всегда указано, что должно быть видно в нормальном случае.

---

## Содержание

1. Указать домен для установки
2. Обновить сервер и установить пакеты
3. Проверить DNS и HTTP
4. Создать страницу-заглушку
5. Выпустить TLS-сертификат Let's Encrypt
6. Настроить Nginx на HTTPS
7. Установить Xray
8. Создать рабочие значения Xray
9. Создать `config.json` для Xray
10. Добавить XHTTP location в Nginx
11. Создать клиентскую ссылку
12. Финальная проверка

---

## Требования

Для этой настройки нужен сервер с Ubuntu Server 26.04 LTS или другой совместимой Ubuntu/Debian-системой.

Также обязательно нужен домен или DDNS-имя, которое указывает на внешний IPv4 сервера.

Без домена эта схема не заработает нормально, потому что:

* Let's Encrypt должен проверить домен перед выпуском TLS-сертификата;
* Nginx принимает HTTPS-подключение именно на домене;
* клиентская VLESS-ссылка использует домен, SNI и Host.

Можно использовать обычный домен или DDNS-имя.

На сервере или в Security Group должны быть открыты входящие порты:

| Порт      | Зачем нужен                          |
| --------- | ------------------------------------ |
| `80/tcp`  | HTTP и проверка домена Let's Encrypt |
| `443/tcp` | HTTPS и клиентское подключение       |

Порт `8443/tcp` наружу не открывается. Он используется только внутри сервера для связи Nginx с Xray.

---

## 1. Указать домен для установки

Сначала указываем домен, который будет использоваться для всей настройки.

Это нужно, чтобы дальше не вписывать домен вручную в каждую команду. Мы один раз сохраняем его в рабочий файл:

```text
/root/xray-xhttp-values.env
```

После этого остальные команды будут брать домен из этого файла автоматически.

Скопируйте строку, выполните её и введите свой домен, когда терминал спросит:

```bash
bash -c 'read -rp "Enter your domain: " DOMAIN; printf "DOMAIN=\"%s\"\n" "$DOMAIN" > /root/xray-xhttp-values.env; chmod 600 /root/xray-xhttp-values.env; cat /root/xray-xhttp-values.env'
```

Терминал спросит:

```text
Enter your domain:
```

Введите свой домен и нажмите `Enter`.

> [!TIP]
> Правильно, если видно ваш домен:
>
> ```text
> DOMAIN="example.com"
> ```

---

## 2. Обновить сервер и установить пакеты

На этом этапе обновляем систему и ставим минимальный набор пакетов для Nginx, DNS-проверок, TLS-сертификата и генерации случайных значений.

```bash
apt update && apt upgrade -y
```

```bash
apt install -y curl wget unzip tar nginx dnsutils ca-certificates openssl
```

Запустить Nginx и добавить его в автозагрузку:

```bash
systemctl enable --now nginx
```

Проверить Nginx:

```bash
systemctl status nginx --no-pager
```

> [!TIP]
> Правильно, если видно:
>
> ```text
> active (running)
> ```

---

## 3. Проверить DNS и HTTP

Перед выпуском TLS-сертификата нужно убедиться, что домен действительно указывает на этот сервер, а порт `80/tcp` доступен снаружи.

Let's Encrypt будет проверять домен именно через HTTP.

Проверить внешний IPv4 сервера:

```bash
curl -4 ifconfig.me
```

> [!TIP]
> Правильно, если показан внешний IP этого сервера.

Проверить, куда указывает домен:

```bash
bash -c 'source /root/xray-xhttp-values.env; dig +short "$DOMAIN"'
```

> [!TIP]
> Правильно, если в ответе виден внешний IP этого сервера.

Если `dig` показывает несколько IP-адресов, убедитесь, что нужный домен действительно ведёт на этот сервер.

Проверить HTTP:

```bash
bash -c 'source /root/xray-xhttp-values.env; curl -I "http://$DOMAIN"'
```

> [!TIP]
> Правильно, если есть HTTP-ответ, например:
>
> ```text
> HTTP/1.1 200 OK
> ```

Если ответа нет, проверьте DNS, Nginx и входящий порт `80/tcp`.

---

## 4. Создать страницу-заглушку

Эта страница нужна для обычных запросов к сайту. Пока Xray будет скрыт за отдельным XHTTP path, домен должен выглядеть как обычный HTTPS-сайт.

Скопируйте весь блок целиком и выполните его в терминале:

```bash
cat > /var/www/html/index.html <<'EOF'
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Bitrate Reference</title>
  <style>
    body { font-family: Arial, sans-serif; margin: 40px; line-height: 1.6; }
    .card { max-width: 720px; padding: 24px; border: 1px solid #ddd; border-radius: 12px; }
    code { background: #f4f4f4; padding: 2px 6px; border-radius: 4px; }
  </style>
</head>
<body>
  <div class="card">
    <h1>Bitrate Reference</h1>
    <p>This page provides a simple reference point for checking web availability.</p>
    <ul>
      <li>128 kbps — basic streaming quality</li>
      <li>256 kbps — improved lossy quality</li>
      <li>320 kbps — high bitrate lossy audio</li>
      <li>Lossless — depends on codec, sample rate and bit depth</li>
    </ul>
    <p>Status: <code>online</code></p>
  </div>
</body>
</html>
EOF
```

Проверить:

```bash
bash -c 'source /root/xray-xhttp-values.env; curl -I "http://$DOMAIN"'
```

> [!TIP]
> Правильно, если видно:
>
> ```text
> HTTP/1.1 200 OK
> ```

---

## 5. Выпустить TLS-сертификат Let's Encrypt

TLS-сертификат нужен для HTTPS на порту `443`. Клиент будет подключаться к домену через TLS, а Nginx будет принимать это подключение с настоящим сертификатом.

Установить `acme.sh`:

```bash
curl https://get.acme.sh | sh
```

Проверить установку:

```bash
/root/.acme.sh/acme.sh --version
```

> [!TIP]
> Правильно, если выводится версия `acme.sh`.

Выбрать Let's Encrypt:

```bash
/root/.acme.sh/acme.sh --set-default-ca --server letsencrypt
```

> [!TIP]
> Правильно, если команда завершилась без ошибки.

Выпустить сертификат:

```bash
bash -c 'source /root/xray-xhttp-values.env; /root/.acme.sh/acme.sh --issue -d "$DOMAIN" --webroot /var/www/html'
```

> [!TIP]
> Правильно, если в конце видно сообщение об успешном выпуске сертификата.

Создать каталог для сертификата:

```bash
mkdir -p /etc/ssl/xray
```

Установить сертификат:

```bash
bash -c 'source /root/xray-xhttp-values.env; /root/.acme.sh/acme.sh --install-cert -d "$DOMAIN" --key-file /etc/ssl/xray/private.key --fullchain-file /etc/ssl/xray/fullchain.cer --reloadcmd "systemctl reload nginx"'
```

Проверить файлы:

```bash
ls -l /etc/ssl/xray/
```

> [!TIP]
> Правильно, если видны два файла:
>
> ```text
> private.key
> fullchain.cer
> ```

Если сертификат не выпустился, проверьте:

* домен указывает на внешний IP сервера;
* порт `80/tcp` открыт снаружи;
* Nginx запущен;
* `curl -I http://domain` возвращает HTTP-ответ.

---

## 6. Настроить Nginx на HTTPS

Теперь подключаем сертификат к Nginx. После этого домен должен открываться по HTTPS, но Xray ещё не участвует.

На этом этапе Nginx обслуживает обычную страницу-заглушку.

Скопируйте весь блок целиком и выполните его в терминале:

```bash
source /root/xray-xhttp-values.env
cat > /etc/nginx/sites-available/xray-xhttp.conf <<EOF
server {
    listen 80;
    server_name $DOMAIN;

    root /var/www/html;
    index index.html;

    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    location / {
        return 301 https://\$host\$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name $DOMAIN;

    ssl_certificate     /etc/ssl/xray/fullchain.cer;
    ssl_certificate_key /etc/ssl/xray/private.key;

    root /var/www/html;
    index index.html;

    location / {
        try_files \$uri \$uri/ =404;
    }
}
EOF
```

Включить конфиг:

```bash
rm -f /etc/nginx/sites-enabled/default; ln -sf /etc/nginx/sites-available/xray-xhttp.conf /etc/nginx/sites-enabled/xray-xhttp.conf
```

Проверить Nginx:

```bash
nginx -t
```

> [!TIP]
> Правильно, если видно:
>
> ```text
> syntax is ok
> test is successful
> ```

Применить конфиг:

```bash
systemctl reload nginx
```

Проверить HTTPS:

```bash
bash -c 'source /root/xray-xhttp-values.env; curl -I "https://$DOMAIN"'
```

> [!TIP]
> Правильно, если есть HTTP-ответ без ошибки сертификата, например:
>
> ```text
> HTTP/2 200
> ```

---

## 7. Установить Xray

Теперь устанавливаем Xray. Он будет работать только как локальный backend за Nginx.

Внешние подключения напрямую к Xray не открываем.

```bash
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)"
```

Проверить:

```bash
xray version
```

> [!TIP]
> Правильно, если выводится версия Xray.

---

## 8. Создать рабочие значения Xray

На этом этапе создаём параметры для Xray:

* локальный порт `8443`;
* случайный XHTTP path;
* UUID первого пользователя.

Эти значения сохраняются в тот же рабочий файл. Потом из него будут созданы `config.json` и клиентская ссылка.

> [!WARNING]
> Команду из этого шага выполняйте один раз.
>
> Повторный запуск создаст новый `XHTTP_PATH` и новый `USER_UUID`.
>
> Если клиентская ссылка уже была создана, после повторного запуска её нужно будет создать заново.

```bash
bash -c 'source /root/xray-xhttp-values.env; XHTTP_LOCAL_PORT="8443"; XHTTP_PATH="/$(openssl rand -hex 8)"; USER_UUID="$(xray uuid)"; printf "DOMAIN=\"%s\"\nXHTTP_LOCAL_PORT=\"%s\"\nXHTTP_PATH=\"%s\"\nUSER_UUID=\"%s\"\n" "$DOMAIN" "$XHTTP_LOCAL_PORT" "$XHTTP_PATH" "$USER_UUID" > /root/xray-xhttp-values.env; chmod 600 /root/xray-xhttp-values.env; cat /root/xray-xhttp-values.env'
```

> [!TIP]
> Правильно, если видны четыре заполненные строки:
>
> ```text
> DOMAIN="ваш-домен"
> XHTTP_LOCAL_PORT="8443"
> XHTTP_PATH="/случайный-путь"
> USER_UUID="uuid-пользователя"
> ```

---

## 9. Создать config.json для Xray

Теперь создаём конфигурацию Xray из сохранённых значений.

Xray будет слушать только локальный адрес:

```text
127.0.0.1:8443
```

UUID и XHTTP path вручную не вставляются.

Скопируйте весь блок целиком и выполните его в терминале:

```bash
source /root/xray-xhttp-values.env
cat > /usr/local/etc/xray/config.json <<EOF
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "tag": "vless-xhttp-in",
      "listen": "127.0.0.1",
      "port": $XHTTP_LOCAL_PORT,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "$USER_UUID",
            "email": "user@$DOMAIN"
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "xhttp",
        "xhttpSettings": {
          "path": "$XHTTP_PATH"
        }
      }
    }
  ],
  "outbounds": [
    {
      "tag": "direct",
      "protocol": "freedom",
      "settings": {}
    },
    {
      "tag": "blocked",
      "protocol": "blackhole",
      "settings": {}
    }
  ]
}
EOF
```

Выставить права:

```bash
chown root:root /usr/local/etc/xray/config.json; chmod 644 /usr/local/etc/xray/config.json
```

Проверить конфиг:

```bash
xray run -test -config /usr/local/etc/xray/config.json
```

> [!TIP]
> Правильно, если команда завершается без ошибок.
>
> Не должно быть строк вида:
>
> ```text
> error
> failed
> invalid
> ```

Запустить Xray:

```bash
systemctl restart xray
```

Проверить статус:

```bash
systemctl status xray --no-pager
```

> [!TIP]
> Правильно, если видно:
>
> ```text
> active (running)
> ```

Посмотреть последние логи:

```bash
journalctl -u xray -n 20 --no-pager
```

> [!TIP]
> Правильно, если нет ошибок запуска:
>
> ```text
> permission denied
> failed
> invalid
> ```

Проверить локальный порт:

```bash
ss -tulpn | grep :8443
```

> [!TIP]
> Правильно, если видно:
>
> ```text
> 127.0.0.1:8443
> ```

---

## 10. Добавить XHTTP location в Nginx

На этом этапе связываем внешний HTTPS-вход с локальным Xray.

Nginx будет принимать запросы на секретный XHTTP path и передавать их внутрь на:

```text
127.0.0.1:8443
```

Для location используется regex-формат:

```nginx
location ~ ^/path(/|$)
```

Такой вариант принимает запросы на `/path`, `/path/` и подпути XHTTP.

Скопируйте весь блок целиком и выполните его в терминале:

```bash
source /root/xray-xhttp-values.env
cp /etc/nginx/sites-available/xray-xhttp.conf /etc/nginx/sites-available/xray-xhttp.conf.bak
cat > /etc/nginx/sites-available/xray-xhttp.conf <<EOF
server {
    listen 80;
    server_name $DOMAIN;

    root /var/www/html;
    index index.html;

    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    location / {
        return 301 https://\$host\$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name $DOMAIN;

    ssl_certificate     /etc/ssl/xray/fullchain.cer;
    ssl_certificate_key /etc/ssl/xray/private.key;

    root /var/www/html;
    index index.html;

    location ~ ^$XHTTP_PATH(/|$) {
        proxy_redirect off;
        proxy_pass http://127.0.0.1:8443;

        proxy_http_version 1.1;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;

        proxy_buffering off;
        proxy_request_buffering off;
    }

    location / {
        try_files \$uri \$uri/ =404;
    }
}
EOF
```

Проверить Nginx:

```bash
nginx -t
```

> [!TIP]
> Правильно, если видно:
>
> ```text
> syntax is ok
> test is successful
> ```

Применить конфиг:

```bash
systemctl reload nginx
```

Проверить обычный HTTPS-сайт:

```bash
bash -c 'source /root/xray-xhttp-values.env; curl -I "https://$DOMAIN"'
```

> [!TIP]
> Правильно, если есть HTTP-ответ без ошибки сертификата, например:
>
> ```text
> HTTP/2 200
> ```

---

## 11. Создать клиентскую ссылку

Теперь собираем VLESS-ссылку для клиента.

Ссылка создаётся автоматически из сохранённых значений:

* домен;
* UUID;
* XHTTP path;
* TLS/SNI/Host.

```bash
bash -c 'source /root/xray-xhttp-values.env; ENCODED_PATH="${XHTTP_PATH//\//%2F}"; CLIENT_LINK="vless://${USER_UUID}@${DOMAIN}:443?encryption=none&security=tls&sni=${DOMAIN}&host=${DOMAIN}&type=xhttp&path=${ENCODED_PATH}#${DOMAIN}-xhttp"; echo "$CLIENT_LINK" | tee /root/xray-client-link.txt'
```

Проверить файл со ссылкой:

```bash
cat /root/xray-client-link.txt
```

> [!TIP]
> Правильно, если видна строка вида:
>
> ```text
> vless://uuid@domain:443?encryption=none&security=tls&sni=domain&host=domain&type=xhttp&path=%2Fpath#domain-xhttp
> ```

Импортируйте эту ссылку в клиент.

Если подключение включается и сайты открываются, настройка завершена.

---

## 12. Финальная проверка

Этот этап нужен не для настройки, а для быстрой проверки всей цепочки после установки.

Проверить Nginx:

```bash
nginx -t
```

> [!TIP]
> Правильно, если видно:
>
> ```text
> syntax is ok
> test is successful
> ```

Проверить Xray:

```bash
systemctl status xray --no-pager
```

> [!TIP]
> Правильно, если видно:
>
> ```text
> active (running)
> ```

Проверить последние логи Xray:

```bash
journalctl -u xray -n 20 --no-pager
```

> [!TIP]
> Правильно, если нет ошибок запуска.

Проверить локальный порт Xray:

```bash
ss -tulpn | grep :8443
```

> [!TIP]
> Правильно, если видно:
>
> ```text
> 127.0.0.1:8443
> ```

Проверить клиентскую ссылку:

```bash
cat /root/xray-client-link.txt
```

> [!TIP]
> Если клиент подключается и интернет работает, вся цепочка настроена правильно.

---

## Что считается успешной установкой

Установка считается успешной, если:

* `nginx -t` показывает `test is successful`;
* `systemctl status xray --no-pager` показывает `active (running)`;
* `ss -tulpn | grep :8443` показывает `127.0.0.1:8443`;
* клиент импортирует ссылку и открывает сайты.

---

## Важные файлы

| Файл                                             | Назначение                              |
| ------------------------------------------------ | --------------------------------------- |
| `/root/xray-xhttp-values.env`                    | домен, XHTTP path, UUID, локальный порт |
| `/root/xray-client-link.txt`                     | клиентская VLESS-ссылка                 |
| `/usr/local/etc/xray/config.json`                | конфигурация Xray                       |
| `/etc/nginx/sites-available/xray-xhttp.conf`     | конфигурация Nginx                      |
| `/etc/nginx/sites-available/xray-xhttp.conf.bak` | резервная копия Nginx-конфига           |
| `/etc/ssl/xray/fullchain.cer`                    | TLS-сертификат                          |
| `/etc/ssl/xray/private.key`                      | приватный ключ TLS-сертификата          |

---

## Коротко о логике

Nginx принимает внешний HTTPS-трафик на порту `443`.

Обычные запросы открывают страницу-заглушку.

Запросы на XHTTP path передаются внутрь на:

```text
127.0.0.1:8443
```

Xray напрямую наружу не открыт.
