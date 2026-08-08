+++
title = "transmission on a freebsd server"
date = 2026-08-05
description = "Notes on setting up transmission on a freebsd server"
+++

## Installation

```shell
pkg install transmission-daemon transmission-web transmission-cli
```

`daemon` - required to install the service

`web` - installs the web interface

`cli` - for management

## Configuration

## Enabling the service (daemon)

Via `sysrc`
```shell
sysrc transmission_enable="YES"
```

or manually add to `/etc/rc.conf`
```shell
transmission_enable="YES"
```

## Configuring `settings.json`

The config file is located at `/usr/local/etc/transmission/home/settings.json`

```
⚠️ Important:
Edit the file only while the service is stopped (service transmission stop),
otherwise Transmission will reset all changes.
```

Some values in `settings.json` are set via rc.conf variables; the default values are located at `/usr/local/etc/rc.d/transmission`.
A useful file describing all the variables, what they do, and their default values.

```
⚠️ Important:
Values are taken from the variables, so if you set a value in settings.json,
transmission may overwrite it
```

## Configuring the download directory

The download directory is set via a variable in `rc.conf`:
```shell
transmission_download_dir="/mnt/transmission/downloads"
```

Default is `/usr/local/etc/transmission/home/Downloads`

## Access configuration

### Download directory permissions

By default transmission runs under the transmission user and group, so the download directory permissions need to be set:
```shell
chown -R transmission:transmission /mnt/transmission/downloads
```

You can also add yourself to the transmission group:
```shell
pw groupmod transmission -m $USER
```

### Login/password authentication

You can add login/password access:
```json
"rpc-authentication-required": true,
"rpc-username": "your_login",
"rpc-password": "your_password",
```

If this is on a home network and login/password isn't needed, it can be disabled:
```json
"rpc-authentication-required": false,
```

### Address restrictions

You can allow access from specific devices on the network,
set via the `rpc-whitelist` parameter as a comma-separated string, with `rpc-whitelist-enabled` turned on:
```json
"rpc-whitelist": "127.0.0.1,192.168.1.2",
"rpc-whitelist-enabled": true,
```

There's a similar setting, but for restricting the addresses at which the web interface will open.

`rpc-host-whitelist` (note the word host in the name), also set as a comma-separated string:
```json
"rpc-host-whitelist": "transmission.lan,192.168.1.10",
"rpc-host-whitelist-enabled": true,
```
