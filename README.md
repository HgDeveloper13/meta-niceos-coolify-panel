# Coolify Production Setup — НАЙС.ОС (NiceOS)

## 📋 Production-дополнения

Этот репозиторий содержит production-конфигурацию Coolify с минимальными, но критически важными дополнениями:

### ✅ Добавленные компоненты

1. **Traefik Proxy** — автоматический HTTPS через Let's Encrypt
2. **Healthcheck** — мониторинг состояния Coolify
3. **Production переменные** — оптимизация очередей и кэша

---

## 🚀 Быстрый старт

### 1. Подготовка на сервере

```bash
# Создать структуру папок
mkdir -p /data/coolify/{source,ssh/keys,applications,databases,backups,services,proxy,traefik}
mkdir -p /data/coolify-postgres
mkdir -p /data/coolify-redis
```

### 2. Сгенерировать ключи

```bash
cd /data/coolify

# Сгенерировать .env с реальными значениями
cat > .env << EOF
APP_NAME=coolify
APP_ENV=production
APP_DEBUG=false
APP_URL=http://$(hostname -I | awk '{print $1}'):8000
APP_HOST=0.0.0.0
APP_PORT=8000

APP_ID=$(openssl rand -hex 16)
APP_KEY=base64:$(openssl rand -base64 32)
ENCRYPTION_KEY=base64:$(openssl rand -base64 32)

DB_CONNECTION=pgsql
DB_HOST=coolify-db
DB_PORT=5432
DB_DATABASE=coolify
DB_USERNAME=coolify
DB_PASSWORD=$(openssl rand -base64 32)

REDIS_HOST=coolify-redis
REDIS_PORT=6379
REDIS_PASSWORD=$(openssl rand -base64 32)

PUSHER_APP_ID=$(openssl rand -hex 32)
PUSHER_APP_KEY=$(openssl rand -hex 32)
PUSHER_APP_SECRET=$(openssl rand -hex 32)
PUSHER_HOST=coolify-realtime
PUSHER_PORT=6001

DOCKER_HOST=unix:///var/run/docker.sock
REGISTRY_URL=ghcr.io
AUTOUPDATE=true

# === PRODUCTION OPTIMIZATION ===
HORIZON_BALANCE=10
HORIZON_MAX_PROCESSES=1
HORIZON_MIN_PROCESSES=1
QUEUE_CONNECTION=redis
CACHE_DRIVER=redis
SESSION_DRIVER=redis
LOG_CHANNEL=stderr
EOF

# Установить правильные права
chown -R 9999:root /data/coolify
chmod -R 700 /data/coolify
```

### 3. Запустить

```bash
cd /data/coolify
docker compose up -d

# Проверить статус
docker compose ps

# Посмотреть логи
docker compose logs -f coolify
```

### 4. Открыть порты (Nice OS, iptables)

```bash
# Открыть порты 80, 443, 8000
sudo iptables -I INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 443 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 8000 -j ACCEPT

# Проверить
sudo ss -tlnp | grep -E ':(80|443|8000)'

# Тест
curl -I http://127.0.0.1:8000
```

---

## 📊 Структура сервисов

```
┌─────────────────────────────────────────────────────────────┐
│                    Traefik (80/443)                         │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              Coolify Panel (8000)                      │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │  │
│  │  │  PostgreSQL  │  │    Redis     │  │  Real-time   │  │  │
│  │  │   (5432)     │  │   (6379)     │  │  (6001)      │  │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔧 Production переменные

| Переменная | Значение | Описание |
|------------|----------|----------|
| `HORIZON_BALANCE` | 10 | Балансировка очередей |
| `HORIZON_MAX_PROCESSES` | 1 | Максимум воркеров |
| `HORIZON_MIN_PROCESSES` | 1 | Минимум воркеров |
| `QUEUE_CONNECTION` | redis | Драйвер очереди |
| `CACHE_DRIVER` | redis | Драйвер кэша |
| `SESSION_DRIVER` | redis | Драйвер сессий |
| `LOG_CHANNEL` | stderr | Логирование в stdout |

---

## 🛡️ Безопасность

### Критичные файлы для бэкапа

```bash
# Сохранить .env в надёжное место!
cp /data/coolify/.env ~/coolify-backup-$(date +%F).env

# Полный бэкап данных
tar -czf coolify-backup-$(date +%F).tar.gz /data/coolify
tar -czf coolify-postgres-$(date +%F).tar.gz /data/coolify-postgres
tar -czf coolify-redis-$(date +%F).tar.gz /data/coolify-redis
```

### Права доступа

```bash
chown -R 9999:root /data/coolify
chmod -R 700 /data/coolify
```

---

## 📈 Мониторинг

### Проверка healthcheck

```bash
docker inspect --format='{{.State.Health.Status}}' coolify
```

### Логи

```bash
# Все сервисы
docker compose logs -f

# Только Coolify
docker compose logs -f coolify

# Только Traefik
docker compose logs -f traefik
```

### Ресурсы

```bash
docker stats
```

---

## 🔄 Обновление

```bash
cd /data/coolify
docker compose pull
docker compose up -d
```

---

## 🛑 Остановка

```bash
cd /data/coolify
docker compose down
```

---

## 🔍 Troubleshooting

### Coolify не запускается

```bash
# Проверить логи
docker compose logs coolify

# Проверить .env
cat /data/coolify/.env

# Проверить БД
docker compose logs coolify-db
```

### Нет доступа к порту 8000

```bash
# Проверить слушает ли порт
ss -tlnp | grep ':8000'

# Проверить firewall
sudo iptables -L INPUT -n | grep 8000

# Тест локально
curl -I http://127.0.0.1:8000
```

### Traefik не выдаёт сертификаты

```bash
# Проверить логи Traefik
docker compose logs traefik

# Проверить acme.json
ls -la /data/coolify/traefik/acme.json

# Проверить DNS
dig +short your-domain.com
```

---

## 📚 Дополнительная документация

- [Официальная документация Coolify](https://coolify.io/docs)
- [НАЙС.ОС — Установка Docker](https://z.niceos.ru/manual/ustanovka-i-nastrojka-docker)
- [НАЙС.ОС — Firewall](https://z.niceos.ru/manual/nastrojka-fajervola-firewalld-iptables)

---

## 📝 Версия

- **Дата:** 2026-03-04
- **Coolify:** latest
- **Traefik:** v3.0
- **PostgreSQL:** 15-alpine
- **Redis:** 7-alpine

---

## ✅ Чеклист установки

- [ ] Docker установлен и работает
- [ ] Создана структура `/data/coolify`
- [ ] Сгенерирован `.env` с уникальными ключами
- [ ] Установлены права (chown 9999:root)
- [ ] Запущено: `docker compose up -d`
- [ ] Открыт порт 8000 в firewall
- [ ] Открыт порт 80, 443 в firewall
- [ ] Проверен доступ: `curl http://IP:8000`
- [ ] Сохранён бэкап `.env`
- [ ] Проверен healthcheck Coolify