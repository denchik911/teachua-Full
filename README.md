# Моніторинг TeachUA у K3s

Гілка `Monitoring` підготовлена на основі `Kubernetes`, коміт `68a0b5b`. Вона зберігає застосунок, Kubernetes-маніфести й наявний CI/CD, додаючи налаштування та документацію моніторингу.

## Компоненти

| Компонент | Призначення |
|---|---|
| Prometheus | Збір і зберігання метрик у часі |
| Grafana | Дашборди й графіки |
| Alertmanager | Обробка сповіщень; зовнішній отримувач ще не налаштований |
| Prometheus Operator | Керування Prometheus та пов'язаними ресурсами Kubernetes |
| kube-state-metrics | Метрики стану ресурсів Kubernetes |
| node-exporter | Метрики Linux-вузла |

Системні метрики контейнерів збираються також із kubelet. Це моніторинг інфраструктури: окреме експортування метрик HTTP-запитів Java-застосунку, трасування та централізовані логи ще не налаштовані.

## Середовище й налаштування

Debian 12, K3s `v1.37.2+k3s1`, один вузол `192.168.50.139`, 4 vCPU, 8 ГБ RAM. Перед установленням: близько 5.3Gi доступної RAM за `free`, 32 ГБ диска. Це початкові значення, не вимірювання після встановлення.

`values.yaml` містить лише перевизначення стандартного chart:

- історія Prometheus до 2 діб, цільовий розмір блоків 3GB, PVC 5Gi;
- постійне сховище Grafana 1Gi;
- requests/limits для основних контейнерів Prometheus, Grafana, Alertmanager;
- вимкнений стандартний збір метрик etcd, controller-manager, scheduler і kube-proxy та відповідні групи правил для нашого K3s. Це не вимикає самі компоненти Kubernetes;
- інтерфейси залишені всередині кластера, без NodePort/Ingress.

Додаткові контейнери й exporter також споживають ресурси. Local-path не гарантує жорсткої квоти; retentionSize не є лімітом усього каталогу Prometheus. Alertmanager без окремого PVC: його локальний стан не гарантовано переживе перестворення pod. Prometheus і Grafana зберігають дані на диску єдиної VM — це не резервна копія.

## Відтворення

Виконуйте на окремій навчальній Debian VM з готовим K3s. Для існуючої установки не запускайте повторний install. Усі наступні команди Helm використовують sudo: репозиторій chart також потрібно додати для root.

### Helm

Якщо Helm ще не встановлений:

```bash
mkdir -p ~/labs/monitoring
cd ~/labs/monitoring
curl -fsSL -o get-helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
bash get-helm.sh
helm version --short
```

Інсталятор отримує актуальний Helm 4; перевірений тут реліз — 4.3.1. Скрипт установлює клієнт на VM, а не Prometheus.

### Chart і перевірка

Поверніться до кореня цього репозиторію:

```bash
cd ~/labs/teachua-k8s
sudo helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
sudo helm repo update
sudo helm template monitoring prometheus-community/kube-prometheus-stack \
  --version 93.2.0 --namespace monitoring \
  --values monitoring/values.yaml > /dev/null
```

`template` перевіряє генерацію маніфестів, не створюючи їх у кластері. Версія chart зафіксована явно.

### Установлення

```bash
sudo helm install monitoring prometheus-community/kube-prometheus-stack \
  --version 93.2.0 --namespace monitoring --create-namespace \
  --values monitoring/values.yaml \
  --kubeconfig /etc/rancher/k3s/k3s.yaml --wait --timeout 10m
sudo k3s kubectl get pods,pvc -n monitoring
```

Chart створює також кластерні CRD й правила доступу, а не лише ресурси namespace. TeachUA та БД цим install не оновлюються. При timeout спочатку перевірте стан і події, не запускайте повторний install.

## Доступ до Grafana

На Debian у першому терміналі:

```bash
sudo k3s kubectl port-forward -n monitoring service/monitoring-grafana 3001:80
```

На Windows у PowerShell:

```powershell
ssh -N -L 3001:127.0.0.1:3001 manager@192.168.50.139
```

Відкрийте `http://localhost:3001`. Обидва процеси повинні працювати. Ctrl+C закриває відповідний тунель, не зупиняючи моніторинг.

На Debian в іншому сеансі отримайте пароль лише для особистого входу:

```bash
sudo k3s kubectl get secret monitoring-grafana -n monitoring \
  -o jsonpath='{.data.admin-password}' | base64 --decode
echo
```
## Дашборд і сценарій презентації

1. Dashboards → `Kubernetes / Compute Resources / Namespace (Pods)`.
2. Вибрати Prometheus як джерело, namespace `teachua`; cluster — доступне значення, якщо фільтр є.
3. Період Last 15 minutes, оновлення 30s.
4. Показати pod frontend, backend і db та пояснити графіки.

| Показник | Значення |
|---|---|
| CPU Usage | Використані ядра: 0.1 = десята частина ядра |
| Memory / Working Set | Оцінка активно використовуваної пам'яті, не розмір образу на диску |
| Requests | Запит ресурсів, який враховує планувальник Kubernetes |
| Limits | Межі ресурсів: CPU може обмежуватися, перевищення пам'яті може завершити контейнер |
| Network | Трафік; конкретні одиниці дивіться в назві та легенді панелі |

## Діагностика

```bash
sudo helm status monitoring -n monitoring --kubeconfig /etc/rancher/k3s/k3s.yaml
sudo k3s kubectl get pods,pvc,service -n monitoring
sudo k3s kubectl get events -n monitoring --sort-by=.metadata.creationTimestamp
sudo k3s kubectl top pods -n monitoring
sudo k3s kubectl top nodes
free -h
df -h /
```

При відсутності графіків перевірте фільтри, період, джерело Prometheus і готовність pod. Дані про CPU/RAM від `free` та Kubernetes можуть відрізнятись через різний облік кешу й робочої пам'яті.

Документація: [chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack), [каталог chart](https://prometheus-community.github.io/helm-charts/), [Helm](https://helm.sh/docs/intro/install/).
