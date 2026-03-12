# Infrastructure Monitoring Stack

Этот проект представляет собой модульную, готовую к развертыванию систему мониторинга на основе Docker Compose. Он предназначен для сбора метрик с хост-системы и их визуализации с помощью стека Prometheus + Grafana.

## Особенности архитектуры

Проект использует директиву `include` в Docker Compose. Основной файл `docker-compose.yml` в корне проекта не описывает сами сервисы, а лишь подключает их из отдельных файлов, расположенных в директории `compose/`. Это позволяет гибко управлять составом стека и упрощает его поддержку.

Кроме того, в проекте определены две сети:
*   `core-net`: Внутренняя сеть для связи сервисов мониторинга между собой.
*   `chome-net`: Внешняя сеть, которая должна существовать заранее. Она позволяет этому стеку мониторинга "видеть" другие Docker-контейнеры и собирать с них метрики.

## Состав стека

| Сервис | Файл конфигурации | Описание |
| :--- | :--- | :--- |
| **Prometheus** | `compose/prometheus/docker-compose.yml` | Собирает и хранит метрики. |
| **Node Exporter** | `compose/node-exporter/docker-compose.yml` | Собирает метрики хост-машины (CPU, RAM, диск). Работает в режиме host network. |
| **cAdvisor** | `compose/cadvisor/docker-compose.yml` | Экспортирует метрики Docker-контейнеров для Prometheus (внутренний сервис без внешнего порта). |
| **Grafana** | `compose/grafana/docker-compose.yml` | Визуализация метрик и дашборды. |
| **Portainer CE** | `compose/portainer/docker-compose.yml` | Веб-управление Docker (опционально, но включено). |

## Предварительные требования

Перед запуском необходимо выполнить **три шага** на хост-машине:

#### 1. Установить Docker и Docker Compose

Убедитесь, что у вас установлены актуальные версии.

#### 2. Создать внешнюю Docker-сеть

Сервисы подключаются к внешней сети `chome-net`. Ее нужно создать вручную **до** первого запуска:
```bash
docker network create chome-net
```

#### 3. Создать конфигурацию для Prometheus

Стек ожидает, что конфигурационный файл `prometheus.yml` будет находиться по пути `/mnt/data/projects/infra/volumes/prometheus/`.

Выполните следующие команды для создания директории и базового файла конфигурации:

```bash
# Создаем директорию рекурсивно
sudo mkdir -p /mnt/data/projects/infra/volumes/prometheus

# Создаем файл prometheus.yml
sudo nano /mnt/data/projects/infra/volumes/prometheus/prometheus.yml
```

Вставьте в этот файл минимальную конфигурацию, которая будет собирать метрики с `node-exporter`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['host.docker.internal:9100']
    relabel_configs:
      - source_labels: []
        target_label: instance
        replacement: 'odin'

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
    relabel_configs:
      - source_labels: []
        target_label: instance
        replacement: 'odin'
```

## Развертывание

1.  **Клонируйте репозиторий:**
    ```bash
    git clone https://github.com/Snejock/local.infra.git
    cd local.infra
    ```

2.  **Запустите стек:**
    Находясь в корневой директории проекта (где лежит основной `docker-compose.yml`), выполните команду:
    ```bash
    docker compose up -d
    ```
    Docker Compose прочитает корневой файл, подключит все сервисы из поддиректорий и запустит их в фоновом режиме.

3.  **Проверьте статус контейнеров:**
    ```bash
    docker compose ps
    ```
    Все сервисы (Prometheus, Grafana, Node Exporter, cAdvisor, Portainer) должны иметь статус `Up` или `running`.

## Доступ к сервисам

Веб-интерфейсы будут доступны по следующим портам вашего сервера:

| Сервис | Адрес |
| :--- | :--- |
| **Prometheus** | `http://<IP-адрес-сервера>:39090` |
| **Grafana** | `http://<IP-адрес-сервера>:33000` |
| **Portainer** | `https://<IP-адрес-сервера>:39443` (или Edge Agent: `http://<IP-адрес-сервера>:38000`) |
| **Node Exporter** | `http://<IP-адрес-сервера>:9100/metrics` (host network, без проброса порта) |
| **cAdvisor** | без внешнего порта (используется Prometheus по адресу `cadvisor:8080` внутри сети) |

## Первоначальная настройка Grafana

При первом входе в Grafana потребуется быстрая настройка.

1.  **Войдите в Grafana:**
    *   Откройте `http://<IP-адрес-сервера>:33000`.
    *   Логин по умолчанию: `admin`
    *   Пароль по умолчанию: `admin`
    *   Grafana попросит вас сменить пароль.

2.  **Добавьте Prometheus как источник данных (Data Source):**
    *   Перейдите в `Configuration` (значок шестеренки) -> `Data Sources`.
    *   Нажмите `Add data source` и выберите `Prometheus`.
    *   В поле **URL** введите `http://prometheus:9090`. (Используйте имя сервиса, так как Grafana и Prometheus находятся в одной Docker-сети `core-net`).
    *   Нажмите `Save & test`. Должно появиться сообщение "Data source is working".

3.  **Импортируйте готовый дашборд для Node Exporter:**
    *   Перейдите в `Dashboards` (значок четырех квадратов) -> `Import`.
    *   В поле "Import via grafana.com" введите ID популярного дашборда, например: `1860` (Node Exporter Full).
    *   Нажмите `Load`.
    *   На следующем шаге внизу выберите ваш источник данных Prometheus и нажмите `Import`.

Теперь у вас есть полнофункциональный дашборд с подробными метриками вашего сервера.

## Управление стеком

*   **Остановка стека:**
    ```bash
    docker compose down
    ```
*   **Просмотр логов (например, Prometheus):**
    ```bash
    docker compose logs -f prometheus
    ```
