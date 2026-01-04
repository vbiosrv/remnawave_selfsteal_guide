remnawave_selfsteal_guide
Полное пошаговое руководство по развертыванию ноды SelfSteal на Remnawave, включая полный процесс настройки самой платформы Remnawave. Это руководство охватывает предварительные требования, установку, конфигурацию и советы по устранению неисправностей для обеспечения плавного и безопасного развертывания.
Это руководство предоставляет пошаговую инструкцию по настройке панели управления и ноды, каждая из которых развертывается как минимум на двух отдельных экземплярах VPS, соответствующих системным требованиям, описанным здесь: https://remna.st/docs/install/requirements.
Мы начнем с базовой настройки панели управления Remnawave.
Шаг 0 — Обновление и установка необходимых инструментов
Обновление всей системы и установка curl
apt update && apt upgrade && apt install curl
Шаг 1 — Установка Docker
Установка Docker с использованием официального удобного скрипта
curl -fsSL https://get.docker.com | sh
Шаг 2 — Загрузка необходимых файлов
Создание директории проекта и переход в нее
mkdir -p /opt/remnawave && cd /opt/remnawave
Загрузка файла Docker Compose
curl -o docker-compose.yml https://raw.githubusercontent.com/remnawave/backend/refs/heads/main/docker-compose-prod.yml
Загрузка примера файла окружения
curl -o .env https://raw.githubusercontent.com/remnawave/backend/refs/heads/main/.env.sample
Шаг 3 — Конфигурация .env
Генерация секретов JWT для аутентификации и подписи API-токенов
sed -i "s/^JWT_AUTH_SECRET=.*/JWT_AUTH_SECRET=$(openssl rand -hex 64)/" .env && sed -i "s/^JWT_API_TOKENS_SECRET=.*/JWT_API_TOKENS_SECRET=$(openssl rand -hex 64)/" .env
Генерация дополнительных секретов для метрик и вебхуков
sed -i "s/^METRICS_PASS=.*/METRICS_PASS=$(openssl rand -hex 64)/" .env && sed -i "s/^WEBHOOK_SECRET_HEADER=.*/WEBHOOK_SECRET_HEADER=$(openssl rand -hex 64)/" .env
Ротация пароля Postgres и синхронизация DATABASE_URL
pw=$(openssl rand -hex 24) && sed -i "s/^POSTGRES_PASSWORD=.*/POSTGRES_PASSWORD=$pw/" .env && sed -i "s|^\(DATABASE_URL=\"postgresql://postgres:\)[^\@]*\(@.*\)|\1$pw\2|" .env
Настройка доменов
nano .env
Найдите в .env эти значения
FRONT_END_DOMAIN="panel.yourdomain.com"
SUB_PUBLIC_DOMAIN="sub.yourdomain.com"
Замените "panel.yourdomain.com" и "sub.yourdomain.com" на ваши домены
Шаг 4 — Запуск стека
Запуск контейнеров и просмотр логов с метками времени
docker compose up -d && docker compose logs -f -t
Шаг 5 — Направление доменных имен на ваш сервер через DNS-провайдера или регистратора
Шаг 6 — Конфигурация Caddy
Создайте файл Caddyfile в директории /opt/remnawave/caddy.
mkdir -p /opt/remnawave/caddy && cd /opt/remnawave/caddy && nano Caddyfile
Вставьте следующую конфигурацию
texthttps://REPLACE_WITH_YOUR_DOMAIN {
        reverse_proxy * http://remnawave:3000
}
:443 {
    tls internal
    respond 204
}
https://SUBSCRIPTION_PAGE_DOMAIN {
        reverse_proxy * http://remnawave-subscription-page:3010
}
Создайте docker-compose.yml
cd /opt/remnawave/caddy && nano docker-compose.yml
Вставьте следующую конфигурацию.
textservices:
    caddy:
        image: caddy:2.9
        container_name: 'caddy'
        hostname: caddy
        restart: always
        ports:
            - '0.0.0.0:443:443'
            - '0.0.0.0:80:80'
        networks:
            - remnawave-network
        volumes:
            - ./Caddyfile:/etc/caddy/Caddyfile
            - caddy-ssl-data:/data
networks:
    remnawave-network:
        name: remnawave-network
        driver: bridge
        external: true
volumes:
    caddy-ssl-data:
        driver: local
        external: false
        name: caddy-ssl-data
Запуск контейнера
docker compose up -d && docker compose logs -f -t
Страница подписки Remnawave
Создание файла docker-compose.yml
mkdir -p /opt/remnawave/subscription && cd /opt/remnawave/subscription && nano docker-compose.yml
Вставьте содержимое файла docker-compose.yml
textservices:
    remnawave-subscription-page:
        image: remnawave/subscription-page:latest
        container_name: remnawave-subscription-page
        hostname: remnawave-subscription-page
        restart: always
        environment:
            - REMNAWAVE_PANEL_URL=https://panel.com
            - APP_PORT=3010
            - META_TITLE="Subscription Page Title"
            - META_DESCRIPTION="Subscription Page Description"
        ports:
            - '127.0.0.1:3010:3010'
        networks:
            - remnawave-network
networks:
    remnawave-network:
        driver: bridge
        external: true
Запуск контейнера
docker compose up -d && docker compose logs -f
Шаг 7 — Конфигурация ноды Remnawave
Откройте ваш второй VPS-сервер, предназначенный для ноды
Обновление всей системы и установка curl
apt update && apt upgrade && apt install curl
Установка Docker с использованием официального удобного скрипта
curl -fsSL https://get.docker.com | sh
Создание директории проекта
mkdir -p /opt/remnanode && cd /opt/remnanode
Конфигурация файла .env
nano .env
Содержимое файла .env
textNODE_PORT=2222
SECRET_KEY=CERT_FROM_MAIN_PANEL
Создание файла docker-compose.yml
nano docker-compose.yml
Содержимое файла docker-compose.yml
textservices:
    remnanode:
        container_name: remnanode
        hostname: remnanode
        image: remnawave/node:latest
        restart: always
        network_mode: host
        env_file:
            - .env
Запуск контейнера
docker compose up -d && docker compose logs -f
Шаг 8 — Настройка Selfsteal (SNI)
Создание рабочей директории и редактирование Caddyfile
mkdir -p /opt/selfsteel && cd /opt/selfsteel && nano Caddyfile
Вставьте следующее
text{
    https_port {$SELF_STEAL_PORT}
    default_bind 127.0.0.1
    servers {
        listener_wrappers {
            proxy_protocol {
                allow 127.0.0.1/32
            }
            tls
        }
    }
    auto_https disable_redirects
}
http://{$SELF_STEAL_DOMAIN} {
    bind 0.0.0.0
    redir https://{$SELF_STEAL_DOMAIN}{uri} permanent
}
https://{$SELF_STEAL_DOMAIN} {
    root * /var/www/html
    try_files {path} /index.html
    file_server
}
:{$SELF_STEAL_PORT} {
    tls internal
    respond 204
}
:80 {
    bind 0.0.0.0
    respond 204
}
Конфигурация переменных окружения
nano .env
Вставьте (замените steel.domain.com на ваш домен-заполнитель):
SELF_STEAL_DOMAIN=steel.domain.comДОЛЖЕН совпадать с XRAY realitySettings.serverNamesSELF_STEAL_PORT=9443ДОЛЖЕН совпадать с XRAY realitySettings.dest
Создание docker-compose.yml
nano docker-compose.yml
Вставьте:
textservices:
  caddy:
    image: caddy:latest
    container_name: caddy-remnawave
    restart: unless-stopped
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - ../html:/var/www/html
      - ./logs:/var/log/caddy
      - caddy_data_selfsteal:/data
      - caddy_config_selfsteal:/config
    env_file:
      - .env
    network_mode: "host"
volumes:
  caddy_data_selfsteal:
  caddy_config_selfsteal:
Запуск и проверка
docker compose up -d && docker compose logs -f -t
Создание сайта-заполнителя
textmkdir -p /opt/html
printf '%s\n' '<!doctype html><meta charset="utf-8"><title>Selfsteal</title><h1>It works.</h1>' \
  > /opt/html/index.html
Шаг 9
Конфигурация Xray (Reality) — обновление через панель
Примечание: Inbound Shadowsocks является опциональным и может быть безопасно удален в будущем, если не требуется.
text{
  "log": {
    "loglevel": "info"
  },
  "inbounds": [
    {
      "tag": "Shadowsocks",
      "port": 1234,
      "protocol": "shadowsocks",
      "settings": {
        "clients": [],
        "network": "tcp,udp"
      },
      "sniffing": {
        "enabled": true,
        "destOverride": [
          "http",
          "tls",
          "quic"
        ]
      }
    },
    {
      "tag": "VLESS",
      "port": 443,
      "listen": "0.0.0.0",
      "protocol": "vless",
      "settings": {
        "clients": [],
        "decryption": "none"
      },
      "sniffing": {
        "enabled": true,
        "routeOnly": false,
        "destOverride": [
          "http",
          "tls",
          "quic",
          "fakedns"
        ],
        "metadataOnly": false
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "tcpSettings": {
          "header": {
            "type": "none"
          },
          "acceptProxyProtocol": false
        },
        "realitySettings": {
          "dest": "9443",
          "show": false,
          "xver": 0,
          "spiderX": "/",
          "shortIds": [
            "CHANGE_ME_SHORTID"
          ],
          "publicKey": "CHANGE_ME_PUBLIC_KEY",
          "privateKey": "CHANGE_ME_PRIVATE_KEY",
          "fingerprint": "chrome",
          "serverNames": [
            "steel.domen.com"
          ]
        }
      }
    }
  ],
  "outbounds": [
    {
      "tag": "DIRECT",
      "protocol": "freedom"
    },
    {
      "tag": "BLOCK",
      "protocol": "blackhole"
    }
  ],
  "routing": {
    "rules": [
      {
        "ip": [
          "geoip:private"
        ],
        "type": "field",
        "outboundTag": "BLOCK"
      },
      {
        "type": "field",
        "domain": [
          "geosite:private"
        ],
        "outboundTag": "BLOCK"
      },
      {
        "type": "field",
        "protocol": [
          "bittorrent"
        ],
        "outboundTag": "BLOCK"
      }
    ]
  }
}
