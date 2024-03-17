## Docker-compose
## Unifi
```
---
version: "2.1"
services:
  unificontroller:
    image: jacobalberty/unifi:latest
    container_name: unifi-controller
    environment:
      - TZ=Europe/Amsterdam
     #- MEM_LIMIT=1024 #optional
     #- MEM_STARTUP=1024 #optional
    volumes:
      - ./config:/config
    ports:
      - 8443:8443
      - 3478:3478/udp
      - 10001:10001/udp
      - 8080:8080
      - 1900:1900/udp
      - 8843:8843
      - 8880:8880
      - 6789:6789
      - 5514:5514/udp
    restart: unless-stopped
```
## Gitea
```
version: "3"

services:
  gitea:
    container_name: gitea
    environment:
      - USER_UID=1000
      - USER_GID=1000
    hostname: gitea
    ports:
      - 3000:3000 #webgui
      - 2222:22 #ssh
    image: gitea/gitea:latest
    restart: unless-stopped
    volumes:
      - ./gitea/data:/data
```
## Glances
```
version: '3'

services:
  monitoring:
    image: nicolargo/glances:latest-full
    pid: host
    network_mode: host
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
    environment:
      - "GLANCES_OPT=-w"
```
## Grafana-Mikrotik
```
version: '3.9'

services:
  grafana:
    image: grafana/grafana:9.0.5
    env_file:
      - .grafana
    container_name: mk_grafana
    restart: always
    volumes:
      - ./grafana/provisioning/:/etc/grafana/provisioning/
    ports:
      - "3000:3000"

  prometheus:
    image: prom/prometheus
    env_file:
      - .prometheus
    user: ${CURRENT_USER}
    container_name: mk_prometheus
    restart: always
    volumes:
      - ./prometheus/:/etc/prometheus
      - ./prometheus/data/:/prometheus
    ports:
      - "9090:9090"

  snmp_exporter:
    image: prom/snmp-exporter
    container_name: mk_snmp_exporter
    restart: always
    volumes:
      - ./snmp/:/etc/snmp_exporter/
    ports:
      - "9116:9116"
    depends_on:
      - prometheus
```
## Graylog
```
version: '3'

networks:
  graynet:
    driver: bridge

# This is how you persist data between container restarts
volumes:
  mongo_data:
    driver: local
  log_data:
    driver: local
  graylog_data:
    driver: local

services:
  # Graylog stores configuration in MongoDB
  mongo:
    image: mongo:6.0.5-jammy
    container_name: mongodb
    volumes:
      - "mongo_data:/data/db"
    networks:
      - graynet
    restart: unless-stopped

  # The logs themselves are stored in Opensearch
  opensearch:
    image: opensearchproject/opensearch:2
    container_name: opensearch
    environment:
      - "OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g"
      - "bootstrap.memory_lock=true"
      - "discovery.type=single-node"
      - "action.auto_create_index=false"
      - "plugins.security.ssl.http.enabled=false"
      - "plugins.security.disabled=true"
    volumes:
      - "log_data:/usr/share/opensearch/data"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    ports:
      - 9200:9200/tcp
    networks:
      - graynet
    restart: unless-stopped

  graylog:
    image: graylog/graylog:5.2.2
    container_name: graylog
    environment:
      # CHANGE ME (must be at least 16 characters)!
      GRAYLOG_PASSWORD_SECRET: "somepasswordpepper"
      # Password: admin
      GRAYLOG_ROOT_PASSWORD_SHA2: "8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918"
      GRAYLOG_HTTP_BIND_ADDRESS: "0.0.0.0:9000"
      GRAYLOG_HTTP_EXTERNAL_URI: "http://localhost:9000/"
      GRAYLOG_ELASTICSEARCH_HOSTS: "http://opensearch:9200"
      GRAYLOG_MONGODB_URI: "mongodb://mongodb:27017/graylog"
      GRAYLOG_TIMEZONE: "Europe/Amsterdam"
      TZ: "Europe/Amsterdam"

    entrypoint: /usr/bin/tini -- wait-for-it opensearch:9200 -- /docker-entrypoint.sh
    volumes:
      - "${PWD}/config/graylog/graylog.conf:/usr/share/graylog/config/graylog.conf"
      - "graylog_data:/usr/share/graylog/data"
    networks:
      - graynet
    restart: always
    depends_on:
      opensearch:
        condition: "service_started"
      mongo:
        condition: "service_started"
    ports:
      - 9000:9000/tcp   # Graylog web interface and REST API
      - 1514:1514/tcp   # Syslog
      - 1514:1514/udp   # Syslog
```
## Home Assistant
```
version: '3'
services:
  homeassistant:
    container_name: homeassistant
    image: "ghcr.io/homeassistant/home-assistant:stable"
    ports:
      - 8123:8123
    volumes:
      - ./config:/config
      - /etc/localtime:/etc/localtime:ro
    restart: unless-stopped
    privileged: true
```
## Kasm
```
---
version: "2.1"
services:
  kasm:
    image: lscr.io/linuxserver/kasm:latest
    container_name: kasm
    privileged: true
    environment:
      - KASM_PORT=443
    volumes:
      - ./data:/opt
      - ./profiles:/profiles
    ports:
      - 3000:3000
      - 443:443
    restart: unless-stopped
```
## Paperless-ngx
```
version: "3.4"
services:
  broker:
    image: docker.io/library/redis:7
    restart: unless-stopped
    volumes:
      - redisdata:/data

  webserver:
    image: ghcr.io/paperless-ngx/paperless-ngx:latest
    restart: unless-stopped
    depends_on:
      - broker
    ports:
      - "8000:8000"
    healthcheck:
      test: ["CMD", "curl", "-fs", "-S", "--max-time", "2", "http://localhost:8000"]
      interval: 30s
      timeout: 10s
      retries: 5
    volumes:
      - data:/usr/src/paperless/data
      - media:/usr/src/paperless/media
      - ./export:/usr/src/paperless/export
      - ./consume:/usr/src/paperless/consume
    env_file: docker-compose.env
    environment:
      PAPERLESS_REDIS: redis://broker:6379


volumes:
  data:
  media:
  redisdata:
```
## Heimdall
```
version: "3.4"
services:
  broker:
    image: docker.io/library/redis:7
    restart: unless-stopped
    volumes:
      - redisdata:/data

  webserver:
    image: ghcr.io/paperless-ngx/paperless-ngx:latest
    restart: unless-stopped
    depends_on:
      - broker
    ports:
      - "8000:8000"
    healthcheck:
      test: ["CMD", "curl", "-fs", "-S", "--max-time", "2", "http://localhost:8000"]
      interval: 30s
      timeout: 10s
      retries: 5
    volumes:
      - data:/usr/src/paperless/data
      - media:/usr/src/paperless/media
      - ./export:/usr/src/paperless/export
      - ./consume:/usr/src/paperless/consume
    env_file: docker-compose.env
    environment:
      PAPERLESS_REDIS: redis://broker:6379


volumes:
  data:
  media:
  redisdata:
```
## Torrent
```
---
version: "2.1"
services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - TZ=Europe/Amsterdam
      - WEBUI_PORT=8080
    volumes:
      - ./config:/config
      - ./downloads:/downloads
    ports:
      - 8080:8080
      - 6881:6881
      - 6881:6881/udp
    restart: unless-stopped
```
## Uptime-kuma
```
version: '3.8'

services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    volumes:
      - ./uptime-kuma:/app/data
    ports:
      - "3001:3001"
    restart: always

volumes:
  uptime-kuma:
```
