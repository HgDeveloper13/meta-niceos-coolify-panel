# СКИЛЛ: Установка Coolify Panel на NiceOS через Docker

## Метаинформация

- **Название**: NiceOS Coolify Panel Installation
- **Версия**: 1.0
- **ОС**: NiceOS (РФ)/НАЙС ОС
- **Платформа**: Docker
- **Автор**: AI Assistant

## Обзор

Данный скилл предоставляет пошаговую инструкцию для установки Coolify Panel (open-source PaaS) на российскую операционную систему НАЙС ОС с использованием Docker Compose.

## Требования

- NiceOS 5.2+ (или любой RHEL-based дистрибутив)
- Docker Engine 20.10+
- Docker Compose 2.0+
- Минимум 2 ядра CPU, 4GB RAM, 30GB диск
- SSH доступ к серверу с правами root

## Предварительные проверки

### 1. Проверить статус Docker
```bash
docker --version
docker ps
```

### 2. Проверить доступные ресурсы
```bash
free -h
df -h /
nproc
```

### 3. Проверить что k0s/K8s не установлены (конфликт с Docker)
```bash
systemctl list-units | grep -i k0s
which k0s
```

## Структура файлов

### Основная директория
```
/data/coolify/
├── docker-compose.yml
├── .env
├── source/
├── ssh/keys/
├── applications/
├── databases/
├── backups/
├── services/
└── proxy/
```

### Дополнительные директории
```
/data/coolify-postgres/   # Данные PostgreSQL
/data/coolify-redis/      # Данные Redis
```

## Шаг 1: Создание структуры папок

```bash
mkdir -p /data/coolify/{source,ssh/keys,applications,databases,backups,services,proxy}
mkdir -p /data/coolify-postgres
mkdir -p /data/coolify-redis
```

## Шаг 2: Создание .env файла

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

**Важно**: Замените `<IP-СЕРВЕРА>` на IP адрес вашего VPS.

## Шаг 3: Создание docker-compose.yml

```bash
cat > /data/coolify/docker-compose.yml << 'EOF'
version: '3.8'

services:
  coolify:
    image: ${REGISTRY_URL:-ghcr.io}/coollabsio/coolify:latest
    container_name: coolify
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      - APP_NAME=${APP_NAME:-coolify}
      - APP_ENV=${APP_ENV:-production}
      - APP_DEBUG=${APP_DEBUG:-false}
      - APP_URL=${APP_URL:-http://localhost:8000}
      - APP_HOST=${APP_HOST:-0.0.0.0}
      - APP_PORT=${APP_PORT:-8000}
      - APP_KEY=${APP_KEY}
      - APP_ID=${APP_ID}
      - ENCRYPTION_KEY=${ENCRYPTION_KEY}
      - DB_CONNECTION=${DB_CONNECTION:-pgsql}
      - DB_HOST=${DB_HOST:-coolify-db}
      - DB_PORT=${DB_PORT:-5432}
      - DB_DATABASE=${DB_DATABASE:-coolify}
      - DB_USERNAME=${DB_USERNAME:-coolify}
      - DB_PASSWORD=${DB_PASSWORD}
      - REDIS_HOST=${REDIS_HOST:-coolify-redis}
      - REDIS_PORT=${REDIS_PORT:-6379}
      - REDIS_PASSWORD=${REDIS_PASSWORD}
      - PUSHER_APP_ID=${PUSHER_APP_ID}
      - PUSHER_APP_KEY=${PUSHER_APP_KEY}
      - PUSHER_APP_SECRET=${PUSHER_APP_SECRET}
      - PUSHER_HOST=${PUSHER_HOST:-coolify-realtime}
      - PUSHER_PORT=${PUSHER_PORT:-6001}
      - DOCKER_HOST=${DOCKER_HOST:-unix:///var/run/docker.sock}
      - SSL_MODE=${SSL_MODE:-off}
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
      - POSTGRES_USER=${DB_USERNAME:-coolify}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=${DB_DATABASE:-coolify}
    volumes:
      - /data/coolify-postgres:/var/lib/postgresql/data
    networks:
      - coolify-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USERNAME:-coolify} -d ${DB_DATABASE:-coolify}"]
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
    image: ${REGISTRY_URL:-ghcr.io}/coollabsio/coolify-realtime:latest
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

## Шаг 4: Установка прав и запуск

```bash
# Установить права
chown -R 9999:root /data/coolify
chmod -R 700 /data/coolify

# Запустить контейнеры
cd /data/coolify
docker-compose up -d
```

## Шаг 5: Открытие портов в firewall

```bash
# Открыть порт 8000 (Coolify Panel)
sudo iptables -I INPUT -p tcp --dport 8000 -j ACCEPT

# Открыть порт 6001 (WebSockets)
sudo iptables -I INPUT -p tcp --dport 6001 -j ACCEPT

# Сохранить правила (если поддерживается)
sudo iptables-save > /etc/iptables/rules.v4
```

## Шаг 6: Проверка статуса

```bash
# Проверить контейнеры
docker-compose ps

# Проверить логи
docker-compose logs -f coolify

# Проверить порты
ss -tlnp | grep -E ':(8000|6001)'
```

## Шаг 7: Сохранение бэкапа .env

**Критично**: Сохраните .env файл в надёжное место!

```bash
# Сохранить бэкап
cp /data/coolify/.env ~/coolify-backup-$(date +%F).env

# Показать содержимое для сохранения
cat /data/coolify/.env
```

## Критичные проблемы и решения

### Проблема 1: HTTP запросы таймаутятся извне

**Симптомы**:
- Локально curl работает
- TCP соединение проходит
- HTTP запросы таймаутятся

**Решение**:
1. Добавить `SSL_MODE=off` в .env файл
2. Перезапустить контейнер: `docker-compose restart coolify`
3. Проверить группу безопасности облачного провайдера

### Проблема 2: Конфликт Docker и k0s

**Симптомы**:
- Docker контейнеры не запускаются
- Ошибки containerd

**Решение**:
1. Удалить k0s: `k0sctl reset`
2. Перезагрузить сервер
3. Удалить старые контейнеры: `docker system prune -a`

### Проблема 3: high CPU потребление

**Симптомы**:
- Сервер тормозит
- high load average

**Решение**:
```bash
# Ограничить ресурсы контейнера Coolify
docker update --cpus=0.75 --memory=768m --memory-swap=768m coolify

# Добавить переменные для Horizon
docker exec coolify sh -c "echo 'HORIZON_BALANCE=10
HORIZON_MAX_PROCESSES=1
HORIZON_MIN_PROCESSES=1' >> /var/www/html/.env"

docker restart coolify
```

## Доступ после установки

- **URL**: http://<IP-СЕРВЕРА>:8000
- **First login**: Следуйте инструкциям на экране для создания первого администратора

## Безопасность

1. **Никогда не коммитьте .env в Git**
2. **Сохраните пароли в менеджере паролей**
3. **Используйте HTTPS в production** (настройте позже)
4. **Регулярно обновляйте**: `docker-compose pull && docker-compose up -d`

## Troubleshooting

### Контейнер не запускается
```bash
docker-compose logs coolify | grep -i error
docker-compose logs coolify-db
```

### Нет доступа к порту
```bash
sudo ss -tlnp | grep ':8000'
sudo iptables -L INPUT -n | grep 8000
```

### DNS не работает внутри контейнера
```bash
# Проверить resolv.conf
docker exec coolify cat /etc/resolv.conf

# Использовать bridge network (не host)
```

## Полезные команды

```bash
# Старт
docker-compose start

# Стоп
docker-compose stop

# Перезапуск
docker-compose restart

# Логи
docker-compose logs -f

# Обновление
docker-compose pull && docker-compose up -d

# Удаление (осторожно!)
docker-compose down
```

## Дополнительные ресурсы

- Официальная документация: https://coolify.io/docs
- GitHub: https://github.com/coollabsio/coolify
- Troubleshooting: https://coolify.io/docs/troubleshoot/overview