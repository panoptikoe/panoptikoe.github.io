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
