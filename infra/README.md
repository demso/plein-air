# plein-air infra — omp + ompweb на VPS

Мульти-инстанс хостинг [oh-my-pi](https://github.com/can1357/oh-my-pi) +
[ompweb](https://github.com/kahme247/ompweb): каждому пользователю — свой
linux-user, свой `~/.omp/agent`, свой systemd-сервис `ompweb@<name>` и свой
URL `https://<домен>:<порт>` с TLS от Caddy. Метрики и логи alloy шлёт на
центральный сервер мониторинга (Prometheus remote_write + Loki).

## Структура

```
site.yml                     # единый плейбук
inventory/
  hosts.ini.example          # → hosts.ini (генерирует визард)
  group_vars/all/
    main.yml                 # домен, диапазон портов, бэкапы
    zz-local.yml             # переопределения от визарда (домен, URL мониторинга)
    users.yml.example        # → users.yml: список пользователей и портов
    vault.yml.example        # → vault.yml: пароли (ansible-vault!)
roles/
  base/                      # пакеты, admin-user, sshd, UFW, swap 512M, апгрейды
  runtimes/                  # Node 22 + bun + глобальные omp/ompweb (npm -g)
  ompweb/                    # per-user systemd instances ompweb + health-check
  filebrowser/               # веб-доступ к home-каталогам (один инстанс, per-user scope)
  caddy/                     # edge: TLS (ACME) + reverse proxy ко всем сервисам
  monitoring/                # node_exporter + alloy → центральный сервер
  backup/                    # ежедневный tar ~/.omp → /var/backups/omp (systemd timer)
scripts/setup-wizard.sh      # интерактивная подготовка inventory/users/vault
agent-config/                # скелет .agents/.omp, копируется каждому юзеру при создании
```

## Быстрый старт

1. `scripts/setup-wizard.sh` — соберёт inventory, список пользователей, секреты.
2. `ansible-galaxy collection install -r requirements.yml`
3. `ansible-playbook site.yml --ask-vault-pass` (первый прогон от root, `-k` если по паролю).
4. После прогона: поменяй в `inventory/hosts.ini` `ansible_user` на `deploy`.

## Добавить пользователя

1. Допиши `- {name: newuser, port: 30XXX}` в `inventory/group_vars/all/users.yml`.
2. Добавь пароль в `vault.yml`: `ansible-vault edit inventory/group_vars/all/vault.yml`.
3. `ansible-playbook site.yml --ask-vault-pass`.

## Эксплуатация

- Логи инстанса: `journalctl -u ompweb@alice -f`
- Конфиг инстанса: `/home/alice/.omp/agent/web-service.env` → `systemctl restart ompweb@alice`
- Обновление omp/ompweb: `npm install -g @oh-my-pi/pi-coding-agent@latest @kahme247/ompweb@latest`,
  затем `systemctl restart 'ompweb@*'`
- Бэкап вручную: `systemctl start omp-backup.service`; архивы в `/var/backups/omp/`.

## Файлы пользователей (filebrowser)

- URL: `https://<домен>:30200` — каждый юзер логинится своим именем и видит только свой `/home/<name>`.
- Пароль: тот же, что для ompweb (`omp_web_passwords` в vault.yml); смена = правка vault + перепрогон.
- Админских аккаунтов в UI нет (дефолтный admin удаляется). Управление — CLI на хосте:
  `filebrowser users ls -d /etc/filebrowser/filebrowser.db`
- Обновление: bump `filebrowser_version` в `main.yml` + перепрогон.

## Offsite-бэкапы

Локальные архивы не спасут при смерти VPS. Подними restic/rclone с копией
`/var/backups/omp` в объектное хранилище (S3/B2) — отдельная роль по запросу.

## Мониторинг

На центральном сервере в Uptime Kuma добавь мониторы:
`https://<домен>:<порт>` на каждого пользователя (ожидаемый код 200).
