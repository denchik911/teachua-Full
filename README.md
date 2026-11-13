# TeachUA — навчальний DevOps-проєкт

**TeachUA** — вебзастосунок для пошуку дитячих гуртків з українською мовою навчання в різних містах. Застосунок підтримує реєстрацію та авторизацію користувачів.

Цей репозиторій демонструє роботу з розгортанням готового застосунку: ручний запуск на віртуальних машинах, контейнеризацію, автоматизацію серверів, Kubernetes, CI/CD та моніторинг.

## Компоненти застосунку

| Компонент | Технології | Призначення |
|---|---|---|
| Frontend | React, JavaScript | Інтерфейс користувача та звернення до API |
| Backend | Java 17, Spring Boot, Maven | API, бізнес-логіка та доступ до БД |
| Database | MariaDB / MySQL | Зберігання даних застосунку |

Браузер завантажує frontend і надсилає запити до backend API. Backend читає та змінює дані в базі. Конкретні адреси й порти залежать від варіанта розгортання.

## Навігація за гілками

Кожна гілка містить окремий етап роботи. Інструкції запуску та особливості відповідного середовища наведені в README цієї гілки.

| Гілка | Частина проєкту | Що містить |
|---|---|---|
| [main](https://github.com/denchik911/teachua-Full/tree/main) | Огляд | Код застосунку та навігація між етапами |
| [manual](https://github.com/denchik911/teachua-Full/tree/manual) | Ручне розгортання | Налаштування окремих VM для frontend, backend і MariaDB, встановлення залежностей та перевірка взаємодії |
| [docker](https://github.com/denchik911/teachua-Full/tree/docker) | Контейнеризація | Dockerfile frontend/backend і Docker Compose для запуску із зовнішньою БД або контейнером MariaDB |
| [nginx](https://github.com/denchik911/teachua-Full/tree/nginx) | Балансування навантаження | nginx перед двома backend, алгоритм least_conn та опис перевірок балансування |
| [ansible](https://github.com/denchik911/teachua-Full/tree/ansible) | Автоматизація серверів | Inventory, playbook і ролі для пакетів, firewall, MariaDB, backend та frontend |
| [ci](https://github.com/denchik911/teachua-Full/tree/ci) | Початковий CI | Jenkinsfile для автоматизації збірки застосунку |
| [Kubernetes](https://github.com/denchik911/teachua-Full/tree/Kubernetes) | Оркестрація та CI/CD | Маніфести K3s, Deployment, Service, PVC, Jenkinsfile і скрипт доставки та оновлення образів |
| [Monitoring](https://github.com/denchik911/teachua-Full/tree/Monitoring) | Моніторинг Kubernetes | Розвиток гілки Kubernetes: Prometheus, Grafana, Helm values та інструкції перегляду метрик TeachUA |

Рекомендована послідовність ознайомлення: **manual → docker → nginx → ansible → ci → Kubernetes → Monitoring**.


## Запуск і секрети

Універсальної команди запуску для всіх гілок немає: оберіть потрібний етап і дотримуйтесь його README. Не додавайте в Git `.env`, паролі, приватні SSH-ключі, kubeconfig або експорт Kubernetes Secret. Використовуйте лише навчальні дані й окремі облікові записи.
