# TeachUA — Kubernetes (K3s)

Гілка `Kubernetes` створена на основі `docker`. TeachUA розгорнуто локально на окремій VMware VM: React frontend, Java 17 / Spring Boot backend і MariaDB 10.11. Dockerfile залишені без змін; Docker використовується для збірки, containerd K3s — для запуску.

## Перевірений результат

За результатами виконання користувачем 11 листопада 2026 року:

- Debian 12, x86_64, 4 vCPU, 8 ГБ RAM, диск 48 ГБ; K3s `v1.37.2+k3s1`.
- Вузол Ready; frontend, backend і db — `1/1 Running`, без перезапусків на момент перевірки.
- PVC `db-data` — Bound; початкові таблиці імпортовані, користувач БД підключається.
- Backend успішно підключився до MariaDB; контекст API `/dev`.
- Сторінка відкривається з Windows, дані завантажуються.
- В окремому тесті Nginx підтверджено перестворення pod після видалення. Відмовостійкість VM цим не перевірялась.

## Архітектура

```text
Браузер ── VM:30300 ── Service frontend ── React :3000
   └────── VM:30080 ── Service backend ─── Spring :8080 /dev
                                              │
                                         Service db:3306
                                              │
                                         MariaDB ── PVC db-data
```

Усе працює в namespace `teachua` на одному вузлі. БД не має зовнішнього порту. Deployment підтримує бажану кількість pod, Service надає стабільну мережеву адресу, PVC зберігає дані окремо від pod. Паролі беруться із Secret, SQL — із ConfigMap.

## Файли

| Файл | Призначення |
|---|---|
| `k8s/db-storage.yaml` | PVC 5Gi, local-path |
| `k8s/db.yaml` | MariaDB, перевірки готовності, внутрішній Service |
| `k8s/backend.yaml` | Backend, ресурси, probes, NodePort 30080 |
| `k8s/frontend.yaml` | Frontend, адреса API, ресурси, NodePort 30300 |
| `README.docker.md` | Попередня інструкція Docker Compose |

ConfigMap генерується з наявного `teachua-backend/data.sql`. Secret створюється на VM й у Git не зберігається. Архіви образів та дані БД також не комітяться.

## Відтворення на чистій навчальній VM

Команди нижче виконуються в Bash на окремій Debian VM із sudo та інтернетом. Docker і K3s потрібні обидва; Compose для цього запуску не використовується. Не повторюйте підготовку Secret і namespace на вже працюючому кластері.

### 1. Інструменти та код

```bash
sudo apt update
sudo apt install -y curl ca-certificates git docker.io
sudo systemctl enable --now docker
curl -fL https://get.k3s.io -o /tmp/install-k3s.sh
sudo sh /tmp/install-k3s.sh
sudo k3s kubectl get nodes -o wide
git clone --branch Kubernetes --single-branch https://github.com/denchik911/teachua-Full.git ~/labs/teachua-k8s
cd ~/labs/teachua-k8s
```

Інсталятор встановлює поточну stable-версію K3s, яка може відрізнятися від перевіреної. Дочекайтеся Ready; firewall не вимикайте автоматично. Порти 30300/30080 мають бути доступні з навчальної мережі, не з публічного інтернету.

### 2. Образи

```bash
sudo docker build -t teachua-backend:demo1 ./teachua-backend
sudo docker build -t teachua-frontend:demo1 ./teachua-frontend
sudo docker save -o /tmp/teachua-backend-demo1.tar teachua-backend:demo1
sudo docker save -o /tmp/teachua-frontend-demo1.tar teachua-frontend:demo1
sudo k3s ctr images import /tmp/teachua-backend-demo1.tar
sudo k3s ctr images import /tmp/teachua-frontend-demo1.tar
sudo k3s ctr images list | grep teachua
```

Docker і K3s мають окремі сховища образів. `imagePullPolicy: Never` вимагає імпорту на кожний вузол, де запускатиметься застосунок. Для цього демо вузол один.

### 3. Namespace, SQL і паролі

```bash
sudo k3s kubectl create namespace teachua
sudo k3s kubectl create configmap db-init --namespace=teachua \
  --from-file=01-data.sql=teachua-backend/data.sql
read -rsp 'Пароль користувача teachua: ' DB_PASSWORD; echo
read -rsp 'Окремий пароль root MariaDB: ' DB_ROOT_PASSWORD; echo
(
  set -o pipefail
  if [ -z "$DB_PASSWORD" ] || [ -z "$DB_ROOT_PASSWORD" ]; then
    echo 'Паролі порожні — Secret не створено.'
    exit 1
  fi
  printf '%s\n' "DB_PASSWORD=$DB_PASSWORD" "DB_ROOT_PASSWORD=$DB_ROOT_PASSWORD" |
    sudo k3s kubectl create secret generic db-credentials \
      --namespace=teachua --from-env-file=/dev/stdin
)
unset DB_PASSWORD DB_ROOT_PASSWORD
sudo k3s kubectl get secret db-credentials --namespace=teachua
```

Продовжуйте лише після успішного створення Secret (`DATA 2`). Використовуйте окремі навчальні паролі. Не публікуйте Secret, kubeconfig, `.env` або токени. ConfigMap призначений лише для демонстраційного SQL без приватних даних. Secret сам по собі не гарантує шифрування на диску.

### 4. MariaDB

```bash
sudo k3s kubectl apply -f k8s/db-storage.yaml
sudo k3s kubectl apply -f k8s/db.yaml
sudo k3s kubectl rollout status deployment/db -n teachua --timeout=300s
sudo k3s kubectl get pods,pvc,service -n teachua
sudo k3s kubectl exec -it deployment/db -n teachua -- mariadb -u teachua -p teachua
```

У MariaDB: `SHOW TABLES;`, потім `exit;`. SQL виконується лише при першому запуску з порожнім каталогом даних. Зміна Secret не змінює пароль уже ініціалізованої БД. Не видаляйте PVC для виправлення помилок.

### 5. Backend і frontend

Перед запуском змініть `REACT_APP_ROOT_SERVER` у `k8s/frontend.yaml`: замість `192.168.50.139` поставте IP своєї VM, доступний із браузера. Залиште порт `30080`, без `/dev` — frontend додає цей префікс сам.

```bash
sudo k3s kubectl apply -f k8s/backend.yaml
sudo k3s kubectl rollout status deployment/backend -n teachua --timeout=300s
sudo k3s kubectl logs deployment/backend -n teachua --tail=60
sudo k3s kubectl apply -f k8s/frontend.yaml
sudo k3s kubectl rollout status deployment/frontend -n teachua --timeout=600s
sudo k3s kubectl get pods,service -n teachua
```

Відкрийте `http://<IP-VM>:30300` з Windows і перевірте завантаження даних. Браузер викликає backend напряму через `http://<IP-VM>:30080/dev`. HTTP 403 на `/dev/` підтверджує відповідь сервера, але не роботу конкретного API.

## Діагностика

```bash
sudo k3s kubectl get pods,pvc,service -n teachua
sudo k3s kubectl logs deployment/backend -n teachua --tail=60
sudo k3s kubectl logs deployment/frontend -n teachua --tail=60
sudo k3s kubectl get events -n teachua --sort-by=.metadata.creationTimestamp
sudo k3s kubectl top nodes
sudo k3s kubectl top pods -n teachua
```

`ErrImageNeverPull` — перевірте імпорт і тег образу. Помилки браузера діагностуйте через F12 → Network/Console: адресу API, HTTP-статус і CORS. Метрики потребують готового metrics-server. Логи можуть містити чутливі значення — перевіряйте перед публікацією.

## Обмеження і наступні кроки

- Один вузол: при падінні VM зупиняється весь проєкт. Local-path не є резервною копією; 5Gi не гарантує жорсткої квоти.
- Frontend використовує Node 16 і dev-сервер. Під час збірки npm повідомив про несумісну вимогу Node однієї залежності та 193 вразливості; автоматичні виправлення не запускались.
- Тести backend пропущені Dockerfile через `-DskipTests`. TCP probes перевіряють порт, а не бізнес-функції API. Liveness probes ще не додані.
- TLS, Ingress для застосунку, production-збірка frontend, registry та автоматичне CD ще не налаштовані.
- Prometheus/Grafana й хмарне розгортання — наступні етапи. Floci планується як локальна емуляція AWS; це не міграція в реальну хмару.

## Зупинка й очищення

Для тимчасової зупинки зі збереженням PVC та даних:

```bash
sudo k3s kubectl scale deployment/frontend deployment/backend deployment/db -n teachua --replicas=0
```

Для запуску встановіть `--replicas=1`; спочатку дочекайтеся готовності БД, потім застосунку.

**Руйнівна операція:** наступна команда видаляє весь namespace `teachua`, включно із Secret, ConfigMap, застосунками й PVC. Для local-path це може видалити і самі дані MariaDB. Виконуйте лише після резервного копіювання потрібних даних та перевірки цільового кластера:

```bash
sudo k3s kubectl delete namespace teachua
```

Образи й TAR-архіви цією командою не видаляються. Системні компоненти K3s та окремий тестовий namespace `demo` залишаються.
