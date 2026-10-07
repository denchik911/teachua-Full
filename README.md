# TeachUA — Розгортання за допомогою Docker

Ця гілка (`docker`) запускає вебдодаток TeachUA у контейнерах за допомогою Docker Compose.

TeachUA — це проєкт, що складається з бекенду на Java, фронтенду на React та бази даних MariaDB/MySQL.

Передбачено два режими розгортання:

| Режим | Файл Compose | База даних |
|---|---|---|
| Зовнішня база даних | `docker-compose.external-db.yml` | Окремий сервер (наприклад, ВМ Oracle Linux 10) |
| Локальна база даних | `docker-compose.local-db.yml` | Контейнер MariaDB на тому ж хості |

Усі налаштування для обох режимів містяться в одному файлі `.env`.

---

## Архітектура

```
 Браузер
    │
    ├──► frontend  (Сервер розробки React, :3000)
    │
    └──► backend   (Spring, :8080)
              │
              └──► MariaDB
                     ├─ режим external-db: окрема ВМ, порт 3306
                     └─ режим local-db:    контейнер "db", лише внутрішня мережа
```

Браузер звертається до бекенду напряму, тому `APP_HOST` має бути адресою, доступною **з браузера**, а не внутрішнім ім'ям Docker.

## Структура репозиторію

```
.
├── teachua-backend/
│   ├── Dockerfile                    # збірка в кілька етапів: Maven (JDK 17) -> JRE 17, non-root
│   └── data.sql                      # схема та початкові дані
├── teachua-frontend/
│   ├── Dockerfile                    # Node 16, запускає `npm start`
├── docker-compose.external-db.yml    # бекенд + фронтенд, база даних на іншому сервері
├── docker-compose.local-db.yml       # бекенд + фронтенд + контейнер MariaDB
├── .env.example                      # шаблон для всіх змінних
└── README.md
```

## Передумови

- Docker Engine з Docker Compose v2 (`docker compose version`)
- Принаймні 4 ГБ оперативної пам'яті (сервер розробки React та збірка Maven вимагають багато пам'яті)
- Вільні порти: `3000` (фронтенд) та `8080` (бекенд)
- Для режиму external-db: доступний сервер MariaDB/MySQL із заздалегідь підготовленою базою даних `teachua` та користувачем, якому дозволено підключатися з цього хоста


## Налаштування

```bash
cp .env.example .env
nano .env
```

Файл `.env` ігнорується системою git. Ніколи не комітьте його.

| Змінна | Опис | Використовується в |
|---|---|---|
| `DB_HOST` | Адреса сервера бази даних | external-db |
| `DB_PORT` | Порт бази даних (за замовчуванням `3306`) | external-db |
| `DB_NAME` | Назва бази даних (за замовчуванням `teachua`) | обох |
| `DB_USER` | Користувач бази даних | обох |
| `DB_PASSWORD` | Пароль користувача бази даних | обох |
| `DB_ROOT_PASSWORD` | Пароль root для MariaDB | local-db |
| `JDBC_DRIVER` | Клас JDBC-драйвера (`org.mariadb.jdbc.Driver`) | обох |
| `BACKEND_PORT` | Опублікований порт бекенду (за замовчуванням `8080`) | обох |
| `APP_HOST` | IP або ім'я хоста цього комп'ютера, видиме з браузера | обох |
| `FRONTEND_PORT` | Опублікований порт фронтенду (за замовчуванням `3000`) | обох |
| `IMAGE_TAG` | Тег для зібраних образів (за замовчуванням `local`) | обох |

У режимі local-db бекенд завжди підключається до хоста `db` (контейнера бази даних), тому `DB_HOST` там ігнорується. Той самий файл `.env` працює для обох режимів.

## Запуск

### Режим 1: база даних на іншому сервері

Підготуйте базу даних один раз на сервері баз даних:

```sql
CREATE DATABASE teachua;
CREATE USER 'teachua'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON teachua.\* TO 'teachua'@'%';
FLUSH PRIVILEGES;
```
На сервері з базою даних імпортувати базу даних з репозиторію 
```bash
mariadb -u teachua -p teachua < data.sql
```
Потім на сервері з проєктом можна запустити команду
```bash

docker compose -f docker-compose.external-db.yml up -d --build
```

### Режим 2: база даних у контейнері

```bash
docker compose -f docker-compose.local-db.yml up -d --build
```

Файл `data.sql` імпортується автоматично під час першого запуску (коли том `dbdata` порожній).

Не запускайте обидва режими одночасно: вони використовують однакові порти та назву проєкту. Зупиніть один перед запуском іншого.

## Перевірка

```bash
docker compose -f <compose-file> ps
curl -I http://<APP_HOST>:3000
curl -I http://<APP_HOST>:8080
```

Перший запуск фронтенду займає кілька хвилин. Дочекайтеся повідомлення `Compiled successfully` у виводі:

```bash
docker compose -f <compose-file> logs -f frontend
```

Після цього відкрийте `http://<APP_HOST>:3000`.

## Корисні команди

```bash
docker compose -f <compose-file> logs -f backend      # перегляд логів бекенду в реальному часі
docker compose -f <compose-file> restart backend      # перезапуск одного сервісу
docker compose -f <compose-file> up -d --build        # перезбірка та застосування змін
docker compose -f <compose-file> down                 # зупинка та видалення контейнерів
docker compose -f <compose-file> down -v              # також видалити том бази даних (local-db)
```

## Нотатки щодо реалізації

- **Образ бекенду.** Багатоетапна збірка (multi-stage build): WAR збирається за допомогою Maven на JDK 17 і запускається на базі JRE 17 від імені непривілейованого користувача (UID 10001). Налаштування бази даних передаються через змінні `DATASOURCE_URL`, `DATASOURCE_USER`, `DATASOURCE_PASSWORD` та `JDBC_DRIVER`.
- **Образ фронтенду.** Запускає сервер розробки React (`npm start`), так само як додаток запускався без контейнерів. Це налаштування для розробки; у майбутньому планується збірка для production, що обслуговується через nginx.
- **Ініціалізація бази даних.** У режимі local-db `data.sql` виконується лише один раз. Щоб виконати повторний імпорт, перестворіть том.
- **Порядок запуску.** У режимі local-db бекенд очікує на перевірку стану (healthcheck) бази даних. Сам healthcheck бекенду перевіряє лише те, що порт 8080 приймає з'єднання.
- **Секрети.** Усі облікові дані беруться з `.env`. У git відстежується лише `.env.example`.