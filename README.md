# GRE-туннель с policing 10 Мбит/с, iperf3 и мониторинг

Инструкция по запуску.

## Требования

**Управляющая машина:** macOS или Linux, Python 3.11+, [uv](https://docs.astral.sh/uv/),
SSH-ключ `~/.ssh/id_ed25519`.

**Виртуальные машины:**

| Имя | vCPU | RAM | Диск |
|---|---|---|---|
| iperf-server-1 | 1 | 1 ГБ | 10 ГБ |
| iperf-server-2 | 1 | 1 ГБ | 10 ГБ |
| monitoring-1 | 1 | 2 ГБ | 20 ГБ |

* Debian 13 (amd64).
* Все три VM в одной приватной сети, у каждой есть публичный IP.
* Начальный доступ по SSH-ключу, который выдаёт облако: `root` или пользователь
  образа с sudo. Он нужен только для первого шага: после него Ansible работает
  от пользователя `deploy`, а вход `root` по SSH и вход по паролю отключаются.
* Исходящий доступ в интернет.

## Запуск

1. Укажите публичные IP машин в inventory:

   ```bash
   cp inventory/hosts.example.yml inventory/hosts.yml
   $EDITOR inventory/hosts.yml
   ```

2. Установите зависимости (Ansible и коллекции):

   ```bash
   make venv deps
   ```

3. Разверните стенд и проверьте результат:

   ```bash
   make up
   ```

   `make up` по очереди запускает `bootstrap.yml` (один раз под начальным
   пользователем создаёт `deploy` с SSH-ключом и sudo), `site.yml` (вся настройка от
   `deploy`, первым делом запрещает вход `root` и пароли) и `verify.yml` (проверка
   критериев). Если облако выдаёт не `root`, а пользователя с sudo:
   `make up BOOTSTRAP_USER=debian`.

Те же шаги без Make:

```bash
ansible-galaxy collection install -r requirements.yml -p collections
ansible-playbook bootstrap.yml -e ansible_user=root
ansible-playbook site.yml
ansible-playbook verify.yml
```

### Параметры

| Параметр | Файл | По умолчанию |
|---|---|---|
| `tunnel_type`: `gre` или `ipip` | `inventory/group_vars/iperf_servers.yml` | `gre` |
| `tunnel_encap`: `none` (протокол 47/4 напрямую) или `fou` (туннель внутри UDP, если сеть провайдера не пропускает GRE) | `inventory/group_vars/iperf_servers.yml` | `fou` |
| `traffic_limit_ingress_rate`, `traffic_limit_egress_rate` | `inventory/group_vars/iperf_servers.yml` | `10mbit` |
| Другой SSH-ключ | `make up SSH_PUBLIC_KEY_FILE=~/.ssh/key.pub` | `~/.ssh/id_ed25519.pub` |

## Проверка

```bash
make verify
```

Запускает iperf3 через туннель (в обе стороны) и напрямую, затем проверяет
Prometheus и Grafana. Ожидаемый результат: через туннель около 10 Мбит/с, напрямую
полная скорость сети.

Вручную на `iperf-server-1`:

```bash
iperf3 -c 10.10.10.2 -t 30                     # через туннель: ~10 Мбит/с
iperf3 -c <приватный IP iperf-server-2> -t 30  # напрямую: полная скорость сети
```

Grafana и Prometheus доступны через SSH-туннель:

```bash
make ui                 # Grafana: http://localhost:3000, Prometheus: http://localhost:9090
make grafana-password   # пароль пользователя admin в Grafana
make load               # 2 минуты нагрузки через туннель
```

Дашборд **iperf nodes: CPU, RAM, network, tunnel** открывается на главной странице
Grafana. Пример под нагрузкой: [docs/dashboard.png](docs/dashboard.png).

## Команды

| Команда | Действие |
|---|---|
| `make help` | список команд |
| `make ping` | проверка SSH и sudo на всех VM |
| `make provision` | повторное применение `site.yml` (`ARGS="--tags tunnel"` для отдельных ролей) |
| `make check` | dry run: `--check --diff` |
| `make idempotency` | повторный прогон, ошибка, если есть изменения |
| `make reboot-test` | перезагрузка iperf-серверов и повторная проверка |
| `make lint` | yamllint, ansible-lint, проверка синтаксиса |
