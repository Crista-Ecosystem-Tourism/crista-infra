# crista-infra (Ansible bootstrap под Coolify)

Идемпотентная подготовка нового VPS (Debian/Ubuntu) под прод стек **`crs`**:
базовая система → Docker Engine + Compose plugin → Coolify.

## Требования к локальной машине

- Python 3 и `pipx`/`pip`
- Установить Ansible: `pipx install ansible` или `python3 -m pip install --user ansible`
- Установить коллекции:

  ```bash
  cd ansible
  ansible-galaxy collection install -r requirements.yml
  ```

## Подготовить inventory

```bash
cp inventory/hosts.yml.example inventory/hosts.yml
# подставить свои ansible_host / ansible_user / ansible_ssh_private_key_file
```

`inventory/hosts.yml` уже в `.gitignore`, реальный IP и путь к ключу в публичный репо не уйдут.

## Запуск

Сухой прогон (показать diff, ничего не менять):

```bash
ansible-playbook playbooks/bootstrap.yml --check --diff
```

Полный bootstrap:

```bash
ansible-playbook playbooks/bootstrap.yml
```

## Что делает playbook

1. **common**: timezone, базовые пакеты (`curl`, `git`, `ufw`, `fail2ban`, `ripgrep`, `jq`…), `unattended-upgrades`, swap-файл, `ufw` (22/80/443/8000), хардненинг sshd (`PasswordAuthentication no`, `PermitRootLogin prohibit-password`).
2. **docker**: официальный repo Docker → `docker-ce` + `docker-compose-plugin`, `systemctl enable --now docker`, проверка `docker compose version`.
3. **coolify**: официальный installer Coolify (`https://cdn.coollabs.io/coolify/install.sh`); пропускается, если Coolify уже установлен (`/data/coolify/source/docker-compose.yml`).
4. **backups**: optional systemd timer для PostgreSQL backup. По умолчанию выключен.

## Scheduled backup

На сервере создайте `/etc/crista/backup.env` с owner `root`, mode `0600`. В нём должны быть `DATABASE_URL` и `BACKUP_DIR`; для локальной медиатеки также задайте `MEDIA_STORAGE_BACKEND=local` и `MEDIA_STORAGE_DIR`. Не добавляйте этот файл в Git.

После этого включите `configure_backups: true` в inventory и запустите bootstrap. Timer запускает backup ежедневно в `03:17` в timezone сервера, сохраняет пропущенный запуск после перезагрузки и использует те же PostgreSQL dump/checksum/retention правила, что и runbook `ai_agent/ops`.

## После bootstrap (вручную, в UI Coolify)

- Зайти в Coolify (`http://<host>:8000`), создать первого админа.
- Подключить GitHub-репозиторий **`crs`** (ветка `main`) как Docker Compose Application.
- Задать переменные окружения / секреты приложения (как в текущем проде).
  - Если фронт открываете **напрямую по `http(s)://IP:3333`**, без домена Coolify, задайте **`COMPOSE_LAN_BIND=0.0.0.0`** иначе порты висят на `127.0.0.1` и снаружи не доступны (см. `crs/.env.example`).
  - Если сайт через **домен в Coolify**, в UI назначьте FQDN **сервису `frontend`** (контейнерный порт **80**): прокси ходит по внутренней сети стека.
- DNS и сертификаты.
- В GitHub секреты репозитория **`crs`** обновить **`COOLIFY_DEPLOY_WEBHOOK`** (и при необходимости **`COOLIFY_TOKEN`**) — webhook от **нового** инстанса Coolify.

## Сменить переменные

`inventory/group_vars/all.yml` — список пакетов, swap, порты ufw, ssh-флаги, `install_coolify`.

## Как это вписывается в общий пайплайн

См. `../info/Пайплайн.md` — раздел **«Смена сервера»** ссылается именно на этот playbook.
