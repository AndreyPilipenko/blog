+++
title = "Nginx від А до Я: архітектура, різниця з Apache, location, proxy, балансування, TLS і траблшутінг"
date = 2026-10-10T12:00:00+03:00
draft = false
tags = ["nginx", "apache", "linux"]
categories = ["Infrastructure"]
description = "Повний розбір nginx для DevOps/SRE: чим nginx принципово відрізняється від Apache, як влаштовані master/worker і event loop, як nginx обирає server і location, reverse proxy, балансування, TLS, кешування, rate limiting, логи, траблшутінг."
+++

# Nginx від А до Я: все, що треба знати DevOps-інженеру

> Nginx стоїть майже перед кожним продакшеном: як web-сервер для статики, як reverse proxy перед застосунком, як балансувальник, як TLS-термінатор, як кеш. Ця стаття — не «встав конфіг зі StackOverflow», а пояснення **чому** nginx працює саме так, з типовими граблями, на які наступає кожен.

---

## 1. Що таке nginx і звідки він взявся

Nginx з'явився як відповідь на **проблему C10k** — як одному серверу одночасно обслуговувати 10 000+ з'єднань. Класичні сервери того часу (і Apache в першу чергу) виділяли **окремий процес або потік на кожне з'єднання**. 10 000 з'єднань = 10 000 процесів/потоків = гігабайти пам'яті та постійне перемикання контексту. Nginx вирішив це іншою архітектурою — асинхронним event loop, про який нижче.

Що варто знати про екосистему сьогодні:

| Що | Коротко |
|---|---|
| **nginx (open source)** | Основна безкоштовна версія, nginx.org. Гілки **mainline** (непарна мінорна версія, нові фічі) і **stable** (парна, тільки багфікси) |
| **NGINX Plus** | Комерційна версія від F5 (купила NGINX Inc. у 2019): активні health checks, API для динамічної зміни upstream, розширений моніторинг, sticky sessions з коробки |
| **OpenResty** | nginx + LuaJIT — можна писати логіку на Lua прямо в конфігу |
| **Angie, freenginx** | Форки nginx від колишніх розробників ядра nginx |
| **ingress-nginx** | Ingress-контролер Kubernetes на базі nginx. Community-проєкт `kubernetes/ingress-nginx` оголошено retired (підтримку завершено в березні 2026) — для нових кластерів дивіться в бік Gateway API. Не плутати з **NGINX Ingress Controller** від F5 — це окремий продукт |

> **Mainline vs stable:** попри назву, команда nginx сама рекомендує mainline для більшості випадків — він не менш стабільний, просто отримує фікси й фічі раніше. Stable — для тих, хто хоче мінімум змін.

---

## 2. Nginx vs Apache — головна різниця

Якщо треба відповісти одним реченням:

> **Apache виділяє процес або потік на з'єднання (у кращому випадку — на активний запит) і вміє виконувати код застосунку всередині себе. Nginx обробляє тисячі з'єднань у кількох однопотокових worker-процесах через event loop і принципово не виконує код застосунку — він передає запит далі (FastCGI, HTTP proxy, uWSGI).**

Усе інше — наслідки цієї різниці.

### 2.1 Архітектура обробки з'єднань

**Apache** має модель MPM (Multi-Processing Module), яку обирають при запуску:

```
Apache prefork MPM                     Apache worker / event MPM
(процес на з'єднання)                  (потік на з'єднання / на запит)

┌──────────── master ────────────┐     ┌──────────── master ───────────┐
│                                │     │                               │
├─ child process ── client 1     │     ├─ child process                │
├─ child process ── client 2     │     │    ├─ thread ── client 1      │
├─ child process ── client 3     │     │    ├─ thread ── client 2      │
├─ child process ── client 4     │     │    └─ thread ── client 3      │
│   ... 1 процес = 1 з'єднання   │     ├─ child process                │
│   (~10-50 МБ RAM кожен з       │     │    ├─ thread ── client 4      │
│    mod_php)                    │     │    └─ ...                     │
└────────────────────────────────┘     └───────────────────────────────┘
```

- **prefork** — один процес на з'єднання. Найважча модель, але потрібна для не thread-safe модулів (класичний `mod_php`).
- **worker** — кілька процесів, у кожному багато потоків, потік на з'єднання.
- **event** (за замовчуванням у більшості сучасних дистрибутивів) — як worker, але idle keepalive-з'єднання тримає окремий listener-потік, тож потік зайнятий тільки поки йде активний запит. Це вже набагато ближче до nginx, але все одно «потік на активний запит».

**Nginx**:

```
                 ┌─────────────── master process (root) ───────────────┐
                 │ читає конфіг, відкриває порти 80/443, керує worker'ами│
                 └───────┬───────────────┬───────────────┬─────────────┘
                         │               │               │
                 ┌───────▼──────┐ ┌──────▼───────┐ ┌─────▼────────┐
                 │  worker #1   │ │  worker #2   │ │  worker #N   │  N = кількість ядер
                 │ (www-data)   │ │ (www-data)   │ │ (www-data)   │
                 │              │ │              │ │              │
                 │  event loop  │ │  event loop  │ │  event loop  │
                 │  epoll()     │ │  epoll()     │ │  epoll()     │
                 │  ├ conn 1    │ │  ├ conn 1    │ │  ...         │
                 │  ├ conn 2    │ │  ├ conn 2    │ │              │
                 │  ├ ...       │ │  ├ ...       │ │              │
                 │  └ conn 4096 │ │  └ conn 4096 │ │              │
                 └──────────────┘ └──────────────┘ └──────────────┘
```

Кожен worker — **один потік**, який у циклі питає ядро через `epoll` (Linux) / `kqueue` (BSD): «на яких сокетах з'явились події?» — і обробляє лише ті, що готові. Поки один клієнт повільно завантажує файл через мобільний інтернет, worker не чекає на нього, а обслуговує інших. Idle-з'єднання коштує кілька сотень байт — кілобайти пам'яті, а не окремий потік зі стеком.

### 2.2 Динамічний контент (PHP, Python...)

| | Apache | Nginx |
|---|---|---|
| PHP | Може виконувати **всередині себе** через `mod_php` (або через PHP-FPM) | **Тільки** зовнішній процес: PHP-FPM через FastCGI |
| Python | `mod_wsgi` всередині Apache | Gunicorn / uWSGI / Uvicorn за `proxy_pass` або `uwsgi_pass` |

Наслідок: з `mod_php` кожен процес Apache «товстий» навіть тоді, коли віддає картинку на 2 КБ. Nginx віддає статику сам, а важку динаміку — окремому пулу процесів, який масштабується незалежно.

### 2.3 Конфігурація: `.htaccess` vs централізований конфіг

**Apache** підтримує `.htaccess` — файли конфігурації в каталогах сайту. Зручно для shared hosting (клієнт сам міняє rewrite-правила без root), але дорого: при `AllowOverride All` на **кожен запит** Apache перевіряє наявність `.htaccess` у кожному каталозі шляху. Запит `/a/b/c/file.jpg` = перевірка `/.htaccess`, `/a/.htaccess`, `/a/b/.htaccess`, `/a/b/c/.htaccess`.

**Nginx** не має аналога `.htaccess` взагалі. Весь конфіг читається один раз при старті/reload і тримається в пам'яті у вигляді дерева. Будь-яка зміна = правка конфігу + `nginx -t` + reload. Для shared hosting це мінус, для продакшену з Ansible/GitOps — плюс.

### 2.4 Модулі

- **Apache** — модулі динамічно вмикаються `a2enmod` / `LoadModule` без перекомпіляції, їх сотні.
- **Nginx** — історично модулі вкомпільовувались при збірці (`./configure --with-...`). З версії 1.9.11 є dynamic modules (`load_module`), але модуль має бути зібраний під конкретну версію nginx. Що вкомпільовано — дивіться через `nginx -V`.

### 2.5 Зведена таблиця

| Критерій | Apache httpd | Nginx |
|---|---|---|
| Модель | Процес/потік на з'єднання (prefork/worker), потік на запит (event) | Асинхронний event loop, кілька worker'ів |
| Пам'ять при 10k з'єднань | Сотні МБ — гігабайти | Десятки МБ |
| Статика | Добре | Дуже добре (`sendfile`, мінімум оверхеду) |
| Динаміка | Може всередині (`mod_php`, `mod_wsgi`) | Тільки через зовнішній процес |
| Конфіг per-directory | `.htaccess` | Немає, тільки центральний конфіг |
| Rewrite | `mod_rewrite` (потужний, складний) | `rewrite`, `return`, `try_files` — простіше |
| Reverse proxy / LB | Є (`mod_proxy`, `mod_proxy_balancer`) | Основний сценарій використання |
| Модулі | Динамічні, дуже багато | Менше, частина лише при збірці |
| Типова роль сьогодні | Legacy, shared hosting, застосунки, які покладаються на `.htaccess` | Edge/reverse proxy, статика, LB, TLS termination, Ingress |

### 2.6 Коли що обирати

- **Nginx** — reverse proxy перед застосунками (Node, Go, Python, Java), статика/CDN-origin, балансування, TLS termination, високе число одночасних з'єднань, WebSocket.
- **Apache** — legacy-застосунки, що залежать від `.htaccess` і `mod_rewrite` (старі WordPress-плагіни, CMS на shared hosting), специфічні модулі (наприклад, певні схеми автентифікації), коли команда його добре знає.
- **Обидва разом** — класична схема: nginx на 80/443 віддає статику, тримає TLS і повільних клієнтів, а Apache на `127.0.0.1:8080` обробляє PHP зі своїми `.htaccess`. Nginx тут «щит»: повільний клієнт тримає дешеве з'єднання nginx, а не дорогий процес Apache.

```
Client ──HTTPS──▶ nginx :443 ─┬─ /static/*  ──▶ диск (sendfile)
                              └─ /*         ──HTTP──▶ Apache 127.0.0.1:8080 ──▶ mod_php
```

---

## 3. Як nginx працює всередині

### 3.1 Процеси

```bash
$ ps -ef --forest | grep [n]ginx
root      1201     1  nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
www-data  1202  1201  nginx: worker process
www-data  1203  1201  nginx: worker process
www-data  1204  1201  nginx: cache manager process
```

- **master** — працює від root (щоб відкрити порти < 1024 і читати ключі TLS), читає конфіг, запускає/перезапускає worker'ів. Запитів не обробляє.
- **worker** — від непривілейованого користувача (`user www-data;`), обробляють усі з'єднання.
- **cache manager / cache loader** — з'являються, якщо налаштовано `proxy_cache_path`: стежать за розміром кешу і завантажують його метадані після старту.

### 3.2 Скільки клієнтів витримає nginx

```nginx
worker_processes auto;          # = кількість ядер CPU
worker_rlimit_nofile 65535;     # ліміт відкритих файлів на worker

events {
    worker_connections 4096;    # макс. з'єднань на ОДИН worker
    # use epoll;                # на Linux обирається автоматично
    # multi_accept on;          # приймати всі нові з'єднання за раз
}
```

Формула:

```
max_clients = worker_processes × worker_connections          # web-сервер
max_clients = worker_processes × worker_connections / 2      # reverse proxy
```

Чому `/2` для проксі: кожен клієнт займає **два** з'єднання — від клієнта до nginx і від nginx до upstream. І пам'ятайте, що `worker_connections` не може бути більшим за ліміт файлових дескрипторів — звідси `worker_rlimit_nofile` (або `LimitNOFILE` у systemd unit).

### 3.3 Що відбувається при `nginx -s reload`

Reload в nginx **graceful** — жодне з'єднання не рветься:

1. Master отримує `SIGHUP`.
2. Перечитує й перевіряє конфіг. Якщо є помилка — лишається на старому, пише помилку в error log.
3. Відкриває нові listen-сокети (якщо змінились).
4. Запускає **нових** worker'ів з новим конфігом.
5. Старим worker'ам надсилає команду graceful shutdown: перестати приймати нові з'єднання, дообробити поточні й завершитись.

Тому після reload `ps` може деякий час показувати `nginx: worker process is shutting down` — це нормально, вони дочікуються довгих запитів (наприклад, WebSocket або великих завантажень).

Сигнали master-процесу:

| Сигнал | Команда | Дія |
|---|---|---|
| `HUP` | `nginx -s reload` | Перечитати конфіг (graceful) |
| `QUIT` | `nginx -s quit` | Graceful shutdown |
| `TERM` / `INT` | `nginx -s stop` | Швидка зупинка |
| `USR1` | `nginx -s reopen` | Перевідкрити лог-файли (використовує logrotate) |
| `USR2` | — | Оновлення бінарника на льоту без даунтайму |

---

## 4. Структура конфігурації

### 4.1 Де що лежить

```
/etc/nginx/
├── nginx.conf              # головний файл
├── conf.d/*.conf           # додаткові конфіги (стиль RHEL і офіційних пакетів nginx.org)
├── sites-available/        # віртуальні хости (стиль Debian/Ubuntu)
├── sites-enabled/          # симлінки на sites-available — те, що реально увімкнено
├── snippets/               # шматки конфігу для include
├── mime.types              # відповідність розширень і Content-Type
└── fastcgi_params          # стандартні параметри для FastCGI

/var/log/nginx/access.log, error.log
/usr/share/nginx/html/ або /var/www/html/   # дефолтний root
```

> На Debian/Ubuntu увімкнути сайт = `ln -s /etc/nginx/sites-available/site /etc/nginx/sites-enabled/`. Класична помилка — забути, що `default` у `sites-enabled` теж слухає 80 порт з `default_server`.

### 4.2 Контексти (блоки)

```nginx
# ── main context ───────────────────────────────
user www-data;
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log warn;

events {                                  # ── events: як обробляти з'єднання
    worker_connections 4096;
}

http {                                    # ── http: все, що стосується HTTP
    include       mime.types;
    default_type  application/octet-stream;
    sendfile      on;

    upstream app_backend {                # ── upstream: група бекендів
        server 10.0.0.11:8000;
        server 10.0.0.12:8000;
    }

    server {                              # ── server: віртуальний хост
        listen 80;
        server_name example.com;

        location / {                      # ── location: правила для URI
            proxy_pass http://app_backend;
        }
    }
}

stream {                                  # ── stream: TCP/UDP-проксі (L4)
    server {
        listen 3306;
        proxy_pass 10.0.0.20:3306;
    }
}
```

### 4.3 Успадкування директив — головна пастка

Директиви успадковуються згори вниз: `http` → `server` → `location`. Але для **масивних** директив (тих, що можна вказати кілька разів: `add_header`, `proxy_set_header`, `fastcgi_param`) працює правило:

> **Якщо на поточному рівні є хоча б одна така директива — всі такі директиви з батьківського рівня ігноруються.**

```nginx
server {
    add_header X-Frame-Options "DENY";
    add_header Strict-Transport-Security "max-age=31536000";

    location /api/ {
        add_header X-API-Version "2";
        # ⚠ Тут НЕМАЄ X-Frame-Options і HSTS!
        # Вони не «додаються», а повністю замінюються.
    }
}
```

Рішення — винести спільні заголовки в `snippets/security-headers.conf` і робити `include` у кожному location, де є власні `add_header`.

Ще одна особливість `add_header`: заголовок додається тільки для відповідей 200, 201, 204, 206, 301, 302, 303, 304, 307, 308. Щоб додати його й до 4xx/5xx — параметр `always`:

```nginx
add_header X-Request-ID $request_id always;
```

---

## 5. Як nginx обирає `server` (virtual host)

Коли приходить запит, nginx у два кроки визначає, який `server` блок його обробить.

**Крок 1 — за `listen`.** Беруться тільки ті server-блоки, чий `listen` відповідає IP:port, на який прийшло з'єднання.

**Крок 2 — за `server_name`** (порівнюється із заголовком `Host`), у такому порядку пріоритету:

1. Точне ім'я — `server_name example.com;`
2. Найдовший wildcard на початку — `server_name *.example.com;`
3. Найдовший wildcard у кінці — `server_name mail.*;`
4. Перший regex, що збігся, у порядку появи в конфігу — `server_name ~^(?<sub>.+)\.example\.com$;`

Якщо нічого не збіглось — запит іде в **`default_server`** для цього `listen`. Якщо `default_server` явно не вказано — у **перший** server-блок з таким `listen` (а «перший» залежить від порядку `include`, тобто фактично від алфавіту файлів — джерело дуже дивних багів).

Добра практика — явний catch-all, який відкидає запити з невідомим Host (сканери, запити по IP):

```nginx
server {
    listen 80 default_server;
    listen 443 ssl default_server;
    server_name _;
    ssl_reject_handshake on;   # для HTTPS: не віддавати жоден сертифікат (nginx 1.19.4+)
    return 444;                # закрити з'єднання без відповіді (nginx-специфічний код)
}
```

> `server_name _;` — не магія, а просто невалідне ім'я, яке ніколи не збіжеться. Catch-all працює саме завдяки `default_server`.

---

## 6. `location` — найважливіша тема

### 6.1 Типи location

| Синтаксис | Тип | Приклад |
|---|---|---|
| `location = /path` | Точний збіг | `location = /favicon.ico` |
| `location ^~ /path` | Префікс, **без перевірки regex** | `location ^~ /static/` |
| `location ~ regex` | Regex, чутливий до регістру | `location ~ \.php$` |
| `location ~* regex` | Regex, нечутливий до регістру | `location ~* \.(jpg\|png\|css)$` |
| `location /path` | Звичайний префікс | `location /api/` |
| `location @name` | Named location (тільки для внутрішніх переходів) | `location @backend` |

### 6.2 Алгоритм вибору

Це найчастіше питання на співбесіді по nginx:

```
1. Шукаємо точний збіг  location = /uri
   └─ знайдено → ВИКОРИСТАТИ, стоп.

2. Перебираємо всі префіксні location, запам'ятовуємо НАЙДОВШИЙ збіг.
   └─ якщо найдовший має модифікатор ^~ → ВИКОРИСТАТИ, стоп.

3. Перевіряємо regex-location (~ і ~*) по черзі В ПОРЯДКУ ПОЯВИ в конфігу.
   └─ перший, що збігся → ВИКОРИСТАТИ, стоп.

4. Жоден regex не збігся → використати найдовший префікс з кроку 2.
```

Важливо: **порядок префіксних location у файлі не має значення** (виграє найдовший), а **порядок regex — має** (виграє перший).

### 6.3 Приклад

```nginx
location = /           { return 200 "A: exact /\n"; }
location /             { return 200 "B: prefix /\n"; }
location /images/      { return 200 "C: prefix /images/\n"; }
location ^~ /static/   { return 200 "D: ^~ /static/\n"; }
location ~* \.(png|jpg)$ { return 200 "E: regex image\n"; }
```

| Запит | Результат | Чому |
|---|---|---|
| `/` | **A** | Точний збіг |
| `/index.html` | **B** | Тільки префікс `/` |
| `/images/` | **C** | Найдовший префікс, regex не збігся |
| `/images/cat.png` | **E** | Найдовший префікс C без `^~`, тому перевіряються regex — E збігся і виграв |
| `/static/logo.png` | **D** | Найдовший префікс має `^~` — regex не перевіряються взагалі |
| `/docs/a.JPG` | **E** | `~*` нечутливий до регістру |

### 6.4 `root` vs `alias`

```nginx
# root — ДОДАЄ повний URI до шляху
location /images/ {
    root /var/www;
}
# /images/cat.png → /var/www/images/cat.png

# alias — ЗАМІНЮЄ частину URI, що збіглась з location
location /images/ {
    alias /data/pictures/;
}
# /images/cat.png → /data/pictures/cat.png
```

Граблі з `alias`: слеші мають збігатися. `location /images/` + `alias /data/pictures` (без слеша) дасть `/data/picturescat.png`. А `location /images` (без слеша) + `alias /data/pictures/` відкриває path traversal: `/images../etc/passwd` → `/data/pictures/../etc/passwd`. Правило: **або обидва зі слешем у кінці, або обидва без**, а ще краще — використовуйте `root`, коли можна.

### 6.5 `try_files` та `index`

```nginx
location / {
    index index.html;
    try_files $uri $uri/ /index.html;
}
```

Перевіряє по черзі: чи існує файл `$uri`, чи існує каталог `$uri/` (тоді віддається `index`), і якщо ні — внутрішній редірект на останній аргумент. Саме так роблять SPA (React/Vue): будь-який невідомий шлях віддає `index.html`, а роутинг робить JS.

Варіанти останнього аргументу:

```nginx
try_files $uri $uri/ =404;                 # повернути код
try_files $uri $uri/ /index.php?$args;     # передати PHP (WordPress, Laravel)
try_files $uri @backend;                   # передати в named location
```

### 6.6 `if is evil`

`if` всередині `location` працює не як в мовах програмування — він створює вкладений неявний location, і більшість директив поводяться в ньому непередбачувано. Безпечно всередині `if` використовувати тільки `return` і `rewrite ... last`. Для умовної логіки краще `map`:

```nginx
# Погано
location / {
    if ($http_user_agent ~* "bot") {
        proxy_pass http://bots_backend;   # працює, але в комбінації з іншими директивами — сюрпризи
    }
}

# Добре
map $http_user_agent $backend_pool {
    default   app_backend;
    ~*bot     bots_backend;
}
server {
    location / {
        proxy_pass http://$backend_pool;
    }
}
```

---

## 7. Редіректи і rewrite

```nginx
# HTTP → HTTPS (правильний спосіб — return, а не rewrite)
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://example.com$request_uri;
}

# rewrite: regex + заміна + прапорець
rewrite ^/old/(.*)$  /new/$1  permanent;   # 301 клієнту
rewrite ^/user/(\d+)$ /profile?id=$1 last; # внутрішній, шукати location заново
```

| Прапорець | Що робить |
|---|---|
| `last` | Припинити rewrite-и в поточному блоці й шукати location для нового URI |
| `break` | Припинити rewrite-и й обробити новий URI в поточному location |
| `redirect` | 302 клієнту |
| `permanent` | 301 клієнту |

> Якщо можна обійтися `return` — використовуйте `return`. Він простіший, швидший і не потребує regex.

---

## 8. Reverse proxy

Це те, для чого nginx використовують найчастіше.

### 8.1 Базовий конфіг

```nginx
server {
    listen 443 ssl;
    server_name app.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 5s;     # час на встановлення TCP-з'єднання з бекендом
        proxy_send_timeout    60s;    # таймаут між двома записами в бекенд
        proxy_read_timeout    60s;    # таймаут між двома читаннями з бекенду → 504
    }
}
```

Навіщо заголовки:

- **`Host`** — за замовчуванням nginx передає бекенду `Host: <ім'я з proxy_pass>` (тобто `127.0.0.1:3000`). Застосунок, який генерує посилання або має virtual hosts, зламається.
- **`X-Real-IP` / `X-Forwarded-For`** — інакше бекенд бачить усі запити з `127.0.0.1`. `$proxy_add_x_forwarded_for` додає IP клієнта до вже наявного ланцюжка.
- **`X-Forwarded-Proto`** — бекенд знає, що клієнт прийшов по HTTPS (інакше нескінченні редіректи на https у Django/Rails/Spring).

> `$host` vs `$http_host`: `$host` — ім'я хоста в нижньому регістрі без порту (з request line, заголовка Host або `server_name`). `$http_host` — сирий заголовок Host як є, з портом. Зазвичай потрібен `$host`.

### 8.2 Слеш у `proxy_pass` — головна пастка

```nginx
location /api/ {
    proxy_pass http://backend;     # без URI
}
# /api/users → http://backend/api/users     (URI передається як є)

location /api/ {
    proxy_pass http://backend/;    # з URI "/"
}
# /api/users → http://backend/users         (/api/ замінюється на /)

location /api/ {
    proxy_pass http://backend/v2/;
}
# /api/users → http://backend/v2/users
```

Правило: **якщо в `proxy_pass` після хоста є хоч щось (навіть один `/`) — частина URI, що збіглась з location, замінюється на це «щось». Якщо нічого немає — URI передається без змін.**

### 8.3 Буферизація

За замовчуванням `proxy_buffering on`: nginx якомога швидше вичитує відповідь бекенду у свої буфери (а якщо не вміщається — у тимчасовий файл), звільняє бекенд, і далі сам повільно віддає клієнту. Це і є той самий «щит від повільних клієнтів».

```nginx
proxy_buffering on;
proxy_buffer_size 16k;       # буфер під заголовки відповіді
proxy_buffers 8 16k;         # буфери під тіло
proxy_busy_buffers_size 32k;
```

Вимикати буферизацію (`proxy_buffering off;`) треба для стрімінгу: Server-Sent Events, long polling, стрімінгові відповіді LLM API. Застосунок також може вимкнути її для конкретної відповіді заголовком `X-Accel-Buffering: no`.

### 8.4 WebSocket

WebSocket починається з HTTP-запиту з `Upgrade: websocket`. Ці заголовки hop-by-hop, і nginx їх не передає, поки не попросити явно:

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    location /ws/ {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_read_timeout 3600s;   # інакше idle-сокет закриється через 60s
    }
}
```

### 8.5 Справжній IP клієнта за балансувальником

Якщо перед nginx стоїть ще один проксі (AWS ALB, Cloudflare, HAProxy), `$remote_addr` буде IP цього проксі. Модуль `realip` це виправляє:

```nginx
set_real_ip_from 10.0.0.0/8;          # довіряти тільки своїм балансувальникам!
set_real_ip_from 173.245.48.0/20;     # діапазони Cloudflare тощо
real_ip_header   X-Forwarded-For;
real_ip_recursive on;
```

> Ніколи не ставте `set_real_ip_from 0.0.0.0/0` — тоді будь-хто підробить свій IP одним заголовком і обійде rate limiting та allow/deny.

---

## 9. Балансування навантаження

```nginx
upstream app_backend {
    least_conn;                          # алгоритм

    server 10.0.0.11:8000 weight=3;
    server 10.0.0.12:8000;
    server 10.0.0.13:8000 max_fails=3 fail_timeout=30s;
    server 10.0.0.14:8000 backup;        # тільки якщо всі основні лягли
    server 10.0.0.15:8000 down;          # тимчасово виведений

    keepalive 32;                        # пул idle-з'єднань до бекендів
}

server {
    location / {
        proxy_pass http://app_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";  # потрібно для keepalive до upstream
        proxy_next_upstream error timeout http_502 http_503;
        proxy_next_upstream_tries 2;
    }
}
```

### 9.1 Алгоритми

| Алгоритм | Директива | Коли |
|---|---|---|
| Round robin | (за замовчуванням) | Однакові сервери, короткі запити |
| Weighted round robin | `weight=N` | Сервери різної потужності |
| Least connections | `least_conn;` | Запити різної тривалості |
| IP hash | `ip_hash;` | Простий sticky session за IP клієнта |
| Generic hash | `hash $request_uri consistent;` | Кеш-сервери: той самий URI → той самий бекенд. `consistent` = ketama, мінімум перерозподілу при зміні складу |
| Random | `random two least_conn;` | Багато балансувальників перед одним пулом (power of two choices) |

### 9.2 Health checks

Open source nginx має тільки **пасивні** health checks: якщо за `fail_timeout` сталося `max_fails` невдалих спроб — сервер вважається недоступним на `fail_timeout`, після чого nginx знову пробує надіслати на нього запит.

**Активні** health checks (`health_check` — періодично стукати на `/healthz`) — тільки в NGINX Plus. В open source це вирішують сторонніми модулями, зовнішнім оркестратором (Consul, Kubernetes) або переходом на HAProxy/Envoy.

### 9.3 Keepalive до upstream

За замовчуванням nginx відкриває **нове TCP-з'єднання до бекенду на кожен запит** і ходить туди по HTTP/1.0. При великому RPS це тисячі з'єднань у `TIME_WAIT`, вичерпання ephemeral портів і зайва затримка на handshake (а з TLS до бекенду — дуже відчутна). Тріада `keepalive N` + `proxy_http_version 1.1` + `proxy_set_header Connection ""` це виправляє. Перевірте документацію своєї версії — у свіжих mainline-релізах дефолти поступово змінюються, але явна конфігурація нічого не ламає.

---

## 10. HTTPS / TLS

### 10.1 Сучасний конфіг

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;                                  # nginx 1.25.1+; раніше: listen 443 ssl http2;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;             # з TLS 1.3 і сучасними шифрами хай обирає клієнт

    ssl_session_cache   shared:SSL:10m;        # ~40k сесій, спільний для всіх worker'ів
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    add_header Strict-Transport-Security "max-age=63072000" always;
}
```

Нюанси:

- **`fullchain.pem`, а не `cert.pem`** — без проміжного сертифіката браузери ще можуть працювати (докачують), а `curl`, Java, Android, мобільні SDK — ні. Класичне «у мене в браузері все ок, а API не працює». Перевірка: `openssl s_client -connect example.com:443 -servername example.com`.
- **HSTS** — після нього браузер протягом `max-age` взагалі не піде на HTTP. Починайте з малого `max-age` і не додавайте `includeSubDomains`/`preload`, поки не впевнені, що всі піддомени працюють по HTTPS.
- **HTTP/3 (QUIC)** — `listen 443 quic reuseport;` + заголовок `Alt-Svc`, з nginx 1.25.0. Потрібен відкритий UDP 443.
- **OCSP stapling** — для сертифікатів Let's Encrypt вже неактуальний: LE у 2025 році відмовився від OCSP на користь CRL.
- Генератор перевіреного конфігу — Mozilla SSL Configuration Generator (ssl-config.mozilla.org).

### 10.2 Let's Encrypt

```bash
apt install certbot python3-certbot-nginx
certbot --nginx -d example.com -d www.example.com    # отримає сертифікат і пропише в конфіг
certbot renew --dry-run                              # перевірити автопродовження
systemctl list-timers | grep certbot                 # таймер продовження
```

---

## 11. PHP через FastCGI (PHP-FPM)

```nginx
server {
    listen 80;
    server_name blog.example.com;
    root /var/www/blog;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        try_files $uri =404;               # не передавати в PHP неіснуючі файли
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    }

    location ~ /\. {                       # заборонити .git, .env, .htaccess
        deny all;
    }
}
```

- `SCRIPT_FILENAME` — PHP-FPM з цього параметра дізнається, який файл виконувати. Без нього — біла сторінка або `Primary script unknown`.
- `try_files $uri =404` у php-location — захист від старої атаки, коли `/uploads/avatar.jpg/x.php` виконував завантажену «картинку» як PHP.
- Unix socket швидший за TCP `127.0.0.1:9000`, але nginx-користувач повинен мати права на сокет (`listen.owner` / `listen.group` у пулі FPM).

---

## 12. Продуктивність

```nginx
http {
    sendfile    on;          # файл копіюється ядром прямо в сокет, без user space
    tcp_nopush  on;          # з sendfile: відправляти заголовки й початок файлу одним пакетом
    tcp_nodelay on;          # не чекати (вимкнути алгоритм Nagle) для keepalive-з'єднань

    keepalive_timeout  65;
    keepalive_requests 1000;

    client_max_body_size 50m;          # за замовчуванням 1m → 413 на завантаженнях
    client_body_buffer_size 128k;

    gzip on;
    gzip_comp_level 5;                 # 5-6 — баланс, вище майже не дає виграшу
    gzip_min_length 1024;              # маленькі відповіді стискати невигідно
    gzip_proxied any;                  # стискати й проксовані відповіді
    gzip_vary on;                      # Vary: Accept-Encoding для кешів
    gzip_types text/plain text/css application/json application/javascript
               text/xml application/xml image/svg+xml;
    # text/html стискається завжди, додавати не треба

    open_file_cache max=10000 inactive=30s;   # кешувати дескриптори й метадані файлів
    open_file_cache_valid 60s;
    open_file_cache_errors on;
}

# Довгий кеш для статики з хешем в імені (app.3f9a2c.js)
location ~* \.(?:css|js|woff2|png|jpg|jpeg|gif|svg|ico|webp|avif)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
    access_log off;
}
```

> Не стискайте вже стиснені формати (jpg, png, mp4, zip, woff2) — CPU витрачено, розмір не змінився.

---

## 13. Кешування відповідей (proxy_cache)

Nginx може бути повноцінним кешем перед бекендом — по суті міні-CDN.

```nginx
http {
    proxy_cache_path /var/cache/nginx/app
                     levels=1:2              # структура каталогів (не 100k файлів в одному)
                     keys_zone=app_cache:50m # ключі в shared memory, 1 МБ ≈ 8000 ключів
                     max_size=10g            # ліміт на диску, далі — LRU-видалення
                     inactive=60m            # видалити, якщо ніхто не запитував 60 хв
                     use_temp_path=off;

    server {
        location /api/public/ {
            proxy_pass http://app_backend;

            proxy_cache app_cache;
            proxy_cache_key "$scheme$request_method$host$request_uri";
            proxy_cache_valid 200 302 10m;
            proxy_cache_valid 404      1m;

            proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
            proxy_cache_background_update on;
            proxy_cache_lock on;              # захист від cache stampede

            add_header X-Cache-Status $upstream_cache_status;  # HIT / MISS / EXPIRED / STALE / BYPASS
        }
    }
}
```

- **`proxy_cache_use_stale`** — якщо бекенд упав, віддавати протерміновану копію. Сайт «живий», навіть коли API лежить.
- **`proxy_cache_lock`** — коли кеш протух, до бекенду піде лише один запит, решта чекатимуть на нього, а не повалять бекенд одночасно.
- Nginx за замовчуванням **не кешує** відповіді з `Set-Cookie` або `Cache-Control: private/no-store`, а також запити, крім GET/HEAD. Якщо кеш «не працює» — перевірте спершу заголовки бекенду.
- Ніколи не кешуйте персоналізовані відповіді з ключем без ідентифікатора користувача — віддасте чужий кабінет іншим людям.

---

## 14. Безпека

### 14.1 Базове

```nginx
server_tokens off;                      # не показувати версію nginx у заголовку Server і сторінках помилок

location ~ /\.(?!well-known) { deny all; }   # .git, .env, .htpasswd, але дозволити .well-known (ACME)

location /admin/ {
    allow 10.0.0.0/8;
    allow 192.168.1.0/24;
    deny  all;

    auth_basic           "Restricted";
    auth_basic_user_file /etc/nginx/.htpasswd;   # htpasswd -c /etc/nginx/.htpasswd admin
}
```

### 14.2 Rate limiting

Nginx використовує алгоритм **leaky bucket**:

```nginx
http {
    # Ключ — IP клієнта (binary форма займає менше пам'яті), 10 МБ ≈ 160k IP, 10 запитів/с
    limit_req_zone  $binary_remote_addr zone=api_limit:10m rate=10r/s;
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

    server {
        location /api/ {
            limit_req zone=api_limit burst=20 nodelay;
            limit_req_status 429;               # за замовчуванням 503 — вводить в оману моніторинг
            limit_conn conn_limit 20;           # не більше 20 одночасних з'єднань з одного IP
            proxy_pass http://app_backend;
        }

        location = /login {
            limit_req zone=api_limit burst=5;   # без nodelay: зайві запити ставляться в чергу
            proxy_pass http://app_backend;
        }
    }
}
```

- **`rate=10r/s`** — nginx рахує з точністю до мілісекунд, тобто це «1 запит на 100 мс», а не «10 будь-коли за секунду».
- **`burst=20`** — дозволити сплеск до 20 запитів понад ліміт. Без `nodelay` вони розтягуються в часі з заданим rate; з `nodelay` — обробляються одразу, але «слоти» звільняються з rate.
- Без `burst` звичайна сторінка, яка паралельно вантажить 5 ресурсів, отримає 4 відмови.

### 14.3 Security-заголовки

```nginx
# snippets/security-headers.conf
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Content-Security-Policy "default-src 'self'" always;   # налаштовується під конкретний сайт
add_header Strict-Transport-Security "max-age=63072000" always;
```

Пам'ятайте про розділ 4.3: якщо в location є свій `add_header` — цей snippet треба включити туди теж.

---

## 15. Логи

### 15.1 Формат з таймінгами

Стандартний формат `combined` не показує головного для SRE — **скільки часу зайняв запит і хто винен: nginx чи бекенд**.

```nginx
log_format main_timed '$remote_addr - $remote_user [$time_local] '
                      '"$request" $status $body_bytes_sent '
                      '"$http_referer" "$http_user_agent" '
                      'rt=$request_time uct=$upstream_connect_time '
                      'uht=$upstream_header_time urt=$upstream_response_time '
                      'ua="$upstream_addr" us=$upstream_status cs=$upstream_cache_status';

access_log /var/log/nginx/access.log main_timed buffer=64k flush=5s;
```

| Змінна | Що означає |
|---|---|
| `$request_time` | Повний час від першого байта запиту клієнта до відправки останнього байта відповіді |
| `$upstream_connect_time` | Час на встановлення з'єднання з бекендом |
| `$upstream_header_time` | Час до отримання заголовків відповіді від бекенду (≈ час «думання» застосунку) |
| `$upstream_response_time` | Повний час отримання відповіді від бекенду |
| `$upstream_addr` | Який саме бекенд обробив (при ретраях — кілька через кому) |

Діагностика: `request_time` великий, а `upstream_response_time` малий → проблема на стороні клієнта (повільна мережа, великі файли). Обидва великі → повільний бекенд.

### 15.2 JSON-логи для Loki / ELK

```nginx
log_format json_log escape=json '{'
    '"time":"$time_iso8601",'
    '"remote_addr":"$remote_addr",'
    '"request_id":"$request_id",'
    '"method":"$request_method",'
    '"uri":"$request_uri",'
    '"status":$status,'
    '"bytes":$body_bytes_sent,'
    '"request_time":$request_time,'
    '"upstream_time":"$upstream_response_time",'
    '"upstream":"$upstream_addr",'
    '"user_agent":"$http_user_agent"'
'}';
```

`$request_id` — унікальний ID запиту. Передайте його бекенду (`proxy_set_header X-Request-ID $request_id;`) — і зможете знайти один запит і в логах nginx, і в логах застосунку.

### 15.3 Error log

```nginx
error_log /var/log/nginx/error.log warn;   # debug | info | notice | warn | error | crit | alert | emerg
```

В error log — найкорисніша інформація при 502/504: `connect() failed (111: Connection refused)`, `upstream timed out (110)`, `no live upstreams`. Рівень `debug` працює тільки якщо nginx зібраний з `--with-debug`.

---

## 16. Команди, які треба знати напам'ять

```bash
nginx -t                  # перевірити синтаксис конфігу — ЗАВЖДИ перед reload
nginx -T                  # перевірити й вивести ПОВНИЙ конфіг з усіма include (дуже корисно для дебагу)
nginx -V                  # версія, опції збірки і список вкомпільованих модулів
nginx -s reload           # graceful reload

systemctl reload nginx    # те саме через systemd
systemctl status nginx
journalctl -u nginx -n 100 --no-pager

# Безпечний reload одним рядком
nginx -t && systemctl reload nginx

# Що слухає nginx
ss -tlnp | grep nginx

# Знайти, в якому файлі описано server_name
nginx -T 2>/dev/null | grep -n "server_name"

# Перевірити конкретний virtual host без DNS
curl -sv -H "Host: example.com" http://127.0.0.1/
curl -sv --resolve example.com:443:127.0.0.1 https://example.com/

# Заголовки відповіді
curl -sI https://example.com/

# Топ IP / статусів / повільних запитів
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
grep -oP 'rt=\K[0-9.]+' /var/log/nginx/access.log | sort -rn | head
```

Для метрик: модуль `stub_status` (активні з'єднання, accepts, handled, requests) + `nginx-prometheus-exporter` → Prometheus/Grafana.

```nginx
location = /nginx_status {
    stub_status;
    allow 127.0.0.1;
    deny all;
}
```

---

## 17. Траблшутінг: коди помилок і що з ними робити

### 17.1 502 Bad Gateway

**Що означає:** nginx не зміг отримати валідну відповідь від бекенду.

```bash
tail -f /var/log/nginx/error.log
```

| У error log | Причина | Що робити |
|---|---|---|
| `connect() failed (111: Connection refused)` | Бекенд не слухає порт/сокет | `systemctl status app`, `ss -tlnp`, правильний порт у `proxy_pass` |
| `connect() to unix:/run/php/...sock failed (2: No such file)` | PHP-FPM не запущений або інша версія сокета | `systemctl status php8.3-fpm`, `ls /run/php/` |
| `(13: Permission denied) while connecting to upstream` | Права на сокет або SELinux | Права на сокет; на RHEL: `setsebool -P httpd_can_network_connect 1` |
| `upstream prematurely closed connection` | Бекенд упав під час обробки (OOM, crash) | Логи застосунку, `dmesg -T \| grep -i oom` |
| `upstream sent too big header` | Заголовки відповіді (часто великі cookies) не влазять у буфер | `proxy_buffer_size 16k; proxy_buffers 8 16k;` (для FastCGI — `fastcgi_buffer_size`) |
| `no live upstreams` | Усі сервери upstream позначені як недоступні | Перевірити бекенди, `max_fails`/`fail_timeout` |

### 17.2 504 Gateway Timeout

**Що означає:** бекенд прийняв з'єднання, але не відповів за `proxy_read_timeout` (за замовчуванням 60s).

- Спершу з'ясуйте, **чому** бекенд повільний (повільні SQL-запити, блокування, вичерпаний пул воркерів), а не просто збільшуйте таймаут.
- Для легітимно довгих операцій (звіти, експорт) — збільшити таймаут **точково** в окремому location, а краще перевести на асинхронні задачі.
- Для FastCGI таймаут окремий: `fastcgi_read_timeout`. Плюс `request_terminate_timeout` / `max_execution_time` на боці PHP.

### 17.3 499 Client Closed Request

Nginx-специфічний код: **клієнт закрив з'єднання, не дочекавшись відповіді**. Це не помилка nginx. Масові 499 означають, що бекенд відповідає довше, ніж готовий чекати клієнт — найчастіше таймаут балансувальника перед nginx (наприклад, ALB idle timeout 60s) менший за `proxy_read_timeout`, або мобільний клієнт з агресивним таймаутом. Дивіться на `$request_time` у цих рядках.

### 17.4 413 Request Entity Too Large

`client_max_body_size` (за замовчуванням **1m**) замалий для завантаження файлу. Збільшити в потрібному `server`/`location`. Якщо перед nginx є ще один проксі або ingress — там теж свій ліміт.

### 17.5 403 Forbidden

- Немає прав на читання файлу або на **виконання (x) кожного каталогу в шляху** для користувача nginx: `namei -l /var/www/site/index.html`.
- Запит на каталог без index-файлу і з `autoindex off`.
- Спрацював `deny`.
- SELinux-контекст: `ls -Z`, `restorecon -Rv /var/www`.

### 17.6 404, хоча файл існує

- Плутанина `root` / `alias` (розділ 6.4) — у error log видно, який шлях nginx реально шукав: `open() "/var/www/images/images/cat.png" failed`.
- Запит потрапив не в той `server` (розділ 5) або не в той `location` (розділ 6). Перевірка: `nginx -T` і тимчасовий `add_header X-Debug-Location "static" always;` у підозрюваних location.

### 17.7 Нескінченний редірект (ERR_TOO_MANY_REDIRECTS)

Класика: TLS термінується на балансувальнику (Cloudflare «Flexible», ALB), до nginx запит приходить по HTTP, nginx редіректить на HTTPS → балансувальник знову шле по HTTP → цикл. Рішення: Cloudflare «Full (strict)» або перевіряти `$http_x_forwarded_proto` замість `$scheme`. Те саме з застосунком, який не отримує `X-Forwarded-Proto`.

### 17.8 `Too many open files` / `worker_connections are not enough`

```bash
cat /proc/$(pgrep -f "nginx: worker" | head -1)/limits | grep "open files"
```

Підняти `worker_rlimit_nofile` у nginx.conf і `LimitNOFILE=65535` в override systemd (`systemctl edit nginx`), збільшити `worker_connections`.

### 17.9 Nginx не стартує: `bind() to 0.0.0.0:80 failed (98: Address already in use)`

Порт зайнятий іншим процесом (Apache, інший nginx, Docker-контейнер): `ss -tlnp | grep ':80 '`.

### 17.10 Алгоритм при будь-якому інциденті

```bash
systemctl status nginx                       # 1. чи живий
nginx -t                                     # 2. чи валідний конфіг (раптом хтось змінив)
tail -100 /var/log/nginx/error.log           # 3. що каже error log
tail -f /var/log/nginx/access.log            # 4. які статуси і таймінги зараз
ss -s; ss -tan state time-wait | wc -l       # 5. з'єднання, TIME_WAIT
curl -sv http://127.0.0.1:3000/health        # 6. бекенд напряму, в обхід nginx
top; free -h; df -h; dmesg -T | tail          # 7. ресурси сервера, OOM, диск
```

Якщо бекенд напряму працює, а через nginx ні — проблема в nginx-конфігу. Якщо й напряму не працює — nginx тут лише гонець із поганими новинами.

---

## 18. Каверзні питання для співбесіди — з відповідями

**1. Чим nginx принципово відрізняється від Apache?**
Архітектурою обробки з'єднань: event-driven, неблокуючі worker'и з `epoll` проти процесу/потоку на з'єднання (prefork/worker) або на активний запит (event MPM). Плюс nginx не виконує код застосунку (немає аналога `mod_php`) і не має `.htaccess`.

**2. Чому worker_processes зазвичай дорівнює кількості ядер?**
Кожен worker однопотоковий і не блокується на I/O, тому одного worker'а на ядро достатньо, щоб завантажити CPU. Більше worker'ів — лише зайве перемикання контексту.

**3. Що станеться, якщо в worker'і виконати блокуючу операцію?**
Зупиниться весь event loop цього worker'а — тисячі з'єднань чекатимуть. Тому повільні операції або винесені в бекенд, або використовують thread pools (`aio threads`) для читання з диска.

**4. Як nginx обирає location?**
Точний `=` → найдовший префікс; якщо він з `^~` — стоп → regex по черзі, перший що збігся → інакше найдовший префікс.

**5. Різниця між `root` і `alias`?**
`root` додає повний URI до шляху, `alias` замінює частину URI, що збіглась з location.

**6. Що змінює слеш у кінці `proxy_pass http://backend/`?**
Наявність URI в `proxy_pass` змушує nginx замінити частину URI, що збіглась з location, на цей URI. Без URI шлях передається без змін.

**7. Чим `reload` відрізняється від `restart`?**
Reload — graceful: нові worker'и з новим конфігом, старі дообробляють поточні з'єднання. Restart — зупинка процесу і всіх з'єднань. Якщо конфіг невалідний, reload залишить старий конфіг, а restart залишить вас без nginx.

**8. Чому `add_header` з `server` не працює в моєму location?**
Масивні директиви успадковуються, тільки якщо на поточному рівні немає жодної такої директиви. Плюс без `always` заголовок не додається до 4xx/5xx.

**9. Що означають 502, 504 і 499?**
502 — бекенд недоступний або повернув невалідну відповідь. 504 — бекенд не відповів вчасно. 499 — клієнт сам закрив з'єднання, не дочекавшись.

**10. Як отримати справжній IP клієнта за балансувальником?**
Модуль `realip`: `set_real_ip_from` (тільки довірені мережі) + `real_ip_header X-Forwarded-For` + `real_ip_recursive on`.

**11. Як зробити zero-downtime деплой бекенду за nginx?**
Кілька серверів в upstream, по черзі виводити їх (`down` + reload, або через service discovery), `proxy_next_upstream` для ретраю на інший сервер, graceful shutdown у самому застосунку.

**12. Чому nginx за замовчуванням не тримає keepalive до бекенду і як це виправити?**
Історично до upstream використовується HTTP/1.0 з `Connection: close`. Виправлення: `keepalive N` в upstream + `proxy_http_version 1.1` + `proxy_set_header Connection ""`.

**13. Що таке `sendfile` і чому він прискорює статику?**
Системний виклик, який копіює дані з файлу в сокет усередині ядра (zero-copy), без проміжного буфера в user space.

**14. Як захистити бекенд від сплеску запитів?**
`limit_req` (leaky bucket) з `burst`, `limit_conn`, кешування з `proxy_cache_lock` і `proxy_cache_use_stale`, буферизація відповідей.

**15. Чому `if` в location вважається злом?**
Він створює неявний вкладений location, і багато директив у ньому поводяться неочікувано. Безпечні лише `return` і `rewrite ... last`; решту робити через `map` і `try_files`.

**16. Nginx vs HAProxy vs Envoy?**
Nginx — універсальний: web-сервер + reverse proxy + кеш. HAProxy — спеціалізований L4/L7-балансувальник з активними health checks, детальною статистикою і runtime API в open source. Envoy — проксі для service mesh з динамічною конфігурацією через xDS API (Istio, Gateway API-реалізації).

---

## 19. Чек-лист: що треба знати про nginx

- [ ] Пояснити різницю nginx vs Apache: event loop vs process/thread per connection, `mod_php` vs PHP-FPM, `.htaccess` vs централізований конфіг
- [ ] Master/worker модель, `worker_processes`, `worker_connections`, формула максимуму клієнтів
- [ ] Що відбувається при `reload` і чому він не рве з'єднання
- [ ] Контексти `main` / `events` / `http` / `server` / `location` / `upstream` / `stream`
- [ ] Пастка успадкування `add_header` / `proxy_set_header` і параметр `always`
- [ ] Як обирається `server`: `listen` → `server_name` → `default_server`
- [ ] Алгоритм вибору `location` і пріоритет `=`, `^~`, `~`, `~*`, префіксу
- [ ] `root` vs `alias`, `try_files`, `return` vs `rewrite`, чому `if is evil`
- [ ] Reverse proxy: обов'язкові заголовки, слеш у `proxy_pass`, таймаути, буферизація, WebSocket
- [ ] Балансування: алгоритми, `max_fails`/`fail_timeout`, `backup`, keepalive до upstream, пасивні vs активні health checks
- [ ] TLS: `fullchain`, TLS 1.2/1.3, HSTS, `http2 on`, certbot
- [ ] PHP-FPM: `SCRIPT_FILENAME`, сокет, `try_files $uri =404`
- [ ] Продуктивність: `sendfile`, gzip, `expires`, `client_max_body_size`
- [ ] Кеш: `proxy_cache_path`, `proxy_cache_use_stale`, `proxy_cache_lock`, `X-Cache-Status`
- [ ] Безпека: `server_tokens off`, `limit_req`/`limit_conn`, allow/deny, basic auth, security headers, `realip`
- [ ] Логи з `$request_time` і `$upstream_response_time` і як за ними визначити винного
- [ ] Команди: `nginx -t`, `nginx -T`, `nginx -V`, `nginx -s reload`, `curl --resolve`
- [ ] Діагностувати 502, 504, 499, 413, 403, 404, нескінченні редіректи, `too many open files`, `address already in use`

---
