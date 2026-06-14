# Cascade VPN (Каскадный VPN)

Плейбуки Ansible для настройки серверов каскадного VPN..

## Архитектура

Клиенты VPN → Транзитный сервер (`amnezia-wireguard`) → Интерфейс `awg0` Удаленный сервер AWG → Выход в интернет через IP-адрес удаленного сервера.

Политика маршрутизации (policy routing) на транзитном сервере перенаправляет весь трафик VPN-клиентов через удаленный туннель, а не через прямое интернет-подключение самого транзитного сервера.

## Системные требования (Prerequisites)

- Установленный локально Ansible.
- SSH-доступ к целевым хостам (по настроенным ключам).
- Использование `ansible-vault` для управления секретами.

## Структура репозитория

```
inventory.ini                     # Инвентарь хостов (host inventory)
tunnel_cascade.yml                # Главный плейбук (main playbook)
host_vars/vk-tunnel-in-tunnel.yml # Переменные конкретного хоста (непубличные/неконфиденциальные) (per-host variables (non-sensitive))
vars.yml                          # Секреты, зашифрованные с помощью vault (vault-encrypted secrets)
templates/                        # Шаблоны конфигураций Jinja2 (Jinja2 config templates)
files/                            # Статические файлы для конфигурации (разворачиваются «как есть») (static files deployed as-is)
```

## Настройка

### 1. Создание файла с секретами (vault)

Для работы плейбука требуется файл `vars.yml` со следующими переменными, зашифрованными через Ansible Vault:

```bash
ansible-vault create vars.yml
```

Добавьте в него следующее содержимое:

```yaml
vault_awg_client_private_key: "..."
vault_awg_server_public_key: "..."
vault_awg_preshared_key: "..."
vault_awg_server_endpoint: "host:port"
```

Примечание: Эти значения можно взять из конфигурации нового пользователя, созданной в клиенте AmneziaVPN.

### 2. Проверка подключения

```bash
ansible all -i inventory.ini -m ping
```

## Запуск плейбука

Развертывание каскадного VPN-туннеля на хосте `vk-tunnel-in-tunnel`:

```bash
ansible-playbook -i inventory.ini tunnel_cascade.yml --ask-vault-pass
```

Или с использованием файла пароля vault:

```bash
ansible-playbook -i inventory.ini tunnel_cascade.yml --vault-password-file ~/.vault_pass
```

## Проверка статуса туннеля на сервере

```bash
ssh root@<tunnel-in-tunnel-ip>
sudo systemctl status amnezia-awg-client
sudo docker logs amnezia-awg-client
sudo ip rule show
sudo ip route show table 100
```

## Принцип работы

Плейбук настраивает каскадный VPN на стороне `vk-tunnel-in-tunnel` следующим образом::

1. Разворачивает клиентский туннель AmneziaWG (`awg0`) до удаленного сервера, используя Docker-образ `amneziavpn/amneziawg-go:latest`.
2. Запускает контейнер в виде службы systemd (`amnezia-awg-client.service`) с сетевым режимом `--network host`, благодаря чему туннельный интерфейс появляется непосредственно в операционной системе хоста.
3. Настраивает политику маршрутизации (policy routing), чтобы весь трафик VPN-клиентов уходил через удаленный сервер, а не через основное интернет-подключение машины.
4. Добавляет исключение в iptables, чтобы WireGuard-сервер (к которому подключаются конечные устройства, например, ноутбуки) продолжал отвечать клиентам напрямую.
