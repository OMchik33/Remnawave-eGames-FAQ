# Мониторинг Remnawave через Prometheus и Grafana

Полная инструкция по настройке мониторинга панели [Remnawave](https://remna.st) с помощью Prometheus, Grafana и Node Exporter, а также безопасного доступа к метрикам через SSH-туннель и nginx. 
*(Туннель нужен если метрики закрыты по куки авторизации, например при установке Remnawave по скрипту eGames)*

---

## 📦 1. Настройка `docker-compose.yml` на сервере с Remnawave

Убедитесь, что в `docker-compose.yml` Remnawave открыт порт `3001`, по которому отдаются метрики:

```yaml
ports:
  - '127.0.0.1:3001:3001'
```

<details>
  <summary>Пример конфигурации (вариант установки по скрипту eGames):</summary>

```yaml
remnawave:
  image: remnawave/backend:latest
  container_name: remnawave
  hostname: remnawave
  restart: always
  env_file:
    - .env
  ports:
    - '127.0.0.1:3000:3000'
    - '127.0.0.1:3001:3001'
  networks:
    - remnawave-network
  depends_on:
    remnawave-db:
      condition: service_healthy
    remnawave-redis:
      condition: service_healthy
  logging:
    driver: 'json-file'
    options:
      max-size: '30m'
      max-file: '5'
```

</details>

---

## 🔐 2. Настройка SSH-туннеля

### На мониторинговом сервере:

1. Генерация ключа:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/remna_tunnel_key
```

> Без пароля.

2. Добавление публичного ключа на сервере с Remnawave:

В файл `~/.ssh/authorized_keys`:

```bash
from="IP_мониторингового_сервера",no-pty,no-agent-forwarding,no-X11-forwarding,command="/bin/false" ssh-ed25519 AAAAC3... remna_tunnel_key
```

> Вместо `AAAAC3...` вставьте содержимое `remna_tunnel_key.pub`, начинающееся на `AAAAC3`.

3. Установка `autossh`: (продолжаем на мониторинговом сервере)

```bash
sudo apt install autossh
```

4. Создание systemd-сервиса `/etc/systemd/system/remna-tunnel.service`:

```ini
[Unit]
Description=SSH tunnel to Remnawave for Prometheus and Node Exporter
After=network.target

[Service]
User=root
Environment="AUTOSSH_GATETIME=0"
ExecStart=/usr/bin/autossh -N \
 -o "ServerAliveInterval 60" \
 -o "ServerAliveCountMax 3" \
 -i /root/.ssh/remna_tunnel_key \
 -L 9001:localhost:3001 \
 -L 9002:localhost:9100 \
 remnauser@REMNA_SERVER_IP
Restart=always

[Install]
WantedBy=multi-user.target
```

> Замените `remnauser@REMNA_SERVER_IP`, это ssh логин и адрес сервера панели Remnawave.

>На первом сервере (где запускается autossh) нужно добавить ключ второго сервера в known_hosts, чтобы избежать ошибки проверки подлинности хоста:

```
ssh-keyscan -H 12.345.678.91 >> ~/.ssh/known_hosts
```

где 12.345.678.91 - IP адрес панели Remnawave


5. Запуск сервиса:

```bash
sudo systemctl daemon-reexec
sudo systemctl enable remna-tunnel
sudo systemctl start remna-tunnel
```

Проверка успешности запуска:

```
sudo systemctl status remna-tunnel
```

Теперь метрики Remnawave и Node Exporter доступны по `http://localhost:9001/metrics` и `http://localhost:9002/metrics`.

---

## 📈 3. Установка Prometheus и Grafana

Создание директорий:

```bash
mkdir -p /opt/monitoring/{grafana,prometheus}
```

Файл `/opt/monitoring/docker-compose.yml`:

<details>
  <summary>Открыть пример файла</summary>
  
```yaml
services:
#  uptime-kuma:
#    image: louislam/uptime-kuma
#    container_name: uptime-kuma
#    restart: always
#    ports:
#      - "3001:3001"
#    volumes:
#      - ./uptime-kuma-data:/app/data
#    network_mode: host
      
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.external-url=https://sub.mydomain.com/prometheus/'
      - '--web.route-prefix=/'
    network_mode: host

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SERVER_DOMAIN=yourdomain.com
      - GF_SERVER_ROOT_URL=https://sub.mydomain.com/grafana
      - GF_SERVER_SERVE_FROM_SUB_PATH=true
      - GF_SERVER_HTTP_PORT=3000
      - GF_SERVER_PROTOCOL=http
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_ANALYTICS_REPORTING_ENABLED=false
    network_mode: host
    
  xray-checker:
    image: kutovoys/xray-checker
    environment:
      - "SUBSCRIPTION_URL=https://podpiska.mydomain.com/6f5g46df46g45f54"
      - "PROXY_STATUS_CHECK_URL=http://google.com/generate_204"
      - "PROXY_CHECK_INTERVAL=60"
    ports:
      - "2112:2112"
    network_mode: host

volumes:
  prometheus-data:
  grafana-data:
```

Здесь `sub.mydomain.com` - адрес домена, прикрепленного к тестовому VPS, на котором устанавливаются Графана и Прометей

`https://podpiska.mydomain.com/6f5g46df46g45f54` - ВПН подписка, сделайте отдельного пользователя для этой роли.

</details>


---

## ⚙️ 4. Конфигурация Prometheus

Файл `/opt/monitoring/prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['127.0.0.1:9002']
        labels:
          label: "Remnaserver"
  - job_name: 'integrations/node_exporter'
    static_configs:
      - targets: ['127.0.0.1:9001']
        labels:
          cluster: "test"
          job: "integrations/node_exporter"
          instance: "127.0.0.1:9001"
    basic_auth:
      username: "XXXXXXXXXXXXXXX"
      password: "XXXXXXXXXXXXXXX"
  - job_name: "xray-checker"
    metrics_path: "/metrics"
    static_configs:
      - targets: ["localhost:2112"]
    scrape_interval: 1m
```

> username и password из `.env` файла Remnawave (секция `### PROMETHEUS ###`)

Запуск:

```bash
cd /opt/monitoring
docker compose up -d
```

---

## 🌐 5. Настройка Nginx и SSL

Установка:

```bash
apt install nginx
```

Получение SSL-сертификатов:

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d sub.mydomain.com
```

Автообновление:

```bash
0 5 * * * root certbot renew --quiet
```

<details>
  <summary>Пример конфигурации nginx</summary>

```
# Проверка по cookie
map $http_cookie $auth_cookie {
    default 0;
    "~*fd4gd54fg2dfg4241=1" 1;
}

# Проверка по GET-параметру
map $arg_fd4gd54fg2dfg4241 $auth_query {
    default 0;
    "1" 1;
}

# Общий флаг авторизации
map "$auth_cookie$auth_query" $authorized {
    "~1" 1;
    default 0;
}

# Установка куки, если есть параметр
map $arg_fd4gd54fg2dfg4241 $set_cookie_header {
    "1" "fd4gd54fg2dfg4241=1; Path=/; HttpOnly; Secure; SameSite=Strict; Max-Age=31536000";
    default "";
}

# HTTP редирект на HTTPS
server {
    listen 80;
    server_name sub.mydomain.com;
    return 301 https://$server_name$request_uri;
}

# HTTPS сервер блок
server {
    listen 443 ssl http2;
    server_name sub.mydomain.com;
    
    # SSL конфигурация
    ssl_certificate /etc/letsencrypt/live/sub.mydomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/sub.mydomain.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # Устанавливаем куку, если пользователь авторизуется по ссылке
    add_header Set-Cookie $set_cookie_header;

    # Редирект с основного домена сразу на нужный открытый дашбоард в Графане
    location = / {
        return 301 /grafana/public-dashboards/f5g4df4g5df4gd5f4g63d4834379e;
    }

    # Grafana конфигурация
    location /grafana {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $server_name;
        
        # WebSocket поддержка
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        # Убираем Authorization header
        proxy_set_header Authorization "";
    }

    # Grafana Live WebSocket
    location /grafana/api/live/ {
        proxy_pass http://localhost:3000/api/live/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Prometheus
    location /prometheus/ {
        if ($authorized = 0) {
            return 404;
        }

        proxy_pass http://localhost:9090/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $server_name;
        proxy_set_header Authorization "";
    }

    # Xray Checker
    location /checker/ {
        if ($authorized = 0) {
            return 404;
        }

        proxy_pass http://localhost:2112/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $server_name;
    }
}
```

Здесь `sub.mydomain.com` - адрес домена, прикрепленного к тестовому VPS, на котором устанавливаются Графана и Прометей
</details>

---

## 📊 6. Проверка и настройка Grafana

- Перейдите: `https://sub.mydomain.com/grafana`
- Вход: `admin / admin`, затем смените пароль
- Добавьте источник данных: **Prometheus**
  - URL: `http://localhost:9090`
- Перейдите в **Explore → Metrics → Grafana Drilldown → Metrics**

---

## 🧠 7. Node Exporter

Установка на сервер с Remnawave:

```bash
. <(wget -qO- https://raw.githubusercontent.com/g7AzaZLO/NodeExporter-autoinstaller/main/NodeExporter-auto-install.sh)
```

Node Exporter доступен по `localhost:9002` (через SSH-туннель).

Можно установить на другие сервера и добавить в `prometheus.yml`:

```yaml
- job_name: 'external_nodes'
  static_configs:
    - targets: ['1.2.3.4:9100']
```

Здесь `1.2.3.4` - адрес очередной ноды, на которую мы также установили Node Exporter

Или использовать SSH-туннели по аналогии.

Для визуализации:

- Dashboard ID: **1860**
- [https://grafana.com/grafana/dashboards/1860](https://grafana.com/grafana/dashboards/1860)

---

## 📙 Полезные ссылки

- [Remnawave Telegram метрики #1](https://t.me/c/2409638119/3118)
- [Remnawave Telegram метрики #2](https://t.me/c/2409638119/43140)

---

> 💬 Обратную связь, предложения и правки — приветствуются через issues или pull request.

