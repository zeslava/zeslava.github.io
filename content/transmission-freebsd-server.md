+++
title = "transmission на сервере freebsd"
date = 2026-08-05
description = "Заметка по настройке transmission на сервере freebsd"
+++

# Установка

```shell
pkg install transmission-daemon transmission-web transmission-cli
```

`daemon` - нужен для установки службы

`web` - устанавливает web интерфейс

`cli` - для управления

# Настройка

## Включение сервиса(демона)

Через `sysrc`
```shell
sysrc transmission_enable="YES"
```

либо руками прописать в `/etc/rc.conf`
```shell
transmission_enable="YES"
```

## Настройка `settings.json`

Файл конфигурации находится тут `/usr/local/etc/transmission/home/settings.json`

```
⚠️ Важно: 
Редактировать файл только при остановленной службе (service transmission stop), 
иначе Transmission сбросит все изменения.
```

Часть значений в `settings.json` определяется через переменные rc.conf, значения по умолчанию нахоядтся тут `/usr/local/etc/rc.d/transmission`. 
Полезный файл с описанием всех переменных, для чего они и какие значения по умолчанию.

```
⚠️ Важно: 
Значения берутся из переменных, поэтому, если задать значение в settings.json, 
transmission может его перезаписать
```

## Настройка директории для загрузок

Директория для загрузок настраивается через переменную в `rc.conf`:
```shell
transmission_download_dir="/mnt/transmission/downloads"
```

По умолчанию `/usr/local/etc/transmission/home/Downloads`

## Настройка доступа

### Права на директорию загрузок

По умолчанию transmission работает под пользователем и группой transmission, поэтому для директории загрузок надо выставить права:
```shell
chown -R transmission:transmission /mnt/transmission/downloads
```

Можно так же добавить себя в группу transmission:
```shell
pw groupmod transmission -m $USER
```

### Аутентификация по логину/пролю

Можно добавить доступ по логину/паролю:
```json
"rpc-authentication-required": true,
"rpc-username": "ваш_логин",
"rpc-password": "ваш_пароль",
```

Если это в домашней сети и нет смысла во входе по логину/паролю, то можно отключить
```json
"rpc-authentication-required": false,
```

### Ограничение по адресам

Можно настроить доступ с определенных устройств в сети,
задается в параметре `rpc-whitelist` строкой со значениями через запятую и включением `rpc-whitelist-enabled`:
```json
"rpc-whitelist": "127.0.0.1,192.168.1.2",
"rpc-whitelist-enabled": true,
```

Есть аналогичная настройка, но для ограничения адресов, по которым будет открываться web интерфейс.

`rpc-host-whitelist` (слово host в названии), задается так же строкой со значениями через запятую:
```json
"rpc-host-whitelist": "transmission.lan,192.168.1.10",
"rpc-host-whitelist-enabled": true,
```
