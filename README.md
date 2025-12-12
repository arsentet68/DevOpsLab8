# ФИТ-2024 НМ. Полынский Арсений. DevOps. ЛР8
## 1. Собираем веб-приложение (flask+redis)
Устанавливаем compose
```
sudo apt update
sudo apt install docker-compose-v2
```
Создаём каталог для проекта

<img width="325" height="137" alt="image" src="https://github.com/user-attachments/assets/2521f4f9-45b2-43de-8010-d27ddf08553a" />

### Создаём веб-приложение (app.py)
```
import time

import redis
from flask import Flask

app = Flask(__name__)
cache = redis.Redis(host='redis', port=6379)

def get_hit_count():
    return cache.incr('hits')

@app.route('/')
def hello():
    count = get_hit_count()
    return 'Hello World! I have been seen {} times.\n'.format(count)
```
### Объявляем зависимости для python (requirements.txt)
```
flask
redis
```
#### Создаём dockerfile для контейнера с flask
```
# syntax=docker/dockerfile:1
FROM python:3.10-alpine
WORKDIR /code
ENV FLASK_APP=app.py
ENV FLASK_RUN_HOST=0.0.0.0
RUN apk add --no-cache gcc musl-dev linux-headers
COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt
EXPOSE 5000
COPY app.py .
CMD ["flask", "run", "--debug"]
```
### Создаём compose.yaml - описание 2-х сервис-контейнеров
```
services:
  web:
    build: .
    ports:
      - "8000:5000"
  redis:
    image: "redis:alpine"
```
### Запускаем оба сервис-контейнера Flask+Redis

<img width="1487" height="882" alt="image" src="https://github.com/user-attachments/assets/2d96e9b3-cda9-4dde-92fe-2021e8a6047e" />

### Проверка стека Flask+Redis

<img width="1411" height="90" alt="image" src="https://github.com/user-attachments/assets/1d6a5f29-e1e6-4770-87ba-fd54edeb04e1" />
<img width="579" height="135" alt="image" src="https://github.com/user-attachments/assets/543b5ff6-e366-4fe3-8c96-3bd31c52cada" />
<img width="343" height="138" alt="image" src="https://github.com/user-attachments/assets/ada0b395-ad82-4640-aad3-f69d7219056d" />

## 2. Собираем мониторинг (prometheus+grafana)
Создаём каталог для проекта

<img width="296" height="158" alt="image" src="https://github.com/user-attachments/assets/a1de12e2-183b-43bd-979c-8d8a09e7e09d" />

### Создаём compose.yaml для Prometheus+Grafana
```
services:
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    ports:
      - 9090:9090
    restart: unless-stopped
    volumes:
      - ./prometheus:/etc/prometheus
      - prom_data:/prometheus
  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - 3000:3000
    restart: unless-stopped
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=grafana
    volumes:
      - ./grafana:/etc/grafana/provisioning/datasources
      - grafana_data:/var/lib/grafana
volumes:
  prom_data:
  grafana_data:
```

### Создаём конфиг для prometheus (prometheus.yml)
```
global:
  scrape_interval: 5s
  scrape_timeout: 3s
  evaluation_interval: 15s
alerting:
  alertmanagers:
  - static_configs:
    - targets: []
    scheme: http
    timeout: 10s
    api_version: v2
scrape_configs:
- job_name: prometheus
  honor_timestamps: true
  scrape_interval: 15s
  scrape_timeout: 10s
  metrics_path: /metrics
  scheme: http
  static_configs:
  - targets:
    - localhost:9090
```

### Создаём конфиг для grafana (datasource.yml)
```
apiVersion: 1

datasources:
- name: Prometheus
  type: prometheus
  url: http://prometheus:9090
  isDefault: true
  access: proxy
  editable: true
```

### Запускаем оба сервис-контейнера Prometheus+Grafana

<img width="1483" height="439" alt="image" src="https://github.com/user-attachments/assets/cdbf3270-8bd0-400e-b0f7-2319f705b909" />

### Проверка мониторинга Prometheus+Grafana

<img width="1316" height="98" alt="image" src="https://github.com/user-attachments/assets/6690e558-f603-41d3-8e49-8589dc584026" />

<img width="1217" height="877" alt="image" src="https://github.com/user-attachments/assets/84b61241-21f4-4a8b-9880-ba1ca8b2fc5d" />

### Смотрим метрику самого Prometheus

<img width="1910" height="878" alt="image" src="https://github.com/user-attachments/assets/16d8eb6b-bd41-48eb-b811-aa96e5c4a982" />

### Добавляем мониторинг нашего Flask-приложения (и не только)

Добавим новый контейнер blackbox в compose.yaml:

```
  blackbox:
      image: prom/blackbox-exporter
      container_name: blackbox
      ports:
        - 9115:9115
```

Обновим конфиг prometheus, добавим в scrape_configs указания что мониторить через blackbox

```
- job_name: blackbox-http
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets:
      - http://10.0.2.15:8000
      - https://etis.psu.ru
      - https://student.psu.ru
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: blackbox:9115
```

### Пересобираем стек мониторинга

Выключаем старую сборку контейнеров мониторинга

<img width="567" height="113" alt="image" src="https://github.com/user-attachments/assets/10d650a1-680d-4b63-ba3b-004ec08db5bb" />

Запускаем новую сборку контейнеров мониторинга

```
docker compose build
docker compose up -d
```

<img width="563" height="266" alt="image" src="https://github.com/user-attachments/assets/74e56f54-bdb8-4731-aa60-1c9f357496d3" />

### Проверка экспортера

<img width="1089" height="267" alt="image" src="https://github.com/user-attachments/assets/1c3f713e-f2d9-45e5-9d8f-ecfe97566afb" />

### Создаем дашборд

<img width="1198" height="478" alt="image" src="https://github.com/user-attachments/assets/45545f9e-affe-4342-ad4b-c48356fa531b" />

<img width="811" height="719" alt="image" src="https://github.com/user-attachments/assets/5d5bf182-a9f1-453d-a12d-a8bf9ee794a4" />

### Проверяем отображение метрик на дашборде

<img width="1885" height="813" alt="image" src="https://github.com/user-attachments/assets/e543b877-d6d4-4c37-a4d7-e7c084ec9825" />






