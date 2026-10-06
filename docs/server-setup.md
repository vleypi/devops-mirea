# Конфигурация виртуальной машины devops-vm

## 1. Параметры машины

| Параметр | Значение |
|---|---|
| Средство виртуализации | VirtualBox 7.2, хост macOS (Apple Silicon) |
| Гостевая ОС | Ubuntu Server 24.04.5 LTS (arm64) |
| Имя узла | `devops-vm`, полное имя `devops-vm.devops.local` |
| Оперативная память | 2048 МБ |
| Процессор | 2 ядра |
| Диск | 25 ГБ, VDI, динамически расширяемый, разметка LVM |

## 2. Сетевые интерфейсы

| Интерфейс | Адаптер VirtualBox | Адрес | Назначение |
|---|---|---|---|
| `enp0s8` | Адаптер 1, NAT | `10.0.2.15/24` (DHCP) | Выход в интернет, доступ с хоста через проброс порта |
| `enp0s9` | Адаптер 2, виртуальная сеть (Host-only) `HostNetwork` | `192.168.56.3/24` (DHCP) | Прямой доступ с хостовой системы по имени `devops.local` |

Интерфейс `enp0s9` настроен файлом `/etc/netplan/60-hostonly.yaml`.
DNS-серверы `8.8.8.8` и `1.1.1.1` заданы в `/etc/systemd/resolved.conf.d/dns.conf`.
На хостовой системе в `/etc/hosts` добавлена запись `192.168.56.3 devops.local`.

## 3. Правило проброса портов

| Имя | Протокол | Адрес хоста | Порт хоста | Порт гостя |
|---|---|---|---|---|
| ssh | TCP | 127.0.0.1 | 2222 | 2222 |

Изначально порт гостя был 22. Он изменён на 2222 после переноса службы SSH на нестандартный порт.

## 4. Учётные записи

| Имя | Группы | Способ аутентификации |
|---|---|---|
| `student` | `sudo` и стандартные группы установщика | Создана при установке. Вход по SSH запрещён директивой `AllowUsers` |
| `devops` | `devops`, `sudo`, `users` | Вход по SSH только по ключу ED25519 (`~/.ssh/devops_vm` на хосте). Пароль используется только для `sudo` |
| `root` | | Вход по SSH запрещён (`PermitRootLogin no`) |

## 5. Служба SSH

Порт: `2222`.

Изменённые директивы находятся в файле `/etc/ssh/sshd_config.d/99-hardening.conf`. Основной файл `/etc/ssh/sshd_config` не изменялся, его копия сохранена в `/etc/ssh/sshd_config.backup`.

| Директива | Значение |
|---|---|
| `Port` | `2222` |
| `PermitRootLogin` | `no` |
| `PasswordAuthentication` | `no` |
| `PubkeyAuthentication` | `yes` |
| `PermitEmptyPasswords` | `no` |
| `MaxAuthTries` | `3` |
| `LoginGraceTime` | `30` |
| `AllowUsers` | `devops` |
| `X11Forwarding` | `no` |
| `ClientAliveInterval` | `300` |
| `ClientAliveCountMax` | `2` |

В файле `/etc/ssh/sshd_config.d/50-cloud-init.conf` строка `PasswordAuthentication yes` закомментирована.
Юнит `ssh.socket` отключён, служба запускается через `ssh.service`.

## 6. Правила межсетевого экрана

Политики по умолчанию: `deny (incoming)`, `allow (outgoing)`, `disabled (routed)`. Журналирование: `medium`.

| Порт | Действие | Комментарий |
|---|---|---|
| `2222/tcp` | `LIMIT IN` | SSH rate-limited |
| `80/tcp` | `ALLOW IN` | HTTP |
| `443/tcp` | `ALLOW IN` | HTTPS |

Те же правила действуют для IPv6.

## 7. Снимки состояния

| Снимок | Момент создания |
|---|---|
| `01-clean-install` | 29.09.2026 20:37, после установки ОС и обновления пакетов |
| `03-ssh-hardened` | 29.09.2026, после настройки входа по ключу и усиления защиты SSH |
| `04-nginx-https` | 06.10.2026, после установки nginx, перевода ресурса на HTTPS и первого деплоя (ПР № 6) |

## 8. Веб-сервер

Пакет: nginx 1.24.0 из репозитория Ubuntu (sudo apt install -y nginx).

| Параметр | Значение |
|---|---|
| Конфигурация ресурса | /etc/nginx/sites-available/devops-site, ссылка в /etc/nginx/sites-enabled/ |
| Стандартный ресурс | ссылка /etc/nginx/sites-enabled/default удалена, конфигурация сохранена |
| Каталог ресурса | /var/www/devops-site, владелец devops:devops |
| Права ресурса | каталоги 755, файлы 644 (rsync --chmod=D755,F644) |
| Сертификат | /etc/ssl/certs/devops.crt, root:root, права 644 |
| Закрытый ключ | /etc/ssl/private/devops.key, root:root, права 600 |
| Протоколы TLS | TLSv1.2, TLSv1.3 |
| Перенаправление | HTTP (80) на HTTPS (443), код 301 |
| Журналы | /var/log/nginx/devops-site.access.log, /var/log/nginx/devops-site.error.log |

Команда формирования сертификата:

    sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
      -keyout /etc/ssl/private/devops.key \
      -out /etc/ssl/certs/devops.crt \
      -subj "/CN=devops.local" \
      -addext "subjectAltName=DNS:devops.local"

Срок действия: 365 дней, с 06.10.2026 по 06.10.2027.
На хостовой системе сертификат сохранён в ~/devops.crt и передаётся curl параметром --cacert.
