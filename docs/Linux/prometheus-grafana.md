# Установка и настройка Prometheus и Grafana (Docker)

!!! info "Что входит в эту инструкцию?"
- Установка Prometheus
- Установка Grafana
- Настройка Prometheus
- Подключение Grafana к Prometheus
- Импорт готовых dashboards
- Автозапуск и хранение данных

## 1. Предварительные требования

Перед началом убедитесь, что установлены:

- Docker
- Docker Compose Plugin

Проверка:
```sh
docker --version
docker compose version
```

Если нет — установи Docker (у меня есть готовая инструкция, могу вставить).

## 2. Создание структуры каталогов
```shell
mkdir -p /opt/monitoring/prometheus
mkdir -p /opt/monitoring/grafana
```
## 3. Создание prometheus.yml

Создай файл:
```sh
nano /opt/monitoring/prometheus/prometheus.yml
```

Вставь:
``` yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["prometheus:9090"]
```

!!! info "Простой конфиг"
Этот конфиг пока собирает метрики только самого Prometheus.
Позже можно добавить Node Exporter, впн, микротики, докер контейнеры и т.д.

## 4. Docker Compose файл

Создай:
``` bash
nano /opt/monitoring/docker-compose.yml
```

Вставь:
``` yaml 
version: "3.8"

services:
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    restart: unless-stopped

  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
```
## 5. Запуск Prometheus + Grafana

Перейди в директорию:
``` bash
cd /opt/monitoring
```

Запуск:
``` bash
docker compose up -d
```

Проверка:
```bash
docker ps
```
## 6. Доступ к сервисам
**Prometheus:**

http://SERVER_IP:9090

**Grafana:**

http://SERVER_IP:3000

!!! tip "Логин по умолчанию"

- Login: admin
- Password: admin

После входа система попросит изменить пароль.

## 7. Подключение Prometheus к Grafana

Зайди в Grafana → Connections → Data sources

Нажми Add data source

Выбери Prometheus

В поле URL укажи:

http://prometheus:9090


Нажми Save & test

Должно показать "Data source is working"

## 8. Импорт готовых Dashboard'ов

В Grafana:

Dashboards → Import

Вставь ID — например:

Полезные Dashboards:

| Dashboard | ID |
|----------|----|
| Node Exporter Full | 1860 |
| Docker Container Metrics | 193 |
| Prometheus 2.0 Stats | 3662 |
| Grafana Overview | 3590 | 

Нажми Load → выбери datasource Prometheus → Import.

## 9. Добавление Node Exporter (Linux-сервер)

!!! info "Node Exporter — сбор метрик CPU, RAM, дисков, сети"

Установка:
```sh
docker run -d \
  --name=node_exporter \
  -p 9100:9100 \
  --restart unless-stopped \
  prom/node-exporter
```

Добавь в prometheus.yml:
```yaml
  - job_name: "node"
    static_configs:
      - targets: ["SERVER_IP:9100"]
```

Перезапусти Prometheus:
```sh
docker compose restart prometheus
```
## 10. Авторезервное хранение данных (volumes)

Все данные сохраняются в Docker volumes:

Prometheus → prometheus_data

Grafana → grafana_data

Посмотреть:
```bash
docker volume ls
```

Резервная копия:
```bash
docker run --rm -v prometheus_data:/data -v $(pwd):/backup ubuntu \
  tar czf /backup/prometheus_backup.tar.gz /data

docker run --rm -v grafana_data:/data -v $(pwd):/backup ubuntu \
  tar czf /backup/grafana_backup.tar.gz /data
```
## 11. Перезапуск и управление
``` bash
docker compose up -d
docker compose down
docker compose restart prometheus
docker compose logs -f grafana
```
## 12. Добавление микротиков, Docker-контейнеров, VPN и др.

Если нужно — могу подготовить отдельные страницы:

Мониторинг MikroTik через SNMP

Мониторинг Docker контейнеров

Мониторинг WireGuard/VPN

Мониторинг сайтов и HTTP-check

Мониторинг Zabbix-агентов через Prometheus exporter

Уведомления в Telegram

🎉 Prometheus + Grafana полностью установлены и настроены!