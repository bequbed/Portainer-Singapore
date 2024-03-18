# Portainer-Singapore
Real Debrid Stack on Oracle VPS


version: '3.8'

services:
  plex:
    container_name: plex
    network_mode: host
    devices:
      - /dev/dri:/dev/dri
    environment:
      - PLEX_UID=1000
      - PLEX_GID=1000
    ports:
      - 1900:1900/udp
      - 32400:32400/tcp
      - 32410:32410/udp
      - 32412:32412/udp
      - 32413:32413/udp
      - 32414:32414/udp
      - 32469:32469/tcp
      - 8324:8324/tcp
    hostname: plex
    image: lscr.io/linuxserver/plex:latest
    restart: unless-stopped
    volumes:
      - /dev/shm:/dev/shm
      - /mnt/local/transcodes/plex:/transcode
      - /mnt:/mnt
      - /opt/plex:/config
      - /opt/scripts:/scripts
    depends_on:
      - rclone
      - rdtclient
  
  zurg:
    #image: ghcr.io/debridmediamanager/zurg-testing:latest
    image: ghcr.io/debridmediamanager/zurg-testing:v0.9.3-hotfix.11
    container_name: zurg
    restart: unless-stopped
    healthcheck:
      test: curl -f localhost:9999/dav/version.txt || exit 1
    ports:
      - 9999:9999
    volumes:
      - /opt/zurg-testing/plex_update.sh:/app/plex_update.sh
      - /opt/zurg-testing/config.yml:/app/config.yml
      - zurgdata:/app/data

  rclone:
    image: rclone/rclone:latest
    container_name: rclone
    restart: unless-stopped
    environment:
      TZ: Pacific/Auckland
      PUID: 1000
      PGID: 1000
    volumes:
      #- /mnt/zurg:/data:rshared # CHANGE /mnt/zurg WITH YOUR PREFERRED MOUNT PATH
      - /mnt/remote/realdebrid:/data:rshared
      - /opt/zurg-testing/rclone.conf:/config/rclone/rclone.conf
      - /mnt:/mnt
    cap_add:
      - SYS_ADMIN
    security_opt:
      - apparmor:unconfined
    devices:
      - /dev/fuse:/dev/fuse:rwm
    depends_on:
      - zurg
    #command: "mount zurg: /data --allow-other --allow-non-empty --dir-cache-time 10s --vfs-cache-mode full"
    command:  "mount zurg: /data --allow-other --allow-non-empty --dir-cache-time 10s --vfs-cache-mode full --vfs-cache-max-size 25G --vfs-cache-max-age 24h"

  rdtclient:
    container_name: rdtclient
    environment:
      - "PGID=1000"
      - "PUID=1000"
    ports:
      - 6500:6500/tcp
    hostname: rdtclient
    # image: pukabyte/rdtclient:latest
    image: laster13/rdtclient:arm64
    restart: unless-stopped
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /mnt:/mnt
      - /opt/rdtclient/data:/data
      - /opt/rdtclient/data/db:/data/db
    depends_on:
      - rclone
      - zurg
    healthcheck:
      # Replace with an appropriate check for rdtclient
      test: curl -f  http://localhost:6500 || exit 1 
      # interval: 30s
      # timeout: 10s
      # retries: 5  

  sonarr:
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=1000
    ports:
      - 8989:8989/tcp
    hostname: sonarr
    image: ghcr.io/hotio/sonarr:release
    restart: unless-stopped
    volumes:
      - /mnt:/mnt
      - /opt/scripts:/scripts
      - /opt/sonarr:/config
    depends_on:
      - rclone
      - rdtclient
       
  radarr:
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=1000
    ports:
      - 7878:7878/tcp
    hostname: radarr
    image: ghcr.io/hotio/radarr:release
    restart: unless-stopped
    volumes:
      - /mnt:/mnt
      - /opt/radarr:/config
      - /opt/scripts:/scripts
    depends_on:
      - rclone
      - rdtclient
      
  prowlarr:
    container_name: prowlarr
    environment:
      - PUID=1000
      - PGID=1000
    ports:
      - 9696:9696/tcp
    hostname: prowlarr
    image: ghcr.io/hotio/prowlarr:release
    restart: unless-stopped
    volumes:
      - /mnt:/mnt
      - /opt/prowlarr/Definitions/Custom:/Custom
      - /opt/prowlarr:/config
    depends_on:
      - rclone
      - rdtclient  
        
volumes:
  zurgdata:
