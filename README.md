# cad_dns

Конфигурация Unbound DNS для внутренней зоны кластера `cad.internal`.

## Назначение

Сабмодуль хранит конфигурацию DNS-сервера, который используется внутри Proxmox-инфраструктуры и виртуальных сетей. DNS позволяет обращаться к инфраструктурным узлам по именам, а не по IP-адресам.

## Структура

```text
unbound.conf
unbound.conf.d/infra.conf
unbound.conf.d/remote-control.conf
unbound.conf.d/root-auto-trust-anchor-file.conf
```

`unbound.conf` подключает все дополнительные файлы из:

```text
/etc/unbound/unbound.conf.d/*.conf
```

## Основная конфигурация

Файл:

```text
unbound.conf.d/infra.conf
```

DNS-сервер слушает:

```text
0.0.0.0
10.10.0.2
```

Разрешённые клиенты:

```text
127.0.0.0/8
10.10.0.0/24
10.10.1.0/24
```

Включены:

- IPv4;
- UDP;
- TCP;
- `hide-identity`;
- `hide-version`;
- `prefetch`;
- `qname-minimisation`.

## Локальная зона

Локальная зона:

```text
cad.internal.
```

Тип зоны:

```text
static
```

В конфигурации описаны записи Proxmox-нод:

| DNS-имя | IP |
|---|---:|
| `pve1.cad.internal` | `10.18.164.185` |
| `pve2.cad.internal` | `10.18.164.182` |
| `pve3.cad.internal` | `10.18.164.181` |
| `pve4.cad.internal` | `10.18.164.180` |
| `pve5.cad.internal` | `10.18.164.167` |
| `pve6.cad.internal` | `10.18.164.166` |
| `pve7.cad.internal` | `10.18.164.165` |
| `pve8.cad.internal` | `10.18.164.164` |
| `pve9.cad.internal` | `10.18.164.163` |
| `pve10.cad.internal` | `10.18.164.162` |
| `pve11.cad.internal` | `10.18.164.161` |

Также задана запись:

```text
dns1.cad.internal. IN A 10.10.0.3
```

При этом сам Unbound слушает `10.10.0.2`, а DHCP из сабмодуля `cad_dhcp` выдаёт клиентам DNS `10.10.0.2`. Если DNS-сервер должен называться `dns1.cad.internal`, стоит привести эту запись к фактическому адресу сервера.

## DNSSEC trust anchor

Файл:

```text
unbound.conf.d/root-auto-trust-anchor-file.conf
```

Включает root trust anchor:

```text
auto-trust-anchor-file: "/var/lib/unbound/root.key"
```

## Remote control

Файл:

```text
unbound.conf.d/remote-control.conf
```

Включает локальное управление через socket:

```text
/run/unbound.ctl
```

## Установка Unbound

На Debian/Ubuntu:

```bash
apt update
apt install unbound
```

Скопировать конфигурацию:

```bash
cp unbound.conf /etc/unbound/unbound.conf
cp unbound.conf.d/*.conf /etc/unbound/unbound.conf.d/
```

Проверить синтаксис:

```bash
unbound-checkconf
```

Перезапустить:

```bash
systemctl restart unbound
systemctl status unbound
```

## Проверка

С самого DNS-сервера:

```bash
dig @127.0.0.1 pve1.cad.internal
dig @10.10.0.2 pve1.cad.internal
```

С клиентской ВМ:

```bash
dig @10.10.0.2 pve1.cad.internal
nslookup pve1.cad.internal 10.10.0.2
```

Проверить внешний DNS-рекурсивный запрос:

```bash
dig @10.10.0.2 example.com
```

## Связь с DHCP

В `cad_dhcp/kea-dhcp4.conf` клиентам выдаётся DNS:

```text
10.10.0.2
```

Поэтому адрес `10.10.0.2` должен быть назначен DNS-серверу или доступен как VIP/адрес интерфейса в сервисной сети.

## Добавление новой Proxmox-ноды

Чтобы добавить новую ноду в DNS, нужно добавить строку в `local-zone` блок:

```text
local-data: "pve12.cad.internal. IN A 10.18.164.160"
```

После изменения:

```bash
unbound-checkconf
systemctl restart unbound
```

## Типовые проблемы

Если клиенты не получают DNS:

- проверить, что DHCP выдаёт `10.10.0.2`;
- проверить, что Unbound слушает `10.10.0.2`;
- проверить firewall на DNS-сервере;
- проверить `access-control` для клиентской подсети;
- проверить, что в конце DNS-имён в `local-data` стоит точка.

Если локальные имена не резолвятся:

```bash
unbound-checkconf
journalctl -u unbound -f
```

Если внешние имена не резолвятся, проверить gateway и доступ DNS-сервера в интернет.
