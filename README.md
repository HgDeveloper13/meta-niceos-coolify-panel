# Coolify Panel на NiceOS — Руководство по установке

![Coolify](https://coolify.io/android-launchericon-48-48.png)

## Описание

Это руководство поможет вам установить **Coolify Panel** (открытая альтернатива Heroku/Netlify/Vercel) на сервер с операционной системой **NiceOS** (НАЙС ОС, Российская ОС на базе Linux).

## Требования

| Параметр | Минимум | Рекомендуется |
|----------|---------|---------------|
| CPU | 2 ядра | 4+ ядра |
| RAM | 4 GB | 8+ GB |
| Диск | 30 GB | 50+ GB |
| Docker | 20.10+ | Последняя версия |

## Быстрый старт

### 1. Подключение к серверу

```bash
ssh root@<IP-ВАШЕГО-СЕРВЕРА>
```

### 2. Создание директорий

```bash
mkdir -p /data/coolify/{source,ssh/keys,applications,databases,backups,services,proxy}
mkdir -p /data/coolify-postgres
mkdir -p /data/coolify-redis
```

### 3. Создание .env файла

```bash
cd /data/coolify

# Сгенерировать ключи ДО создания файла
APP_ID=$(openssl rand -hex 16)
APP_KEY=$(openssl rand -base64 32)
ENCRYPTION_KEY=$(openssl rand -base64 32)
DB_PASSWORD=$(openssl rand -base64 32)
REDIS_PASSWORD=$(openssl rand -base64 32)
PUSHER_APP_ID=$(openssl rand -hex 32)
PUSHER_APP_KEY=$(openssl rand -hex 32)
PUSHER_APP_SECRET=$(openssl rand -hex 32)

# Записать .env с уже сгенерированными значениями
cat > .env << EOF
APP_NAME=coolify
APP_ENV=production
APP_DEBUG=false
APP_URL=http://<IP-СЕРВЕРА>:8000
APP_HOST=0.0.0.0
APP_PORT=8000

APP_ID=${APP_ID}
APP_KEY=base64:${APP_KEY}
ENCRYPTION_KEY=base64:${ENCRYPTION_KEY}

DB_CONNECTION=pgsql
DB_HOST=coolify-db
DB_PORT=5432
DB_DATABASE=coolify
DB_USERNAME=coolify
DB_PASSWORD=${DB_PASSWORD}

REDIS_HOST=coolify-redis
REDIS_PORT=6379
REDIS_PASSWORD=${REDIS_PASSWORD}

PUSHER_APP_ID=${PUSHER_APP_ID}
PUSHER_APP_KEY=${PUSHER_APP_KEY}
PUSHER_APP_SECRET=${PUSHER_APP_SECRET}
PUSHER_HOST=coolify-realtime
PUSHER_PORT=6001

DOCKER_HOST=unix:///var/run/docker.sock
REGISTRY_URL=ghcr.io
AUTOUPDATE=true

SSL_MODE=off
EOF
```

**Важно**: Замените `<IP-СЕРВЕРА>` на IP адрес вашего сервера!

### 4. Создание docker-compose.yml

```bash
cat > /data/coolify/docker-compose.yml << 'EOF'
version: '3.8'

services:
  coolify:
    image: ghcr.io/coollabsio/coolify:latest
    container_name: coolify
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      - APP_NAME=coolify
      - APP_ENV=production
      - APP_DEBUG=false
      - APP_URL=http://<IP-СЕРВЕРА>:8000
      - APP_HOST=0.0.0.0
      - APP_PORT=8000
      - APP_KEY=${APP_KEY}
      - APP_ID=${APP_ID}
      - ENCRYPTION_KEY=${ENCRYPTION_KEY}
      - DB_CONNECTION=pgsql
      - DB_HOST=coolify-db
      - DB_PORT=5432
      - DB_DATABASE=coolify
      - DB_USERNAME=coolify
      - DB_PASSWORD=${DB_PASSWORD}
      - REDIS_HOST=coolify-redis
      - REDIS_PORT=6379
      - REDIS_PASSWORD=${REDIS_PASSWORD}
      - PUSHER_APP_ID=${PUSHER_APP_ID}
      - PUSHER_APP_KEY=${PUSHER_APP_KEY}
      - PUSHER_APP_SECRET=${PUSHER_APP_SECRET}
      - PUSHER_HOST=coolify-realtime
      - PUSHER_PORT=6001
      - DOCKER_HOST=unix:///var/run/docker.sock
      - SSL_MODE=off
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /data/coolify:/var/lib/coolify
    networks:
      - coolify-net
    depends_on:
      coolify-db:
        condition: service_healthy
      coolify-redis:
        condition: service_started

  coolify-db:
    image: postgres:15-alpine
    container_name: coolify-db
    restart: unless-stopped
    environment:
      - POSTGRES_USER=coolify
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=coolify
    volumes:
      - /data/coolify-postgres:/var/lib/postgresql/data
    networks:
      - coolify-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U coolify -d coolify"]
      interval: 10s
      timeout: 5s
      retries: 5

  coolify-redis:
    image: redis:7-alpine
    container_name: coolify-redis
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - /data/coolify-redis:/data
    networks:
      - coolify-net

  coolify-realtime:
    image: ghcr.io/coollabsio/coolify-realtime:latest
    container_name: coolify-realtime
    restart: unless-stopped
    environment:
      - APP_ID=${PUSHER_APP_ID}
      - APP_KEY=${PUSHER_APP_KEY}
      - APP_SECRET=${PUSHER_APP_SECRET}
    ports:
      - "6001:6001"
    networks:
      - coolify-net
    depends_on:
      - coolify-redis

networks:
  coolify-net:
    driver: bridge
    name: coolify-net
EOF
```

**Важно**: Замените `<IP-СЕРВЕРА>` на IP адрес вашего сервера!

### 5. Запуск

```bash
# Установить права
chown -R 9999:root /data/coolify
chmod -R 700 /data/coolify

# Запустить контейнеры
cd /data/coolify
docker-compose up -d
```

### 6. Открытие портов

```bash
# NiceOS (iptables)
sudo iptables -I INPUT -p tcp --dport 8000 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 6001 -j ACCEPT
```

### 7. Проверка

```bash
# Статус контейнеров
docker-compose ps

# Логи
docker-compose logs -f coolify
```

## Доступ

После установки откройте в браузере:

```
http://<IP-ВАШЕГО-СЕРВЕРА>:8000
```

## Устранение проблем

### Не открывается страница

1. Проверьте, что контейнеры запущены:
   ```bash
   docker-compose ps
   ```

2. Проверьте порты:
   ```bash
   ss -tlnp | grep -E ':(8000|6001)'
   ```

3. Проверьте firewall:
   ```bash
   sudo iptables -L INPUT -n | grep 8000
   ```

4. Проверьте группу безопасности облака

### Высокая нагрузка CPU

```bash
# Ограничить ресурсы
docker update --cpus=0.75 --memory=768m coolify
```

## Бэкап

**Важно**: Сохраните файл `.env` в надёжном месте!

```bash
cp /data/coolify/.env ~/coolify-backup-$(date +%F).env
```

## Обновление

```bash
cd /data/coolify
docker-compose pull
docker-compose up -d
```

## Удаление

```bash
cd /data/coolify
docker-compose down

# Удалить данные (осторожно!)
rm -rf /data/coolify
rm -rf /data/coolify-postgres
rm -rf /data/coolify-redis
```

## Документация

- [Официальная документация Coolify](https://coolify.io/docs)
- [GitHub](https://github.com/coollabsio/coolify)

## Лицензия

MIT License — свободное использование и модификация.
