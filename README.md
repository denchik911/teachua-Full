# TeachUA — Continuous Integration у Jenkins

Гілка [`ci`](https://github.com/denchik911/teachua-Full/tree/ci) містить початковий pipeline автоматичної збірки TeachUA. Jenkins отримує код, збирає Java backend і React frontend у Docker-контейнерах та зберігає WAR backend як артефакт збірки.

**CI (Continuous Integration)** — автоматична збірка та перевірки коду. Ця гілка не розгортає застосунок на серверах або в Kubernetes. Подальша реалізація доставки й розгортання знаходиться в [`Kubernetes`](https://github.com/denchik911/teachua-Full/tree/Kubernetes).

## Застосунок

TeachUA дозволяє шукати дитячі гуртки з українською мовою навчання. Компоненти: React frontend, Java 17 / Spring Boot backend та MariaDB/MySQL. Цей pipeline збирає код; окрему БД і працюючий сайт він не запускає.

## Файли

| Шлях | Призначення |
|---|---|
| [`Jenkinsfile`](https://github.com/denchik911/teachua-Full/blob/ci/Jenkinsfile) | Declarative Pipeline Jenkins |
| `teachua-backend/pom.xml` | Залежності, Java 17, Maven-збірка WAR із назвою dev |
| `teachua-frontend/package.json` | Залежності React і команди npm |
| `teachua-frontend/package-lock.json` | Зафіксовані залежності для npm ci |

## Як працює pipeline

```text
GitHub, гілка ci
       │
       ▼
Jenkins: отримання коду
       │
       ▼
Info: гілка та доступ до Docker
       │
       ▼
Backend: Maven + Java 17 → WAR → Jenkins Artifacts
       │
       ▼
Frontend: Node 16 → npm ci → npm run build
       │
       ▼
CI passed або CI failed
```

### Info

Показує `BRANCH_NAME` та перевіряє Docker Server. `BRANCH_NAME` зазвичай надає Multibranch Pipeline; у звичайному Pipeline це поле може бути порожнім, що саме по собі не означає помилку checkout.

### Backend: build

Етап використовує Docker-образ `maven:3.9-eclipse-temurin-17` і виконує в `teachua-backend`:

```bash
mvn -B -Dmaven.repo.local=.m2/repository clean package
```

- `-B` — неінтерактивний режим Maven.
- `maven.repo.local` — локальний кеш залежностей у робочому каталозі backend.
- `clean package` — очищення попереднього результату та проходження Maven lifecycle до пакування, включно з фазою test. Прапорця пропуску тестів у цій команді немає; це відрізняється від Dockerfile пізнішого Kubernetes-етапу.

Після успіху Jenkins архівує `teachua-backend/target/*.war` із `fingerprint: true`. Очікуваний файл — `dev.war`. Fingerprint допомагає відстежувати конкретний файл у Jenkins. Архівація виконується одразу після успішного backend-етапу, навіть якщо пізніше frontend завершиться помилкою.

### Frontend: build

Етап використовує Docker-образ `node:16`. У робочій копії Jenkins він замінює закоментований рядок оголошення `ROOT_SERVER` на читання `process.env.REACT_APP_ROOT_SERVER`, потім виконує:

```bash
npm ci --cache .npm-cache
npm run build
```

`npm ci` встановлює залежності за lock-файлом. `npm run build` запускає `craco build` і створює статичні файли в `teachua-frontend/build/`. Окремого `npm test` у pipeline немає. Заміна через `sed` змінює лише workspace Jenkins і не створює коміт у GitHub.

## Чому в Jenkins завантажується лише backend

У Jenkinsfile `archiveArtifacts` налаштований **лише для WAR backend**. Frontend також збирається, але каталог `build/` не архівується. Він залишається у workspace та може бути перезаписаний або видалений під час наступних запусків/очищення.

Це два різні результати: backend — виконуваний WAR, frontend — каталог статичних HTML/CSS/JS-файлів. Відсутність frontend у розділі Artifacts не доводить, що його збірка не виконувалася: перевіряйте етап `Frontend: build` у Console Output.

## Передумови

- Jenkins із підтримкою Pipeline, Git і Docker Pipeline.
- Linux-агент із Git, Docker CLI та доступом до запущеного Docker Engine.
- Доступ агента до GitHub, реєстру образів, Maven- і npm-репозиторіїв.
- Достатні CPU, RAM і вільний диск для збірки Java/React та кешів. Ресурси залежать від середовища, жорсткий ліміт пам'яті контейнерів у Jenkinsfile не встановлений.

`agent any` дозволяє Jenkins обрати доступний агент, тому він має відповідати цим вимогам. Docker-етапи використовують `reuseNode true`, щоб працювати з тим самим workspace. Java й Node усередині збірки беруться з образів, але самому Jenkins потрібне власне сумісне Java-середовище.

Доступ користувача Jenkins до Docker надавайте лише на навчальному або довіреному агенті: керування Docker дає широкі можливості на хості. Не використовуйте `chmod 666 /var/run/docker.sock` як виправлення доступу.

## Налаштування завдання Jenkins

1. Створіть **New Item → Pipeline**, наприклад `teachua-ci`.
2. У **Pipeline → Definition** виберіть **Pipeline script from SCM**.
3. Виберіть **SCM → Git**.
4. Repository URL: `https://github.com/denchik911/teachua-Full.git`.
5. Branch Specifier: `*/ci`.
6. Script Path: `Jenkinsfile`.
7. Збережіть і натисніть **Build Now**.

Для читання публічного репозиторію credentials зазвичай не потрібні. Якщо у вашому середовищі потрібна авторизація, зберігайте її в Jenkins Credentials.

Webhook або опитування Git потрібно налаштовувати окремо: Jenkinsfile не містить `triggers`. Без них збірка запускається вручну.

## Налаштування збірки

| Параметр | Значення | Призначення |
|---|---|---|
| `MAVEN_OPTS` | `-Xmx768m` | Максимальна heap-пам'ять процесу Maven |
| `NODE_OPTIONS` | `--max_old_space_size=1536` | Межа old-space heap Node у MiB |
| `CI` | `false` | React build не підвищує попередження до помилок лише через CI-режим |
| `REACT_APP_ROOT_SERVER` | `http://localhost:8080` | Тимчасова адреса API для CI-збірки |

Ці параметри heap не обмежують загальну пам'ять усіх процесів контейнера. Адреса API потрапляє в статичну React-збірку. Для реального розгортання frontend потрібно збирати з адресою backend, доступною з браузера; localhost у браузері вказуватиме на комп'ютер користувача.

Pipeline додає часові мітки, забороняє паралельні запуски цього завдання, зберігає до 10 збірок Jenkins і обмежує запуск 30 хвилинами. Політика історії збірок не є очищенням усіх Docker-образів та workspace-кешів.

## Перевірка результату

- Усі етапи мають завершитися успішно, фінальний статус — `SUCCESS`.
- У Console Output має бути `CI passed`. Повідомлення `CI failed` означає, що треба знайти першу змістовну помилку вище в логу.
- WAR доступний у розділі **Artifacts** відповідної збірки.
- Для frontend перевірте завершення `npm run build` і наявність `teachua-frontend/build/` у workspace.

Успіх CI підтверджує проходження поточних команд збірки, а не працездатність розгорнутого сайту чи повне покриття тестами.

## Типові проблеми

| Симптом | Що перевірити |
|---|---|
| `docker: not found` / Docker daemon недоступний | Docker CLI, службу Engine та обраний агент |
| `Permission denied` для Docker socket | Права користувача, під яким працює агент Jenkins |
| Помилка `agent docker` або невідомий тип агента | Наявність і завантаження плагіна Docker Pipeline |
| Maven завершується помилкою | Першу помилку компіляції/тестів, завантаження залежностей і права workspace |
| `npm ci` відхиляє залежності | Узгодженість package.json і package-lock.json |
| React build завершується `Killed` | Пам'ять агента та журнал OOM; heap-параметр не гарантує достатність RAM |
| `ROOT_SERVER is not defined` | Фактичний рядок ApplicationConfig.js і відповідність шаблону sed |
| Немає frontend в Artifacts | Це поточна поведінка: каталог build не архівується |

## Межі цього етапу

Гілка не створює образи застосунку, не публікує їх у registry й не виконує deployment. Немає окремого frontend-тестування, публікації JUnit-звітів, SonarQube або сканування залежностей. Node 16 і старі frontend-залежності потребують окремого оновлення та перевірки сумісності.

