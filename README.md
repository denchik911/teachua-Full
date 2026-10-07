# Інструкція з розгортання проєкту TeachUA на різних віртуальних машинах

## 1. Загальна схема

Для розгортання проєкту необхідно створити **3 віртуальні машини**.

Рекомендовані мінімальні характеристики кожної віртуальної машини:

| Віртуальна машина | Роль                          | Мінімальні ресурси            |
| ----------------- | ----------------------------- | ----------------------------- |
| `vm-db`           | MariaDB                       | 2 vCPU / 2 GB RAM / 25 GB SSD |
| `vm-backend`      | Backend service (Java 17)     | 2 vCPU / 2 GB RAM / 25 GB SSD |
| `vm-frontend`     | Frontend service (Node.js 16) | 2 vCPU / 2 GB RAM / 25 GB SSD |

Кожна віртуальна машина відповідає за окрему складову частину проєкту:

```text
                    ┌─────────────────────┐
                    │     vm-frontend     │
                    │      Node.js 16     │
                    │      npm / React    │
                    └──────────┬──────────┘
                               │
                               │ HTTP :8080
                               ▼
                    ┌─────────────────────┐
                    │     vm-backend      │
                    │       Java 17       │
                    │    Spring Boot      │
                    └──────────┬──────────┘
                               │
                               │ MariaDB
                               ▼
                    ┌─────────────────────┐
                    │       vm-db         │
                    │       MariaDB       │
                    └─────────────────────┘
```

> **Примітка:** IP-адреси та порт MariaDB у прикладах нижче потрібно замінити на фактичні значення вашого середовища.

---

# 2. Підготовка `vm-db`

`vm-db` використовується для роботи бази даних **MariaDB**.

## 2.1. Встановлення MariaDB

На `vm-db` встановити MariaDB:

```bash
sudo apt update
sudo apt install mariadb-server mariadb-client -y
```

Перевірити статус сервісу:

```bash
sudo systemctl status mariadb
```

За необхідності увімкнути автоматичний запуск MariaDB:

```bash
sudo systemctl enable --now mariadb
```

## 2.2. Налаштування MariaDB

Необхідно створити:

* базу даних;
* користувача;
* пароль користувача;
* права доступу користувача до бази даних.

Приклад:

```sql
CREATE DATABASE teachua;

CREATE USER 'teachua'@'%' IDENTIFIED BY 'password';

GRANT ALL PRIVILEGES ON teachua.* TO 'teachua'@'%';

FLUSH PRIVILEGES;
```
## 2.3. Завантаження та імпорт початкових даних

Початкові дані для бази даних знаходяться у файлі `data.sql` в репозиторії backend:

[teachua-backend — GitHub](https://github.com/DevOps-ProjectLevel/teachua-backend)

Файл необхідно завантажити на `vm-db` та імпортувати до бази даних MariaDB.

### Крок 1. Встановити Git

Якщо Git ще не встановлений:

```bash
sudo apt update
sudo apt install git -y
```

### Крок 2. Клонувати backend-репозиторій

На `vm-db` виконати:

```bash
git clone https://github.com/DevOps-ProjectLevel/teachua-backend.git
```

Перейти до каталогу репозиторію:

```bash
cd teachua-backend
```

Перевірити наявність файлу `data.sql`:

```bash
ls -l data.sql
```

Файл повинен знаходитися за адресою:

```text
teachua-backend/data.sql
```

### Крок 3. Перевірити створення бази даних

Перед імпортом необхідно переконатися, що база даних `teachua` вже створена.

Підключитися до MariaDB:

```bash
sudo mariadb
```

Перевірити наявність бази:

```sql
SHOW DATABASES;
```

Якщо бази `teachua` немає, створити її:

```sql
CREATE DATABASE teachua;
```

Також необхідно створити користувача та надати йому права:

```sql
CREATE USER 'teachua'@'%' IDENTIFIED BY 'password';

GRANT ALL PRIVILEGES ON teachua.* TO 'teachua'@'%';

FLUSH PRIVILEGES;
```

Вийти з MariaDB:

```sql
EXIT;
```

### Крок 4. Імпортувати `data.sql`

Перебуваючи в каталозі `teachua-backend`, виконати:

```bash
mariadb -u teachua -p teachua < data.sql
```

Після виконання команди система запросить пароль користувача MariaDB:

```text
Enter password:
```

Ввести пароль, який був заданий під час створення користувача.

### Крок 5. Перевірити імпорт

Підключитися до бази даних:

```bash
mariadb -u teachua -p teachua
```

Переглянути список таблиць:

```sql
SHOW TABLES;
```

Якщо імпорт виконано успішно, MariaDB повинна показати таблиці, які були створені у `data.sql`.

Для перевірки кількості записів можна виконати запит до конкретної таблиці, наприклад:

```sql
SELECT COUNT(*) FROM table_name;
```

Вийти з MariaDB:

```sql
EXIT;
```

Після цього MariaDB повинна приймати підключення з `vm-backend`.

> **Увага:** для production-середовища замість `'%'` рекомендується вказати конкретну IP-адресу `vm-backend`.

---

# 3. Підготовка `vm-backend`

`vm-backend` відповідає за запуск backend-частини проєкту.

Backend написаний на **Java** та використовує **Maven** для збирання.

## 3.1. Встановлення Java 17 та Maven

Встановити необхідні пакети:

```bash
sudo apt update
sudo apt install openjdk-17-jdk maven -y
```

Перевірити версію Java:

```bash
java -version
```

Перевірити Maven:

```bash
mvn -version
```

Очікується Java 17 та встановлений Maven.

---

## 3.2. Отримання backend

Клонувати репозиторій:

```bash
git clone https://github.com/DevOps-ProjectLevel/teachua-backend.git
```

Перейти до каталогу проєкту:

```bash
cd teachua-backend
```

---

## 3.3. Налаштування підключення до MariaDB

Відкрити файл:

```text
teachua-backend/src/main/resources/application.properties
```

та налаштувати параметри підключення до бази даних:

```properties
spring.datasource.driver-class-name=org.mariadb.jdbc.Driver
spring.datasource.url=jdbc:mariadb://IP_DB:PORT/database
spring.datasource.username=user
spring.datasource.password=password
```

Наприклад:

```properties
spring.datasource.driver-class-name=org.mariadb.jdbc.Driver
spring.datasource.url=jdbc:mariadb://192.168.50.101:3306/teachua
spring.datasource.username=teachua
spring.datasource.password=password
```

де:

| Параметр   | Значення                      |
| ---------- | ----------------------------- |
| `IP_DB`    | IP-адреса `vm-db`             |
| `PORT`     | порт MariaDB, зазвичай `3306` |
| `database` | назва бази даних              |
| `username` | користувач MariaDB            |
| `password` | пароль користувача            |

---

## 3.4. Збірка backend

Виконати:

```bash
mvn clean package
```

Після успішного завершення збірки у каталозі `target` повинен з'явитися файл:

```text
target/dev.war
```

---

## 3.5. Запуск backend

Запустити застосунок:

```bash
java -jar target/dev.war
```

За замовчуванням backend повинен бути доступний на:

```text
http://IP_BACKEND:8080
```

Наприклад:

```text
http://192.168.50.102:8080
```

---

# 4. Підготовка `vm-frontend`

`vm-frontend` використовується для запуску frontend-частини проєкту.

Frontend використовує **Node.js** та **npm**.

## 4.1. Встановлення Git та curl

Встановити необхідні утиліти:

```bash
sudo apt update
sudo apt install curl git -y
```

---

## 4.2. Отримання frontend

Клонувати репозиторій:

```bash
git clone https://github.com/DevOps-ProjectLevel/teachua-frontend.git
```

Перейти до каталогу проєкту:

```bash
cd teachua-frontend
```

---

## 4.3. Налаштування адреси backend

Відкрити файл:

```text
teachua-frontend/src/config/ApplicationConfig.js
```

Змінити параметр `ROOT_SERVER` на адресу `vm-backend`:

```javascript
ROOT_SERVER = "http://IP_BACKEND:8080";
```

Наприклад:

```javascript
ROOT_SERVER = "http://192.168.50.102:8080";
```

Таким чином frontend буде використовувати backend, розгорнутий на `vm-backend`.

---

## 4.4. Встановлення Node.js та npm

Для встановлення Node.js 16 виконати:

```bash
curl -sL https://deb.nodesource.com/setup_16.x | sudo -E bash -
```

Після цього:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install nodejs -y
```

Перевірити встановлені версії:

```bash
node -v
npm -v
```

---

## 4.5. Встановлення залежностей

Перейти до каталогу frontend:

```bash
cd teachua-frontend
```

Встановити залежності:

```bash
npm install
```

---

## 4.6. Запуск frontend

Запустити застосунок:

```bash
npm start
```

Після запуску frontend буде доступний за адресою, яку вказує консоль Node.js.

---

# 5. Порядок запуску всіх компонентів

Для коректної роботи проєкту рекомендується запускати компоненти в такому порядку:

### Крок 1 — Database

На `vm-db`:

```bash
sudo systemctl start mariadb
```

Переконатися, що MariaDB працює та доступна з `vm-backend`.

### Крок 2 — Backend

На `vm-backend`:

```bash
cd teachua-backend
mvn clean package
java -jar target/dev.war
```

Переконатися, що backend успішно підключився до MariaDB.

### Крок 3 — Frontend

На `vm-frontend`:

```bash
cd teachua-frontend
npm install
npm start
```

Frontend повинен використовувати адресу backend, вказану в:

```text
src/config/ApplicationConfig.js
```

---

# 6. Перевірка мережевої доступності

Перед запуском необхідно переконатися, що віртуальні машини можуть взаємодіяти між собою.

Необхідні з'єднання:

```text
vm-frontend ──────► vm-backend:8080
                         │
                         ▼
                    vm-db:3306
```

З `vm-backend` перевірити доступність MariaDB:

```bash
nc -zv IP_DB 3306
```

З `vm-frontend` перевірити доступність backend:

```bash
curl http://IP_BACKEND:8080
```

Якщо використовується firewall, необхідно дозволити відповідні порти.

---

# 7. Репозиторії проєкту

### Frontend

[teachua-frontend — GitHub](https://github.com/DevOps-ProjectLevel/teachua-frontend)

### Backend

[teachua-backend — GitHub](https://github.com/DevOps-ProjectLevel/teachua-backend)

---

# 8. Підсумкова структура

Після завершення розгортання повинна бути наступна структура:

| VM            | Компонент |                     Порт | Основне ПЗ      |
| ------------- | --------- | -----------------------: | --------------- |
| `vm-db`       | MariaDB   |                   `3306` | MariaDB         |
| `vm-backend`  | Backend   |                   `8080` | Java 17, Maven  |
| `vm-frontend` | Frontend  | залежить від `npm start` | Node.js 16, npm |

Основний принцип взаємодії:

```text
┌──────────────────┐
│   VM-FRONTEND    │
│                  │
│   Node.js 16     │
│      npm         │
└────────┬─────────┘
         │
         │ HTTP
         │ :8080
         ▼
┌──────────────────┐
│    VM-BACKEND    │
│                  │
│     Java 17      │
│      Maven       │
└────────┬─────────┘
         │
         │ MariaDB
         │ :3306
         ▼
┌──────────────────┐
│      VM-DB       │
│                  │
│     MariaDB      │
└──────────────────┘
```
