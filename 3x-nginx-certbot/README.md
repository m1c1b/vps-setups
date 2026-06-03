# 3x-ui + nginx-certbot

Стек: [3x-ui](https://github.com/MHSanaei/3x-ui) (XRay-панель) + nginx с автоматическим SSL от Let's Encrypt.

**Что делает каждый сервис:**
- `nginx` — терминирует TLS на порту 8443, проксирует `/sub/` и `/json/` в 3x-ui, отдаёт заглушку на `/`
- `3xui` — XRay-панель, работает в `network_mode: host`
- Сертификаты выпускаются через webroot-challenge и обновляются автоматически каждые 8 дней

---

## Требования

- VPS с публичным IP
- Домен, A-запись которого указывает на этот IP
- Docker + Docker Compose v2

---

## Развёртывание

### 1. Подготовить файлы конфига

Заменить `host-placeholder` на ваш домен в двух местах:

**`docker-compose.yml`** — поле `hostname`:
```yaml
hostname: your.domain.com
```

**`user_conf.d/host-placeholder.conf`** — переименовать файл и заменить все вхождения внутри:
```bash
mv user_conf.d/host-placeholder.conf user_conf.d/your.domain.com.conf
sed -i 's/host-placeholder/your.domain.com/g' user_conf.d/your.domain.com.conf
```

### 2. Указать email для certbot

В `docker-compose.yml`:
```yaml
- CERTBOT_EMAIL=your@email.com
```

### 3. Запустить стек

```bash
docker compose up -d
```

nginx сам выпустит сертификат при первом старте — это занимает ~30 секунд. Логи:
```bash
docker compose logs -f nginx
```

Дождаться строки `nginx: [notice] ... start worker processes`.

### 4. Настроить 3x-ui

Панель доступна на `http://<IP>:2053` (порт по умолчанию).

Войти (admin/admin), сразу сменить логин/пароль в Settings → User.

В Settings → Xray → подписка указать:
- **Port:** `2096`
- **Path prefix:** `/sub/`

### 5. Проверить

```bash
# TLS и заглушка
curl -I https://your.domain.com:8443

# Подписка (после создания inbound)
curl https://your.domain.com:8443/sub/<uuid>
```

---

## Порты

| Порт | Назначение |
|------|------------|
| 80   | HTTP (только для ACME-challenge и редирект на HTTPS) |
| 8443 | HTTPS — публичный вход для клиентов |
| 2053 | Панель 3x-ui (закрыть фаерволом, доступ только для вас) |
| 2096 | Подписки 3x-ui (только localhost, nginx проксирует) |

Закрыть панель от внешнего мира:
```bash
ufw allow 80/tcp
ufw allow 8443/tcp
ufw deny 2053/tcp
ufw enable
```

---

## Структура томов

```
3x-nginx-certbot/
├── docker-compose.yml
├── user_conf.d/          # nginx конфиги (один файл = один домен)
├── static/               # статика, отдаётся на /
├── nginx_secrets/        # сертификаты Let's Encrypt (gitignored)
└── 3x-ui-db/             # база данных 3x-ui (gitignored)
```

---

## Обновление образов

```bash
docker compose pull
docker compose up -d
```

---

## Частые проблемы

**Сертификат не выпускается** — проверить, что порт 80 открыт и A-запись домена уже проставлена. Посмотреть лог: `docker compose logs nginx | grep certbot`.

**3x-ui не стартует** — `network_mode: host` требует, чтобы порты 2053, 2096 и XRay-порты не были заняты на хосте.

**После смены домена** — удалить `nginx_secrets/` и перезапустить, certbot выпустит новый сертификат.
