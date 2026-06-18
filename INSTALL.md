# Installation

Краткий порядок установки Xray + XHTTP + Nginx + Let's Encrypt на Ubuntu VPS.

## Порядок работы

1. Подготовить домен и DNS-запись.
2. Установить и проверить Nginx.
3. Получить TLS-сертификат Let's Encrypt.
4. Установить Xray.
5. Настроить XHTTP.
6. Настроить клиентское подключение.

## Подробные инструкции

Каждый шаг подробно описан в папке docs/:

- docs/01-domain-and-dns.md
- docs/02-nginx.md
- docs/03-letsencrypt.md
- docs/04-xray.md
- docs/05-xhttp.md
- docs/06-client.md

## Примечание

Файл xhttp-setup.sh будет использоваться позже для автоматизации установки.
Пока основная установка выполняется вручную по документации.
