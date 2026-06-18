# 04. Xray

На этом этапе устанавливаем Xray и проверяем, что сервис запускается.

## Установка Xray

bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)"

## Проверка сервиса

systemctl status xray

Сервис должен быть в состоянии:

active (running)

## Основной файл конфигурации

После установки основной конфиг обычно находится здесь:

/usr/local/etc/xray/config.json

## Проверка конфигурации

Перед перезапуском сервиса желательно проверить конфиг:

xray run -test -config /usr/local/etc/xray/config.json

Если ошибок нет, можно перезапустить Xray:

systemctl restart xray

## Просмотр логов

journalctl -u xray -n 50 --no-pager

## Важно

На этом этапе Xray только устанавливается и проверяется.

Настройка XHTTP будет сделана на следующем шаге.
