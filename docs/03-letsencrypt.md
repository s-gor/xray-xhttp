# 03. Let's Encrypt

На этом этапе получаем TLS-сертификат для домена.

## Перед началом

Должно быть готово:

- домен указывает на IP сервера;
- Nginx установлен и запущен;
- порт 80 доступен снаружи.

## Установка acme.sh

curl https://get.acme.sh | sh

Перезагрузить shell:

source ~/.bashrc

## Выбрать Let's Encrypt как CA

~/.acme.sh/acme.sh --set-default-ca --server letsencrypt

## Получить сертификат

Заменить example.com на свой домен:

~/.acme.sh/acme.sh --issue -d example.com --webroot /var/www/html

## Установить сертификат

mkdir -p /etc/ssl/xray

~/.acme.sh/acme.sh --install-cert -d example.com --key-file /etc/ssl/xray/private.key --fullchain-file /etc/ssl/xray/fullchain.cer --reloadcmd "systemctl reload nginx"

## Проверка

ls -l /etc/ssl/xray/

Должны появиться файлы:

private.key
fullchain.cer

## Важно

Если сертификат не выпускается, сначала проверить:

curl -I http://example.com

Если домен не открывается по HTTP, Let's Encrypt не сможет подтвердить владение доменом.
