# TeachUA — розгортання за допомогою Ansible

Ця гілка (`ansible`) містить Ansible-проєкт, запускає вебдодаток TeachUA на трьох віртуальних машинах Debian: сервері бази даних, сервері backend і сервері frontend.

TeachUA — це проєкт, що складається з бекенду на Java, фронтенду на React та бази даних MariaDB/MySQL.

Одна команда налаштовує всі три машини з нуля: пакети, базу даних, застосунок, сервіси та firewall. Повторний запуск нічого не змінює, якщо машини вже в потрібному стані (ідемпотентність).

---

## Архітектура

```
              Ansible (керуюча машина, напр. bastion)
                               │  SSH
          ┌──────────────────┼───────────────────┐
          ▼                  ▼                   ▼
     ┌─────────┐         ┌───────────┐        ┌────────────┐
     │   db    │◄─────── │  backend  │◄────── │  frontend  │
     │ MariaDB │  3306   │ Java :8080│  API   │ nginx :80  │
     └─────────┘         └───────────┘        └────────────┘
                               ▲                    │
                               └──── Браузер ◄──────┘
```

| Хост | Роль | Що працює | Відкриті порти (ufw) |
|---|---|---|---|
| `db` | `db` | MariaDB з базою `teachua` | `22`, `3306` (лише з IP backend) |
| `backend` | `backend` | Spring-застосунок (`java -jar`) як сервіс systemd | `22`, `8080` |
| `frontend` | `frontend` | nginx, що віддає production-збірку React | `22`, `80` |

Браузер завантажує сайт із frontend і звертається до API backend напряму, тому адреса backend вшита в React-збірку під час її створення.

## Структура репозиторію

```
ansible/
├── ansible.cfg                       # налаштування за замовчуванням: inventory, SSH-користувач
├── inventory.ini                     # хости та їхні IP-адреси
├── requirements.yml                  # потрібні колекції Ansible
├── site.yml                          # головний плейбук
├── group_vars/
│   └── all/
│       ├── vars.yml                  # усі налаштування
│       └── vault.yml.example         # шаблон зашифрованого файлу із секретами
└── roles/
    ├── common/                       # пакети, firewall (усі хости)
    ├── db/                           # MariaDB, база, користувач, імпорт даних
    ├── backend/                      # Java, Maven, збірка, сервіс systemd
    └── frontend/                     # Node.js, збірка React, nginx
```

## Вимоги

**Керуюча машина** (де запускається Ansible):

- Debian з Ansible: `sudo apt install ansible`
- SSH-доступ до всіх цільових хостів за ключем (без запиту пароля)

**Цільові хости** (`db`, `backend`, `frontend`):

- Debian 12 або новіший, з Python 3 і SSH-сервером
- Користувач з правами `sudo`. 
- Доступ до інтернету

Рекомендована памʼять: `db` 1–2 ГБ, `backend` 2 ГБ, `frontend` 3–4 ГБ (збірка React потребує близько 2 ГБ).

## Підготовка

### 1. Встанови потрібні колекції

```bash
cd ansible
ansible-galaxy collection install -r requirements.yml
```

### 2. Налаштуй SSH-доступ

```bash
ssh-keygen -t ed25519
ssh-copy-id user@<DB_IP>
ssh-copy-id user@<BACKEND_IP>
ssh-copy-id user@<FRONTEND_IP>
```

Якщо ключ має нестандартне імʼя, потрібно вказати `private_key_file` в `ansible.cfg` або `ansible_ssh_private_key_file` в `inventory.ini`.

### 3. Вкажи адреси хостів

Відредагуй `inventory.ini` і заміни значення `ansible_host` на справжні IP-адреси своїх машин. Ці адреси використовуються й в інших місцях (правила доступу до БД, рядок підключення backend, адреса API у frontend).

### 4. Задай пароль бази даних

Пароль зберігається зашифрованим за допомогою Ansible Vault і ніколи не комітиться у відкритому вигляді:

```bash
cd group_vars/all
cp vault.yml.example vault.yml
nano vault.yml                    # впиши vault_db_password
ansible-vault encrypt vault.yml   # придумай пароль до сховища й запамʼятай його
```

### 5. Перевір зʼєднання

```bash
ansible all -m ping
```

Усі три хости мають відповісти `pong`.

## Налаштування

Усі параметри знаходяться в `group_vars/all/vars.yml`.

| Змінна | За замовчуванням | Опис |
|---|---|---|
| `repo_version` | `main` | Гілка або тег цього репозиторію, з якого збирається застосунок |
| `repo_update` | `false` | Підтягувати нові коміти при кожному запуску (див. «Розгортання нового коду») |
| `backend_subdir`, `frontend_subdir` | `teachua-backend`, `teachua-frontend` | Каталоги застосунків у репозиторії |
| `backend_war` | `dev.war` | Імʼя WAR-файлу, який створює Maven |
| `db_name`, `db_user`, `db_port` | `teachua`, `teachua`, `3306` | Параметри бази даних |
| `db_password` | з vault | Береться з `vault_db_password` у `vault.yml` |
| `java_package` | `openjdk-17-jdk` | Пакет Java для встановлення |
| `backend_port` | `8080` | Порт backend |
| `backend_java_opts` | `-Xms256m -Xmx768m` | Параметри JVM |
| `node_version` | `16.0.0` | Версія Node.js для збірки frontend |
| `npm_ci_flags` | порожньо | Додаткові прапорці для `npm ci` (наприклад, `--legacy-peer-deps`) |
| `frontend_port` | `80` | Порт сайту |
| `frontend_api_url` | `http://<IP backend>:8080` | Адреса backend, вшита в React-збірку |
| `web_root` | `/var/www/teachua` | Каталог зі статичними файлами |

## Запуск

```bash
# Пробний запуск: показує, що змінилося б, але нічого не змінює
ansible-playbook site.yml --check --ask-vault-pass --ask-become-pass

# Справжній запуск
ansible-playbook site.yml --ask-vault-pass --ask-become-pass
```

Перший запуск триває 10–20 хвилин (збірки Maven і npm).

Корисні варіанти:

```bash
ansible-playbook site.yml -K --ask-vault-pass --tags db            # лише одна роль
ansible-playbook site.yml -K --ask-vault-pass --limit backend      # лише один хост
ansible-playbook site.yml -K --ask-vault-pass -e repo_update=true  # розгорнути новий код
ansible-playbook site.yml -K --ask-vault-pass -v                   # докладний вивід
```

(`-K` — коротка форма `--ask-become-pass`.)

### Розгортання нового коду

За замовчуванням репозиторій клонується один раз і далі не чіпається, завдяки чому повторні запуски лишаються ідемпотентними. Щоб розгорнути нові коміти, запусти з `-e repo_update=true`. Змінений код запускає перезбірку, а сервіси перезапускаються автоматично.

## Перевірка

1. **Ідемпотентність.** Запусти плейбук вдруге. У `PLAY RECAP` для кожного хоста має бути `changed=0`.
2. **Сайт.** Відкрий `http://<FRONTEND_IP>` у браузері.
3. **API.**
   ```bash
   curl -i http://<BACKEND_IP>:8080/dev/api/cities
   ```
4. **Доступ до БД обмежений.** З хоста backend це працює, з будь-якого іншого має не працювати:
   ```bash
   nc -zv <DB_IP> 3306
   ```
5. **Firewall.** На кожному хості:
   ```bash
   sudo ufw status verbose
   ```
6. **Сервіси.**
   ```bash
   sudo systemctl status mariadb          # на db
   sudo systemctl status teachua-backend  # на backend
   sudo systemctl status nginx            # на frontend
   ```

## Що роблять ролі

### `common` (усі хости)
Перевіряє, що ОС належить до родини Debian, встановлює базові пакети (`sudo`, `git`, `ufw`, `acl`, `curl` та ін.), потім налаштовує firewall: спершу дозволяється SSH, вхідний трафік за замовчуванням забороняється, і лише після цього вмикається `ufw` (такий порядок не дає відрізати себе від машини).

### `db`
Встановлює MariaDB, налаштовує слухання на мережі та кодування `utf8`, створює базу даних і користувача, який може підключатися **лише з IP backend**, завантажує `data.sql` з цього репозиторію та імпортує його **лише якщо база порожня**. Порт `3306` відкривається в `ufw` тільки для IP backend.

### `backend`
Встановлює Java 17 і Maven, створює непривілейованого користувача `teachua`, клонує репозиторій і збирає WAR-файл. Застосунок працює як сервіс systemd `teachua-backend`. Параметри підключення, включно з паролем БД, зберігаються у файлі `/etc/teachua/backend.env` (права `0640`). Наприкінці роль викликає `/dev/api/cities`, щоб перевірити, що backend працює й бачить базу даних.

### `frontend`
Встановлює nginx і Node.js 16 (завантажується з nodejs.org), клонує репозиторій, збирає React-застосунок і публікує збірку в `/var/www/teachua`. nginx віддає її та повертає `index.html` для маршрутів, що обробляються на стороні клієнта.

## Деталі реалізації

- **Без контейнерів.** Застосунок працює безпосередньо на машинах. Варіант із контейнерами знаходиться в гілці `docker`.
- **Production-frontend.** React-застосунок збирається (`npm run build`) і віддається через nginx замість dev-сервера.
- **Виправлення `ROOT_SERVER`.** У `src/config/ApplicationConfig.js` використовується `ROOT_SERVER`, але його оголошення закоментоване, тому застосунок падає з помилкою `ROOT_SERVER is not defined`. Роль оголошує змінну в робочій копії на сервері зі значення `REACT_APP_ROOT_SERVER`; сам репозиторій не змінюється.
- **Адреса backend у збірці.** URL backend вшивається в React-збірку. Роль запамʼятовує, для якого URL зроблена збірка, і перезбирає її автоматично, якщо він змінився.
- **Як досягається ідемпотентність.** Збірки використовують `creates:` (пропускаються, якщо результат уже є), конфігураційні файли створюються із шаблонів (перезаписуються лише при зміні вмісту), а сервіси перезапускаються handlers лише коли щось справді змінилося.
- **Секрети.** Пароль БД зберігається у зашифрованому Vault-файлі `vault.yml` і прихований у виводі (`no_log`). Відкритим текстом лежить лише шаблон `vault.yml.example`.
- **Що не охоплено.** У Debian за замовчуванням немає SELinux (використовується AppArmor). Особливості Oracle Linux (SELinux у режимі permissive, `firewalld`) не входять до цієї гілки.

## Типові проблеми

| Симптом | Що перевірити |
|---|---|
| `UNREACHABLE ... Permission denied (publickey)` | Публічний ключ не скопійовано (`ssh-copy-id`) або неправильний `ansible_user` |
| `sudo: command not found` / користувача немає в sudoers | Встанови `sudo` і додай користувача в групу `sudo` (див. «Вимоги») |
| `Failed to set permissions on the temporary files` | Не встановлено пакет `acl`. Запусти весь плейбук, щоб `common` його встановив |
| `E: Unable to locate package openjdk-17-jdk` | Перевір `apt-cache search openjdk` і зміни `java_package` |
| `mvn package` падає | Перевір `mvn -v` (Java 17) і що `backend_war` збігається з реальним імʼям файлу в `target/` |
| `npm ci` падає через конфлікт залежностей | Задай `npm_ci_flags: "--legacy-peer-deps"` |
| Збірка React завершується `Killed` | Не вистачає памʼяті: дай хосту frontend 3–4 ГБ |
| Імпорт `data.sql` падає | Імовірно, у файлі є `CREATE DATABASE` або `USE <інша назва>`. Перевір його перші рядки |
| Backend: `Communications link failure` | Firewall на `db`, неправильний IP в `inventory.ini`, неправильний пароль. Лог: `sudo journalctl -u teachua-backend -n 50` |
| Сайт відкривається, але даних немає | Порт backend 8080 недоступний з браузера або змінився IP backend (при наступному запуску роль сама перезбере frontend) |
| 404 на `/dev/api/cities` | Інший `server.servlet.context-path` у конфігурації backend |
| Після вмикання `ufw` втрачено доступ | Зайди через консоль ВМ: `sudo ufw allow 22/tcp` |
