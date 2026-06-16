+++
title = "Docker зсередини: containerd, runc, образи, мережі"
date = 2025-06-16
draft = false
tags = ["devops", "linux"]
categories = ["Infrastructure"]
description = "Повний розбір Docker-стеку: архітектура containerd + runc, шари образів, multi-stage build, Docker Compose, мережі, логування, troubleshooting — з поясненням кожного рядка конфігу."
+++

# Docker зсередини: containerd, runc, образи, мережі та все що треба знати на співбесіді

> Ця стаття — не «Hello World». Це розбір Docker-стеку від ядра Linux до `docker compose up`, з поясненням кожного рядка конфігу, каверзними питаннями зі співбесід і реальними прикладами з production.

---

## 1. Архітектура: що відбувається після `docker run`

Більшість думають, що Docker — це монолітний демон. Насправді це стек із кількох незалежних компонентів.

```
┌─────────────────────────────────────────────┐
│  docker CLI  (клієнт, просто шле HTTP запити)│
└────────────────────┬────────────────────────┘
                     │ Unix socket /var/run/docker.sock
                     ▼
┌─────────────────────────────────────────────┐
│  dockerd  (Docker daemon)                   │
│  ├─ Docker API (REST)                       │
│  ├─ image management                        │
│  ├─ volume management                       │
│  └─ network management                      │
└────────────────────┬────────────────────────┘
                     │ gRPC (CRI або внутрішній)
                     ▼
┌─────────────────────────────────────────────┐
│  containerd  (OCI-сумісний container runtime)│
│  ├─ image pull / push / store               │
│  ├─ snapshot management (overlayfs)         │
│  └─ task lifecycle (create/start/stop)      │
└────────────────────┬────────────────────────┘
                     │ OCI Runtime Spec (JSON)
                     ▼
┌─────────────────────────────────────────────┐
│  runc  (низькорівневий OCI runtime)         │
│  ├─ clone() системний виклик (namespaces)   │
│  ├─ cgroup limits                           │
│  └─ pivot_root / chroot                     │
└─────────────────────────────────────────────┘
```

### Ланцюжок викликів при `docker run`

1. `docker` CLI → HTTP POST `/containers/create` → `dockerd`
2. `dockerd` → gRPC `containerd.TaskService.Create` → `containerd`
3. `containerd` готує OCI bundle (rootfs + config.json) → запускає `runc`
4. `runc` виконує `clone()` з прапорами namespace, налаштовує cgroup, монтує overlayfs, запускає init-процес
5. `runc` виходить — контейнер живе самостійно під управлінням `containerd-shim`

> **Ключовий момент**: `runc` — це одноразовий процес. Він запускає контейнер і завершується. Далі процесом-нянею є `containerd-shim-runc-v2`, який зберігає stdio та exit code.

---

## 2. Linux-примітиви під капотом

Docker — це не «магія». Це зручний інтерфейс над двома фічами ядра Linux.

### Namespaces — ізоляція

| Namespace | Прапор clone() | Що ізолює |
|-----------|---------------|-----------|
| `pid`     | `CLONE_NEWPID` | дерево процесів (PID 1 всередині контейнера) |
| `net`     | `CLONE_NEWNET` | мережеві інтерфейси, routing table, netfilter |
| `mnt`     | `CLONE_NEWNS`  | точки монтування, filesystems |
| `uts`     | `CLONE_NEWUTS` | hostname, domainname |
| `ipc`     | `CLONE_NEWIPC` | System V IPC, POSIX message queues |
| `user`    | `CLONE_NEWUSER`| UID/GID маппінг (rootless containers) |
| `cgroup`  | `CLONE_NEWCGROUP`| cgroup root (з ядра 4.6) |

```bash
# Перевірити namespaces конкретного контейнера
CONTAINER_PID=$(docker inspect --format '{{.State.Pid}}' mycontainer)
ls -la /proc/$CONTAINER_PID/ns/
```

### Cgroups — обмеження ресурсів

```bash
# Де живуть cgroups контейнера (cgroup v2, systemd slice)
ls /sys/fs/cgroup/system.slice/docker-<ID>.scope/

# Подивитися ліміти CPU і пам'яті
cat /sys/fs/cgroup/system.slice/docker-<full_id>.scope/memory.max
cat /sys/fs/cgroup/system.slice/docker-<full_id>.scope/cpu.max
```

**Різниця між cgroup v1 і v2:**
- **v1**: ієрархії окремі для кожного контролера (`/sys/fs/cgroup/memory/`, `/sys/fs/cgroup/cpu/`)
- **v2**: єдина уніфікована ієрархія (`/sys/fs/cgroup/`), обов'язково з Docker 20.10+

```bash
# Перевірити яка версія cgroup використовується
stat -fc %T /sys/fs/cgroup/
# tmpfs → v1, cgroup2fs → v2
```

---

## 3. Образи, шари та overlayfs

### Що таке образ Docker

Образ — це **незмінний** (immutable) набір шарів файлової системи + метадані (manifest). Кожна інструкція `RUN`, `COPY`, `ADD` у Dockerfile створює новий шар.

```
┌──────────────────────────────┐  ← контейнерний шар (read-write)
│  container layer (writable)  │
├──────────────────────────────┤
│  layer N: RUN pip install    │  ← read-only
├──────────────────────────────┤
│  layer N-1: COPY . /app      │  ← read-only
├──────────────────────────────┤
│  layer 2: RUN apt-get update │  ← read-only
├──────────────────────────────┤
│  layer 1: FROM ubuntu:22.04  │  ← базовий образ (read-only)
└──────────────────────────────┘
```

### OverlayFS: як шари монтуються

```bash
# Подивитися mount point контейнера
docker inspect mycontainer | jq '.[0].GraphDriver'

# Приклад виводу:
{
  "Data": {
    "LowerDir": "/var/lib/docker/overlay2/abc.../diff:/var/lib/docker/overlay2/def.../diff",
    "MergedDir": "/var/lib/docker/overlay2/xyz.../merged",  # ← це і є / всередині контейнера
    "UpperDir": "/var/lib/docker/overlay2/xyz.../diff",     # ← writable шар
    "WorkDir": "/var/lib/docker/overlay2/xyz.../work"       # ← тимчасовий, потрібен overlayfs
  },
  "Name": "overlay2"
}
```

**Copy-on-Write (CoW)**: якщо контейнер змінює файл з read-only шару, overlayfs копіює його в `UpperDir`, і далі контейнер бачить змінену копію. Оригінал у `LowerDir` не змінюється.

```bash
# Знайти де фізично лежать шари
ls /var/lib/docker/overlay2/

# Подивитися вміст конкретного шару
ls /var/lib/docker/overlay2/<layer_id>/diff/
```

### Containerd image store (Docker 29+)

З Docker 29 (і Docker Desktop раніше) з'явився **containerd image store** — замість власного сховища образів Docker використовує containerd напряму.

```bash
# Увімкнути в /etc/docker/daemon.json
{
  "features": {
    "containerd-snapshotter": true
  }
}
```

**Що змінюється:**
- Образи тепер зберігаються в `/var/lib/containerd/`, а не `/var/lib/docker/image/`
- `docker images` і `ctr images list` показують одне й те саме
- Підтримка multi-platform образів (BuildKit + containerd)
- Snapshotter — `overlayfs` (раніше був окремий `overlay2` driver dockerd)

```bash
# Переглянути образи через containerd напряму
ctr --namespace moby images list

# Namespace docker-а в containerd завжди "moby"
ctr -n moby containers list

# Якщо використовуєте k8s — namespace "k8s.io"
ctr -n k8s.io images list
```

---

## 4. COPY vs ADD — коли що використовувати

Це класичне питання на співбесіді.

| | `COPY` | `ADD` |
|--|--------|-------|
| Копіює локальні файли | ✅ | ✅ |
| Розпаковує tar-архів | ❌ | ✅ автоматично |
| Завантажує з URL | ❌ | ✅ (але краще не треба) |
| Прозорість | ✅ завжди зрозуміло | ❌ поведінка неочевидна |
| Best practice | **Завжди використовуйте** | Тільки якщо потрібен auto-untar |

```dockerfile
# ✅ Правильно — COPY для звичайних файлів
COPY requirements.txt /app/requirements.txt
COPY --chown=appuser:appgroup ./src /app/src

# ✅ ADD виправдано — якщо потрібно розпакувати архів
ADD app-data.tar.gz /data/

# ❌ Неправильно — не використовуйте ADD для URL
ADD https://example.com/file.jar /app/  # краще: RUN curl -fsSL ... -o /app/file.jar
```

**Чому `ADD` з URL — погана ідея**: шар кешується після першого білда. Якщо файл на URL оновився, Docker цього не знає і використовує старий кеш.

---

## 5. Volumes: типи та коли що монтувати

### Три типи монтування

```
┌─────────────────┬──────────────────────────────────────────┐
│  Тип            │  Приклад                                 │
├─────────────────┼──────────────────────────────────────────┤
│  Named volume   │  -v mydata:/app/data                     │
│  Bind mount     │  -v /host/path:/container/path           │
│  tmpfs mount    │  --tmpfs /tmp:rw,size=100m               │
└─────────────────┴──────────────────────────────────────────┘
```

### Named volumes — для даних

```bash
docker volume create pgdata

docker run -d \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16
```

- Зберігаються в `/var/lib/docker/volumes/<name>/_data`
- Керуються Docker-ом, незалежні від lifecycle контейнера
- Підходять для: БД, черги, persistent state

```bash
# Inspect volume
docker volume inspect pgdata

# Backup volume
docker run --rm \
  -v pgdata:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/pgdata.tar.gz -C /data .
```

### Bind mounts — для розробки

```bash
docker run -d \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
  -v $(pwd)/html:/usr/share/nginx/html \
  nginx:alpine
```

- Монтується конкретний хост-шлях
- `:ro` — read-only всередині контейнера
- Підходить для: dev-режим, конфіги, code hot-reload

### Коли монтування файлу (не директорії) — пастка

```bash
# Це ПОГАНО якщо nginx.conf не існує на хості
-v /host/nginx.conf:/etc/nginx/nginx.conf

# Docker створить /host/nginx.conf як ДИРЕКТОРІЮ, не файл!
# Контейнер впаде з помилкою: IsADirectory
```

**Правило**: якщо монтуєте файл — файл повинен існувати на хості до запуску контейнера. Інакше Docker (bind mount) створить директорію з цим ім'ям.

### tmpfs — для секретів і тимчасових даних

```bash
docker run -d \
  --tmpfs /run:rw,noexec,nosuid,size=65536k \
  --tmpfs /tmp \
  myapp
```

- Зберігається в RAM, не записується на диск
- Зникає при зупинці контейнера
- Підходить для: session tokens, тимчасові файли, де важлива безпека

---

## 6. Multi-stage build: пояснення кожного рядка

Це архітектурний патерн, який зменшує розмір production-образу в 5-20 разів.

```dockerfile
# ─────────────────────────────────────────────
# STAGE 1: builder
# Іменуємо стейдж "builder" — можна посилатися на нього далі
# ─────────────────────────────────────────────
FROM golang:1.22-alpine AS builder

# Встановлюємо системні залежності для компіляції
# alpine не має bash і багатьох GNU-tools, тому apk
# --no-cache: не кешувати індекс пакетів — зменшує шар
RUN apk add --no-cache git ca-certificates tzdata

# Окремо копіюємо файли залежностей ДО копіювання коду
# Причина: кеш Docker. Якщо go.mod/go.sum не змінились —
# RUN go mod download буде взято з кешу, навіть якщо код змінився
WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download

# Тепер копіюємо весь код
# Цей шар інвалідується при будь-якій зміні коду
COPY . .

# Компілюємо статичний бінарник
# CGO_ENABLED=0: не використовувати C-бібліотеки (важливо для scratch/alpine)
# GOOS=linux: явно вказуємо target OS
# -ldflags="-w -s": прибираємо debug info та symbol table (менший бінарник)
# -a: перекомпілювати всі пакети (гарантія чистого білда)
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-w -s" \
    -a \
    -o /app/server \
    ./cmd/server

# ─────────────────────────────────────────────
# STAGE 2: production image
# scratch — абсолютно порожній образ, 0 байт
# Якщо потрібен shell для debug — використовуйте alpine замість scratch
# ─────────────────────────────────────────────
FROM scratch AS production

# Копіюємо SSL certificates зі stage builder
# Без них HTTPS-запити провалюються (x509: certificate signed by unknown authority)
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Копіюємо timezone data (потрібно якщо додаток парсить timezones)
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

# Копіюємо тільки скомпільований бінарник — більше нічого
COPY --from=builder /app/server /server

# Оголошуємо порт (документація, не відкриває порт)
EXPOSE 8080

# USER: не запускаємо від root
# scratch не має /etc/passwd — UID задаємо числом
USER 65534:65534

# ENTRYPOINT vs CMD:
# ENTRYPOINT — що запускається ЗАВЖДИ (не перевизначається без --entrypoint)
# CMD — аргументи за замовчуванням, які можна замінити при docker run
ENTRYPOINT ["/server"]
CMD ["--config", "/etc/app/config.yaml"]
```

### Multi-stage для Python додатка

```dockerfile
# STAGE 1: встановлення залежностей
FROM python:3.12-slim AS deps

# Окрема робоча директорія для залежностей
WORKDIR /deps

# Копіюємо тільки requirements — кеш шару залишиться валідним
# поки requirements.txt не зміниться
COPY requirements.txt .

# --no-cache-dir: не зберігати pip cache (непотрібний в образі)
# --user: встановлюємо в ~/.local (щоб легше скопіювати)
RUN pip install --no-cache-dir --user -r requirements.txt

# ─────────────────────────────────────────────
# STAGE 2: production
# ─────────────────────────────────────────────
FROM python:3.12-slim AS production

# Не генерувати .pyc файли при запуску (непотрібні в контейнері)
ENV PYTHONDONTWRITEBYTECODE=1

# Не буферизувати stdout/stderr — логи одразу видні в docker logs
ENV PYTHONUNBUFFERED=1

# Шлях де pip --user встановлює пакети
ENV PATH="/root/.local/bin:$PATH"

WORKDIR /app

# Копіюємо встановлені пакети зі stage deps
COPY --from=deps /root/.local /root/.local

# Копіюємо код додатка
COPY ./src /app/src

RUN addgroup --system appgroup && \
    adduser --system --ingroup appgroup appuser

USER appuser

CMD ["python", "-m", "uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Корисні BuildKit-фічі (Docker 23.0+ включено за замовчуванням)

```dockerfile
# syntax=docker/dockerfile:1

FROM ubuntu:22.04

# Cache mount: кешувати /var/cache/apt між білдами — не заново качати пакети
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Secret mount: пробрасуємо секрет під час білда, не залишаємо в шарі
# docker build --secret id=npmrc,src=$HOME/.npmrc .
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm install

# SSH mount: використовуємо SSH-агент для приватних git репозиторіїв
# docker build --ssh default .
RUN --mount=type=ssh \
    git clone git@github.com:myorg/private-repo.git /app
```

---

## 7. Docker Compose: повний розбір конфігу

```yaml
# compose.yaml (новий стандарт, замість docker-compose.yml)
# Версія spec більше не потрібна з Compose V2

name: myapp  # Префікс для імен мереж та volumes (замість назви директорії)

services:

  # ─────────────────────────────────────────────
  # PostgreSQL
  # ─────────────────────────────────────────────
  postgres:
    image: postgres:16-alpine
    
    # restart policy: що робити при падінні контейнера
    # unless-stopped: рестартувати завжди, крім явної зупинки (docker stop)
    # on-failure: тільки при non-zero exit code
    # always: навіть якщо явно зупинили (корисно для критичних сервісів)
    restart: unless-stopped
    
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: myuser
      # Краще використовувати secrets або env_file для паролів
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    
    # Секрети Docker (потрібно оголосити в секції secrets нижче)
    secrets:
      - db_password
    
    volumes:
      # Named volume для даних БД
      - pgdata:/var/lib/postgresql/data
      # Bind mount для init SQL скриптів (виконуються при першому старті)
      - ./db/init:/docker-entrypoint-initdb.d:ro
    
    # Обмеження ресурсів (замінює --memory та --cpus в docker run)
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 512M
    
    # Healthcheck: Compose чекає на healthy перед стартом залежних сервісів
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser -d mydb"]
      interval: 10s   # як часто перевіряти
      timeout: 5s     # timeout одної перевірки
      retries: 5      # скільки разів провалюватись перед unhealthy
      start_period: 30s  # не вважати failing під час initial startup
    
    networks:
      - backend

  # ─────────────────────────────────────────────
  # Backend API
  # ─────────────────────────────────────────────
  api:
    # Білдуємо з локального Dockerfile
    build:
      context: .        # директорія для build context (де шукати файли)
      dockerfile: Dockerfile  # ім'я Dockerfile (за замовчуванням "Dockerfile")
      target: production      # multi-stage: зупинитись на stage "production"
      args:
        # Build args передаються в Dockerfile як ARG
        APP_VERSION: ${APP_VERSION:-dev}
      # cache_from: прискорює CI/CD білди
      cache_from:
        - type=registry,ref=myregistry/myapp:cache
      labels:
        com.myapp.component: "api"
    
    # Залежності — визначає порядок запуску
    # depends_on з condition чекає на healthcheck
    depends_on:
      postgres:
        condition: service_healthy  # чекаємо поки postgres стане healthy
      redis:
        condition: service_started  # просто чекаємо старту
    
    environment:
      DATABASE_URL: postgresql://myuser@postgres:5432/mydb
      REDIS_URL: redis://redis:6379
    
    # env_file: завантажити змінні з файлу .env
    env_file:
      - .env
      - .env.local  # локальні overrides, не комітимо в git
    
    ports:
      # HOST:CONTAINER — пробрасуємо порт 8000 хосту на 8000 контейнера
      - "8000:8000"
      # Тільки localhost (безпечніше для internal сервісів)
      - "127.0.0.1:9090:9090"
    
    volumes:
      # Bind mount конфігу (read-only)
      - ./config:/etc/app/config:ro
      # Named volume для uploads
      - uploads:/app/uploads
    
    # Обмеження через ulimits (корисно для proxy-сервісів з many connections)
    ulimits:
      nofile:
        soft: 65536
        hard: 65536
    
    logging:
      driver: "json-file"       # драйвер логування (за замовчуванням)
      options:
        max-size: "50m"         # максимальний розмір одного log file
        max-file: "5"           # кількість ротованих файлів
        compress: "true"        # gzip стиснення старих файлів
    
    networks:
      - frontend
      - backend

  # ─────────────────────────────────────────────
  # Redis
  # ─────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: >
      redis-server
      --save 60 1
      --loglevel warning
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
    volumes:
      - redisdata:/data
    networks:
      - backend

  # ─────────────────────────────────────────────
  # Nginx reverse proxy
  # ─────────────────────────────────────────────
  nginx:
    image: nginx:1.27-alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - certbot_data:/etc/letsencrypt:ro
      - certbot_www:/var/www/certbot:ro
    depends_on:
      - api
    networks:
      - frontend

# ─────────────────────────────────────────────
# Volumes
# ─────────────────────────────────────────────
volumes:
  pgdata:
    # driver: local — за замовчуванням, зберігає в /var/lib/docker/volumes/
    driver: local
    labels:
      com.myapp.backup: "daily"
  redisdata:
  uploads:
  certbot_data:
  certbot_www:

# ─────────────────────────────────────────────
# Networks
# ─────────────────────────────────────────────
networks:
  frontend:
    # bridge — стандартна мережа, контейнери спілкуються через veth pairs
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24
  backend:
    driver: bridge
    # internal: true — мережа без виходу в інтернет (додатковий захист для БД)
    internal: true
    ipam:
      config:
        - subnet: 172.20.1.0/24

# ─────────────────────────────────────────────
# Secrets
# ─────────────────────────────────────────────
secrets:
  db_password:
    # file: читати секрет з файлу (не комітити в git!)
    file: ./secrets/db_password.txt
```

### Корисні команди Compose

```bash
# Запустити в background (-d) і збілдити якщо треба
docker compose up -d --build

# Переглянути логи з timestamps, follow
docker compose logs -f --timestamps api

# Запустити one-off команду в окремому контейнері (не заново запускати сервіс)
docker compose run --rm api python manage.py migrate

# Exec в запущений контейнер
docker compose exec api sh

# Scale — запустити 3 інстанси (потрібно прибрати fixed ports)
docker compose up -d --scale api=3

# Перевірити розгорнуту конфігурацію (з підстановкою змінних)
docker compose config

# Зупинити і видалити контейнери + мережі (volumes лишаються)
docker compose down

# Зупинити і видалити все включно з volumes (НЕБЕЗПЕЧНО — видаляє дані!)
docker compose down -v
```

---

## 8. Docker networking

### Типи мереж

```bash
docker network ls
# NETWORK ID   NAME      DRIVER    SCOPE
# a1b2c3d4e5f6 bridge    bridge    local   ← default для всіх контейнерів без --network
# b2c3d4e5f6a1 host      host      local   ← контейнер використовує мережу хосту напряму
# c3d4e5f6a1b2 none      null      local   ← немає мережі (ізольований контейнер)
```

### Bridge network: як це працює

```bash
# Подивитися деталі bridge мережі
docker network inspect bridge

# На хості з'являється veth pair:
ip link show type veth
# veth0abc@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> ← хостова сторона
# (парна ethX всередині контейнера)

# Перевірити bridge
brctl show docker0
# або
ip link show docker0
```

```bash
# Створити кастомну bridge мережу
docker network create \
  --driver bridge \
  --subnet 192.168.100.0/24 \
  --gateway 192.168.100.1 \
  --opt "com.docker.network.bridge.name=mybridge" \
  mynetwork

# Підключити контейнер до мережі (навіть запущений)
docker network connect mynetwork mycontainer

# Відключити
docker network disconnect mynetwork mycontainer
```

**Важлива різниця**: в default bridge мережі контейнери не резолвяться по імені. В user-defined bridge — так (вбудований DNS).

```bash
# В user-defined bridge: "postgres" резолвиться в IP контейнера
docker run --rm --network mynetwork alpine nslookup postgres

# В default bridge: тільки --link (deprecated) або IP вручну
```

### Host network

```bash
docker run --network host nginx
# nginx слухає на 0.0.0.0:80 ХОСТУ, без NAT
# Підходить для: максимальна мережева продуктивність, складна мережева конфігурація
# Мінус: немає ізоляції, конфлікти портів
```

### Macvlan: контейнер з real IP в LAN

```bash
docker network create \
  --driver macvlan \
  --subnet 192.168.1.0/24 \
  --gateway 192.168.1.1 \
  --opt parent=eth0 \
  macvlan_net

docker run --network macvlan_net --ip 192.168.1.200 nginx
# nginx доступний з будь-якого пристрою в LAN без port forwarding
```

---

## 9. `docker inspect` — твій кращий друг

```bash
# Вся інформація про контейнер
docker inspect mycontainer

# Тільки IP адреса
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' mycontainer

# Порти
docker inspect -f '{{json .NetworkSettings.Ports}}' mycontainer | jq

# PID процесу всередині контейнера (для nsenter, strace)
docker inspect -f '{{.State.Pid}}' mycontainer

# Точки монтування
docker inspect -f '{{json .Mounts}}' mycontainer | jq

# Environment variables
docker inspect -f '{{json .Config.Env}}' mycontainer | jq

# GraphDriver (overlayfs paths)
docker inspect -f '{{json .GraphDriver.Data}}' mycontainer | jq

# Ресурсні ліміти
docker inspect -f '{{.HostConfig.Memory}} {{.HostConfig.NanoCpus}}' mycontainer
```

---

## 10. Топ команди для роботи і діагностики

### Основні операції

```bash
# Список контейнерів: -a включає зупинені
docker ps -a

# Форматований вивід (корисно в скриптах)
docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}\t{{.Ports}}"

# Статистика ресурсів в реальному часі
docker stats
docker stats --no-stream  # one-shot

# Ресурси конкретного контейнера
docker stats mycontainer --no-stream --format "{{.CPUPerc}} {{.MemUsage}}"

# Зайнятий дисковий простір
docker system df
docker system df -v  # детально по кожному образу/контейнеру/volume

# Очищення (видаляє зупинені контейнери, dangling images, unused networks)
docker system prune

# Агресивне очищення (включає unused volumes та всі unused images)
docker system prune -af --volumes
```

### Image management

```bash
# Список образів
docker images
docker images --digests  # з SHA256 digest

# Dangling images (шари без тегу, залишки після rebuid)
docker images -f dangling=true

# Видалити всі dangling
docker image prune

# Переглянути шари образу (history)
docker history myimage:tag

# Детальна інформація про образ
docker manifest inspect myimage:tag

# Експорт образу в tar
docker save myimage:tag | gzip > myimage.tar.gz

# Імпорт
docker load < myimage.tar.gz

# Копіювати образ між registry без docker pull+push
# (потребує skopeo)
skopeo copy docker://source:tag docker://dest:tag
```

### Exec і debug

```bash
# Shell всередині контейнера
docker exec -it mycontainer sh
docker exec -it mycontainer bash

# Виконати команду і отримати вивід
docker exec mycontainer cat /etc/resolv.conf

# Запустити як root (якщо контейнер запущений під іншим user)
docker exec -u root -it mycontainer sh

# nsenter — ввійти в namespace процесу без docker exec
PID=$(docker inspect -f '{{.State.Pid}}' mycontainer)
nsenter --target $PID --mount --uts --ipc --net --pid

# strace процесу всередині контейнера
nsenter -t $PID -n -- strace -p 1

# tcpdump трафіку контейнера
nsenter -t $PID -n -- tcpdump -i eth0 -nn
```

### Copy files

```bash
# Скопіювати файл з контейнера
docker cp mycontainer:/etc/nginx/nginx.conf ./nginx.conf

# Скопіювати файл в контейнер
docker cp ./config.yaml mycontainer:/app/config.yaml
```

---

## 11. Логування: драйвери, JSON файли, ротація

### Драйвери логування

```bash
# Переглянути поточний logging driver
docker info | grep -i "logging driver"

# Логи конкретного контейнера
docker logs mycontainer
docker logs -f mycontainer            # follow
docker logs --tail 100 mycontainer    # останні 100 рядків
docker logs --since 1h mycontainer    # за останню годину
docker logs --timestamps mycontainer  # з timestamp
```

| Драйвер | Опис | Коли використовувати |
|---------|------|----------------------|
| `json-file` | JSON файли на хості (за замовчуванням) | dev, невеликий prod |
| `journald` | systemd journal | systemd-based хости |
| `syslog` | syslog/rsyslog | централізований syslog |
| `fluentd` | Fluentd collector | великий prod, EFK стек |
| `loki` | Grafana Loki | якщо вже є Prometheus/Grafana |
| `none` | вимкнути логування | для шумних batch jobs |
| `awslogs` | CloudWatch | AWS |
| `splunk` | Splunk HEC | enterprise |

### ⚠️ Критична проблема: json-file без ротації

За замовчуванням `json-file` драйвер **не ротує логи**. Контейнер може з'їсти весь диск.

```bash
# Перевірити розмір log файлів
find /var/lib/docker/containers -name "*.log" -exec ls -lh {} \;

# Небезпечна ситуація: 50GB log файл
ls -lh /var/lib/docker/containers/<id>/<id>-json.log
```

**Рішення 1: глобальна конфігурація** (рекомендовано)

```json
// /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "5",
    "compress": "true"
  }
}
```

```bash
# Застосувати
systemctl reload docker
# Увага: нові налаштування застосовуються тільки до нових контейнерів!
# Існуючі контейнери потрібно перезапустити.
```

**Рішення 2: на рівні контейнера**

```bash
docker run \
  --log-driver json-file \
  --log-opt max-size=50m \
  --log-opt max-file=5 \
  --log-opt compress=true \
  myapp
```

**Рішення 3: перемкнутись на journald** (якщо systemd)

```json
// /etc/docker/daemon.json
{
  "log-driver": "journald"
}
```

```bash
# Переглядати через journalctl
journalctl -u docker CONTAINER_NAME=mycontainer -f
journalctl CONTAINER_ID=abc123 --since "1 hour ago"
```

### Де фізично лежать JSON логи

```bash
# Знайти log file для контейнера
CONTAINER_ID=$(docker inspect -f '{{.Id}}' mycontainer)
ls /var/lib/docker/containers/$CONTAINER_ID/
# abc123...-json.log  ← сам лог
# config.v2.json      ← конфіг контейнера
# hostconfig.json     ← host config

# Читати лог напряму (корисно якщо docker logs повільний)
tail -f /var/lib/docker/containers/$CONTAINER_ID/*-json.log | \
  jq -r '.log'
```

### Структура json-file запису

```json
{
  "log": "2024-01-15T10:30:00Z [INFO] Server started on :8080\n",
  "stream": "stdout",
  "time": "2024-01-15T10:30:00.123456789Z"
}
```

---

## 12. Troubleshooting: containerd namespaces і cgroups

### Containerd напряму (bypass docker)

```bash
# Встановити ctr (зазвичай входить в пакет containerd)
which ctr

# Docker використовує namespace "moby"
ctr -n moby containers list
ctr -n moby images list
ctr -n moby tasks list

# Snapshots (шари)
ctr -n moby snapshots list

# Kubernetes використовує namespace "k8s.io"
ctr -n k8s.io containers list
```

### Перевірка cgroups контейнера

```bash
CONTAINER_ID=$(docker inspect -f '{{.Id}}' mycontainer)

# cgroup v2
CGROUP_PATH="/sys/fs/cgroup/system.slice/docker-${CONTAINER_ID}.scope"

# Поточне споживання пам'яті
cat $CGROUP_PATH/memory.current

# Ліміт пам'яті (max = немає ліміту)
cat $CGROUP_PATH/memory.max

# CPU quota (наприклад "200000 100000" = 2 core)
cat $CGROUP_PATH/cpu.max

# Статистика CPU
cat $CGROUP_PATH/cpu.stat

# Перевірити чи спрацьовував OOM killer
cat $CGROUP_PATH/memory.events
# oom_kill 3 ← контейнер вбивали OOM 3 рази!
```

### OOM kill діагностика

```bash
# Чи вбивали контейнер по OOM
docker inspect mycontainer | jq '.[0].State.OOMKilled'

# Системний OOM log
dmesg | grep -i "out of memory"
journalctl -k | grep -i oom
```

### Перевірка мережі всередині контейнера (без ip/netstat)

```bash
PID=$(docker inspect -f '{{.State.Pid}}' mycontainer)

# Мережеві інтерфейси
nsenter -t $PID -n ip addr

# Routing table
nsenter -t $PID -n ip route

# Відкриті порти (якщо ss доступний)
nsenter -t $PID -n ss -tlnp

# DNS резолюція
nsenter -t $PID -n cat /etc/resolv.conf
```

### Containerd snapshotter troubleshooting

```bash
# Перевірити який snapshotter використовується
docker info | grep "Storage Driver"

# Якщо overlayfs не працює (наприклад, btrfs filesystem)
# /etc/docker/daemon.json
{
  "storage-driver": "overlay2"
}

# Перевірити підтримку overlay
cat /proc/filesystems | grep overlay

# Перевірити d_type (потрібно для overlay2)
xfs_info /var/lib/docker | grep ftype
# ftype=1 ← OK
# ftype=0 ← overlay2 не підтримується
```

---

## 13. daemon.json: повна продакшн конфігурація

```json
// /etc/docker/daemon.json
{
  // Containerd image store (Docker 25+, стабільно з 26+)
  "features": {
    "containerd-snapshotter": true
  },

  // Storage driver
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],

  // Логування за замовчуванням для всіх контейнерів
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "5",
    "compress": "true"
  },

  // DNS для контейнерів (якщо хостовий /etc/resolv.conf нестабільний)
  "dns": ["1.1.1.1", "8.8.8.8"],

  // Registry mirrors (якщо є Nexus/Harbor або проксі)
  "registry-mirrors": ["https://registry.mycompany.com"],

  // Insecure registries (для внутрішніх без TLS)
  "insecure-registries": ["registry.internal:5000"],

  // Live restore: контейнери продовжують роботу при рестарті dockerd
  "live-restore": true,

  // User namespace remapping (безпека — root в контейнері = unprivileged на хості)
  "userns-remap": "default",

  // Обмеження ресурсів за замовчуванням
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 65536,
      "Soft": 65536
    }
  },

  // Метрики для Prometheus
  "metrics-addr": "127.0.0.1:9323",
  "experimental": false,

  // Cgroup driver (важливо для Kubernetes — має співпадати з kubelet)
  "exec-opts": ["native.cgroupdriver=systemd"],

  // Subnet для default bridge
  "bip": "172.17.0.1/16",

  // Не дозволяти контейнерам отримувати нові привілеї через setuid
  "no-new-privileges": true
}
```

---

## 14. Плюси і мінуси Docker

### Плюси

- **Портативність**: один і той самий образ працює скрізь (dev, staging, prod)
- **Ізоляція**: залежності не конфліктують між проектами
- **Швидкий деплой**: запуск контейнера — секунди (на відміну від VM)
- **Layer caching**: повторні білди швидкі якщо залежності не змінились
- **Ecosytem**: Docker Hub, Docker Compose, Swarm, підтримка всіх CI/CD
- **Resource efficiency**: на відміну від VM, не тягне повне ядро

### Мінуси і пастки

- **Security**: shared kernel — якщо вдалось вийти з контейнера, ти на хості. VM дає кращу ізоляцію
- **Persistent storage**: stateful застосунки (БД) складніше керувати ніж на голому металі
- **Networking complexity**: overlay, NAT, port mapping — є overhead і складність troubleshoot
- **Bloat**: образи часто набувають 1-2GB через неправильний Dockerfile
- **Log rotation**: забудеш налаштувати — з'їсть диск
- **PID 1 проблема**: якщо ваш процес не обробляє SIGTERM — контейнер не зупиняється коректно
- **Secret management**: env vars видні через `docker inspect`, потрібен Vault або Docker secrets
- **Live restore обмеження**: `--live-restore` не підтримує user namespaces

---

## 15. Каверзні питання на співбесіді

**Q: Чим відрізняється `ENTRYPOINT` від `CMD`?**

`ENTRYPOINT` — незмінна точка входу (не можна переписати без `--entrypoint`). `CMD` — аргументи за замовчуванням для `ENTRYPOINT`, або команда якщо `ENTRYPOINT` не задано. При `docker run myimage custom_arg` — `custom_arg` замінює `CMD`, але не `ENTRYPOINT`.

```dockerfile
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8080"]
# docker run myimage → python app.py --port 8080
# docker run myimage --port 9090 → python app.py --port 9090
```

---

**Q: Чому контейнер не зупиняється по `docker stop`?**

`docker stop` спочатку шле `SIGTERM` процесу PID 1, чекає 10 секунд, потім `SIGKILL`. Якщо ваш процес запущений через shell (наприклад `CMD ["sh", "-c", "python app.py"]`), то `sh` отримує `SIGTERM`, але не форвардить його в дочірній процес.

**Рішення**: використовуйте exec form, а не shell form:

```dockerfile
# Shell form — BAD, sh не форвардить сигнали
CMD python app.py

# Exec form — GOOD, SIGTERM іде напряму в python
CMD ["python", "app.py"]
```

---

**Q: Що відбувається з даними у volume якщо видалити контейнер?**

Named volumes — залишаються. Bind mounts — залишаються (це файли хосту). Anonymous volumes (`-v /data` без імені) — видаляються разом з контейнером якщо `docker rm -v`.

---

**Q: Як зменшити розмір образу?**

1. Використовувати multi-stage build
2. Base image `alpine` або `distroless` або `scratch`
3. Об'єднати `RUN` команди через `&&` (менше шарів)
4. Видаляти кеш пакетних менеджерів в тому ж шарі: `apt-get clean && rm -rf /var/lib/apt/lists/*`
5. `.dockerignore` — не копіювати непотрібне в build context

---

**Q: Чим відрізняється `docker exec` від `docker attach`?**

`docker exec` запускає **новий** процес всередині контейнера. `docker attach` приєднується до **існуючого** PID 1 (stdin/stdout/stderr). `Ctrl+C` в `attach` може зупинити контейнер! Для інтерактивної роботи завжди використовуй `exec`.

---

**Q: Контейнер постійно рестартує. Як дебажити?**

```bash
# Переглянути чому впав
docker logs mycontainer --tail 50

# Exit code
docker inspect mycontainer | jq '.[0].State.ExitCode'
# 0 → нормальне завершення
# 1 → помилка застосунку
# 137 → SIGKILL (OOM або docker kill)
# 143 → SIGTERM (docker stop)

# Запустити з override command для дебагу
docker run --rm -it --entrypoint sh myimage
```

---

**Q: В чому різниця між `containerd` і `docker`?**

`containerd` — OCI-сумісний container runtime, керує lifecycle контейнерів і образів. `dockerd` — надбудова над containerd, додає Docker API, build (через BuildKit), Compose-сумісність, volume plugins. Kubernetes може використовувати containerd напряму через CRI, без docker взагалі.

---

**Q: Що таке distroless образи і навіщо?**

Образи від Google без shell, пакетного менеджера і більшості системних утиліт. Тільки runtime потрібний застосунку.

```dockerfile
FROM gcr.io/distroless/java17-debian12
COPY --from=builder /app/app.jar /app/app.jar
CMD ["/app/app.jar"]
```

Плюси: менший attack surface, менший розмір. Мінус: неможливо `docker exec` та дебажити (рішення — debug варіант образу: `:debug` тег містить busybox).

---

**Q: Як правильно передати секрети в контейнер?**

```bash
# Погано: видно в docker inspect та docker history
ENV DB_PASSWORD=secret123

# Погано: теж видно в env
docker run -e DB_PASSWORD=secret123 myapp

# Краще: читати з файлу/secret на runtime
docker run -v /run/secrets/db_pass:/run/secrets/db_pass myapp
# Застосунок читає /run/secrets/db_pass самостійно

# Docker secrets (тільки в Swarm mode)
echo "secret123" | docker secret create db_password -
docker service create --secret db_password myapp

# Production: Vault Agent Sidecar або External Secrets Operator (k8s)
```

---

## Висновок

Docker — це не просто `docker run`. Щоб розуміти що відбувається, потрібно знати:

- **containerd + runc** — реальні виконавці, dockerd — тільки фронтенд
- **namespaces + cgroups** — ізоляція і ресурси на рівні ядра Linux
- **overlayfs** — як шари образів монтуються і чому CoW важливий для продуктивності
- **multi-stage build** — обов'язковий патерн для production образів
- **logging** — json-file без ротації з'їсть диск, налаштовуйте одразу
- **containerd image store** — майбутнє Docker, вже стандарт у новому рантаймі

Для production: `live-restore: true`, ротація логів глобально, `no-new-privileges: true`, і ніколи не запускайте контейнери без resource limits якщо це не ізольоване середовище.
