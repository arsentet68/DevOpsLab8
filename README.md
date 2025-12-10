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
