+++
title = 'Linux Distros Comparison'
date = 2026-05-29T22:26:41+03:00
draft = false
+++

Вибір дистрибутива Linux для корпоративної інфраструктури чи власного проєкту — це не питання звички чи смаку. За кожним рішенням великих компаній стоїть чітка інженерна та бізнес-логіка.

У цій статті розберемо анатомію сучасного ринку серверних ОС: архітектурне дерево сімейств, конкретні технічні відмінності між ними, підводні камені, які безпосередньо впливають на автоматизацію, безпеку та довгострокову підтримку інфраструктури.

---

## Архітектурне дерево: два з половиною сімейства

Весь сучасний світ серверного Linux тримається на **двох великих сімействах**, які беруть початок на початку 90-х.

### Коріння Debian

- **Debian** — з'явився у 1993 році як суто незалежний від комерції проєкт спільноти. Філософія: повна свобода, передбачуваність та залізобетонна стабільність. Мінімум пакетів з коробки, максимум контролю.
- **Ubuntu** — найвідоміший нащадок Debian, розвивається компанією Canonical. Поєднує стабільну базу з новими пакетами, хмарними оптимізаціями та комерційною підтримкою (корпоративний план через Ubuntu Pro).

### Коріння Red Hat

- **RHEL** — комерційний гігант, який першим почав продавати не сам код, а технічну підтримку з SLA та сертифікацію для Enterprise-сегмента. Стандарт де-факто для банків, телеком і великих корпорацій.
- **Трагедія CentOS** — тривалий час CentOS був безкоштовним бінарним клоном RHEL, який використовували скрізь. Після поглинання Red Hat (потім IBM) вектор змінився на *CentOS Stream* — тестовий полігон для майбутніх релізів RHEL. Використовувати CentOS Stream на продакшені небезпечно: він йде *попереду* RHEL, а не слідує за ним.
- **Нові спадкоємці** — спільнота відреагувала двома повноцінними бінарно сумісними клонами RHEL:
  - **AlmaLinux** — підтримується некомерційною організацією, фінансується CloudLinux.
  - **Rocky Linux** — повністю ком'юніті-проєкт, заснований творцем оригінального CentOS Грегорі Курцером.

Обидва є дроп-ін заміною RHEL і слідують за його релізами 1-в-1.

---

## Lifecycle та EOL: планування на роки вперед

Це перше, що варто перевіряти перед вибором дистрибутива. Поставити сервер на ОС з коротким терміном підтримки — значить планувати міграцію заздалегідь.

| Дистрибутив | Версія | Full Support | Extended/Security | Примітка |
|---|---|---|---|---|
| Ubuntu LTS | 24.04 | 2029 | 2034 (ESM) | ESM платний поза Pro-планом |
| Debian | 12 (Bookworm) | ~2028 | ~2030 (LTS) | Безкоштовно |
| RHEL | 9 | 2032 | 2034 (Maintenance) | Платна підписка |
| AlmaLinux | 9 | 2032 | 2034 | Безкоштовно |
| Rocky Linux | 9 | 2032 | 2034 | Безкоштовно |

**Важливий нюанс щодо Ubuntu ESM:** перші 5 років безкоштовні навіть для серверів. Але Extended Security Maintenance (роки 6–10) входить до платного **Ubuntu Pro** або потребує реєстрації на ubuntu.com (безкоштовно для до 5 машин). Для великого парку без Pro — через 5 років ви отримуєте сервери без security-патчів.

---

## Головні технічні відмінності

### 1. Пакетні менеджери та назви пакетів

Скрипти автоматизації (Dockerfile, Ansible-ролі, cloud-init), написані під одне сімейство, без адаптації не працюватимуть на іншому.

**Менеджери:**

| Сімейство | Менеджер | Формат |
|---|---|---|
| Debian / Ubuntu | `apt` | `.deb` |
| RHEL / Alma / Rocky | `dnf` (раніше `yum`) | `.rpm` |
| Alpine | `apk` | `.apk` |

**Різниця в іменуванні пакетів — щоденний біль:**

| Пакет | Debian/Ubuntu | RHEL/Alma/Rocky |
|---|---|---|
| Веб-сервер Apache | `apache2` | `httpd` |
| Заголовки Python | `python3-dev` | `python3-devel` |
| Заголовки OpenSSL | `libssl-dev` | `openssl-devel` |
| Конфіги Apache | `/etc/apache2/` | `/etc/httpd/` |

**EPEL — обов'язковий репозиторій для RHEL-сімейства:**

Базовий RHEL вкрай бідний на пакети. `htop`, `screen`, `ncdu`, `iotop` — нічого цього немає в базі. Для отримання сучасних утиліт обов'язково підключати **EPEL** (Extra Packages for Enterprise Linux):

```bash
# AlmaLinux / Rocky / RHEL
dnf install -y epel-release

# Або напряму для RHEL
dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
```

У Debian/Ubuntu майже все необхідне є в дефолтних репозиторіях.

**Debconf — інтерактивна конфігурація при встановленні:**

Особливість Debian/Ubuntu: під час встановлення пакета `debconf` може запитати параметри конфігурації (пароль для БД, порт, TLS-сертифікат). У RHEL нічого подібного немає — всі налаштування постфактум. При автоматизації через Ansible або CI це треба враховувати: змінну `DEBIAN_FRONTEND=noninteractive` потрібно виставляти явно, інакше пайплайн зависне на очікуванні введення.

```bash
DEBIAN_FRONTEND=noninteractive apt-get install -y postfix
```

---

### 2. Snap у Ubuntu: реальний біль на серверах

У пості часто про це говорять побіжно. Насправді починаючи з Ubuntu 22.04 ряд пакетів **примусово перейшов у snap-only**: `certbot`, `lxd`, у 24.04 — `firefox`. Для серверів це створює конкретні проблеми:

**Проблема 1 — засмічення `loop`-пристроїв:**

```bash
$ df -h
/dev/loop0   55M   55M     0 100% /snap/core18/...
/dev/loop1   92M   92M     0 100% /snap/lxd/...
/dev/loop2   44M   44M     0 100% /snap/snapd/...
```

Кожен snap-пакет монтується як окремий loop-пристрій. Prometheus `node_exporter` за замовчуванням збирає метрики `node_filesystem_*` для **всіх** точок монтування, включаючи ці. Результат — кардинальне збільшення cardinality у TSDB та шум на дашбордах.

**Рішення для node_exporter:**

```yaml
# docker-compose або systemd unit
--collector.filesystem.mount-points-exclude=^/(dev|proc|sys|snap/.+)($|/)
```

**Проблема 2 — автооновлення без попередження:**

Snap оновлює пакети автоматично вночі. Для стабільних прод-серверів це неприйнятно. Вимкнути:

```bash
snap set system refresh.hold="$(date --date='today + 60 days' +%Y-%m-%dT%H:%M:%S%:z)"
```

**Проблема 3 — зайвий демон `snapd`** з власним мережевим overhead і memory footprint.

**Висновок:** на серверах де потрібна максимальна передбачуваність — Debian, де Snap немає взагалі.

---

### 3. Безпека: SELinux проти AppArmor

**RHEL / Alma / Rocky → SELinux (Mandatory Access Control):**

SELinux працює на рівні міток (labels) в файловій системі. Кожен процес і файл мають контекстну мітку. Навіть якщо права доступу `755` і власник правильний — якщо контекст безпеки SELinux це не дозволяє, процес отримає `Permission Denied`.

Типова пастка при деплої нового сервісу:

```bash
# Nginx не може читати файли зі свого кастомного шляху
$ tail /var/log/audit/audit.log | grep denied
type=AVC msg=audit: avc: denied { read } for pid=1234 comm="nginx" \
  name="app.conf" scontext=system_u:system_r:httpd_t \
  tcontext=unconfined_u:object_r:default_t

# Фікс: виставити правильний контекст
$ semanage fcontext -a -t httpd_config_t "/opt/myapp/nginx(/.*)?"
$ restorecon -Rv /opt/myapp/nginx/
```

Більшість туторіалів радять `setenforce 0` — в продакшені це моветон. Правильний підхід: `audit2allow` для аналізу відмов і генерації мінімального policy.

**Debian / Ubuntu → AppArmor:**

Значно простіший: прив'язується до профілів конкретних бінарників і шляхів. Зазвичай працює тихо у фоні. Профілі лежать у `/etc/apparmor.d/`. Перевірити статус:

```bash
aa-status
```

---

### 4. Мережа та файрволи

**Управління мережею:**

| Дистрибутив | Інструмент | Конфіг |
|---|---|---|
| Ubuntu | Netplan | `/etc/netplan/*.yaml` |
| Debian | ifupdown | `/etc/network/interfaces` |
| RHEL/Alma/Rocky | NetworkManager | `nmcli` / `/etc/NetworkManager/` |

Netplan в Ubuntu — зручний YAML, але є нюанс: він є лише фронтендом над NetworkManager або systemd-networkd. Бекенд важливий при налаштуванні складних схем (bonding, VLAN, policy routing).

**Файрволи:**

- **UFW** (Ubuntu/Debian) — проста обгортка над iptables: `ufw allow 22/tcp`. Зручно для ручного управління, але при автоматизації через Ansible краще працювати напряму з `iptables`/`nftables`.
- **Firewalld** (RHEL-сімейство) — концепція зон (`public`, `internal`, `trusted`). Інтерфейси розподіляються по зонах. Складніше писати Ansible-плейбуки, але гнучкіше для мультімережевих серверів:

```bash
firewall-cmd --zone=internal --add-service=http --permanent
firewall-cmd --reload
```

---

### 5. Ядра, eBPF та IO_uring — чому версія ядра критична

У пості згадується HWE Ubuntu, але без пояснення чому це важливо на практиці.

**RHEL 9 вийшов з ядром 5.14 і залишиться на ньому до 2032 року** — бекпортуються лише security-патчі. Ubuntu LTS через HWE (Hardware Enablement) дозволяє опціонально оновлювати ядро на свіжіші версії.

**Чому це важливо для DevOps:**

| Технологія | Мінімальне ядро | Де використовується |
|---|---|---|
| eBPF (базовий) | 4.9 | Cilium, Falco |
| eBPF (повний функціонал) | 5.8+ | Tetragon, Pixie, Hubble |
| eBPF BTF (CO-RE) | 5.2+ | libbpf-based tools |
| io_uring | 5.1+ | PostgreSQL async I/O, Scylla |
| WireGuard (in-tree) | 5.6 | VPN без DKMS |
| cgroups v2 (повна підтримка) | 5.2+ | containerd, Kubernetes |

**Важливо:** Alpine у контейнерах використовує ядро **хост-машини**. Тобто якщо нода Kubernetes на RHEL 8 (ядро 4.18) — eBPF-фічі не будуть доступні у жодному контейнері, незалежно від базового образу.

**Перевірити версію ядра і підтримку BTF:**

```bash
uname -r
ls /sys/kernel/btf/vmlinux && echo "BTF supported"
```

---

### 6. Логування: де шукати системні логи

Дрібниця, яка стає проблемою при налаштуванні Promtail/Vector/Fluentd:

| Дистрибутив | Системний лог |
|---|---|
| RHEL / Alma / Rocky | `/var/log/messages` |
| Debian / Ubuntu | `/var/log/syslog` |
| Всі (через journald) | `journalctl -u <service>` |

При написанні конфігів для Loki Promtail або Vector враховуйте це в `path`:

```yaml
# Promtail scrape config — враховуємо обидва шляхи
- job_name: syslog
  static_configs:
    - targets: [localhost]
      labels:
        job: syslog
        __path__: /var/log/{messages,syslog}
```

---

### 7. systemd-resolved та DNS-пастка в контейнерах

Ubuntu 22.04+ використовує `systemd-resolved` з symlink:

```
/etc/resolv.conf → /run/systemd/resolve/stub-resolv.conf
```

DNS-сервер там `127.0.0.53` (loopback). При збірці Docker-образу або монтуванні хостового `/etc/resolv.conf` в контейнер — DNS **не працює**, бо `127.0.0.53` недоступний зсередини контейнера.

**Діагностика:**

```bash
# Всередині контейнера
cat /etc/resolv.conf
# Якщо бачите nameserver 127.0.0.53 — це і є проблема

# На хості перевірити реальний DNS
resolvectl status
```

**Рішення:**

```bash
# На хості Ubuntu — переключити на статичний resolv.conf
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
# Тепер там будуть реальні upstream DNS-сервери
```

У Debian та RHEL цієї проблеми немає за замовчуванням.

---

### 8. Alpine Linux: стандарт для контейнерів з підводними каменями

Alpine стоїть окремо від усіх. Базовий Docker-образ важить **~5 МБ** завдяки двом радикальним рішенням:

1. Заміна `glibc` → **musl libc** (мінімалістична, POSIX-сумісна).
2. Заміна GNU utils → **BusyBox** (всі базові команди в одному бінарнику).

**Система ініціалізації:** OpenRC (не SystemD).

**Підводні камені на практиці:**

**Проблема 1 — несумісність musl з glibc:**

Будь-який **statically linked binary скомпільований під glibc** не запуститься в Alpine без перекомпіляції. Часто зустрічається в корпоративних інструментах: агенти моніторингу, VPN-клієнти, комерційні СУБД-клієнти.

Python/Node пакети з C-розширеннями (native extensions) можуть збиратися годинами або взагалі ламатися. Якщо потрібні `numpy`, `cryptography`, `lxml` — часто простіше взяти `python:3.12-slim-bookworm` (~50MB) замість `python:3.12-alpine`.

Правильний патерн multistage build:

```dockerfile
# Build stage — debian для компіляції з glibc
FROM ubuntu:24.04 AS builder
RUN apt-get update && apt-get install -y build-essential
WORKDIR /app
COPY . .
RUN make static  # компілюємо статично

# Runtime stage — alpine тільки якщо binary статично зліновано
FROM alpine:3.20
COPY --from=builder /app/bin /app/bin
ENTRYPOINT ["/app/bin"]
```

**Проблема 2 — DNS у Kubernetes:**

Alpine інакше обробляє `search` та `ndots` у `resolv.conf`. У K8s-кластері це іноді призводить до загадкових мережевих таймаутів при резолвінгу внутрішніх сервісів. Симптом: сервіс резолвиться повільно або з першого разу не відповідає.

**Фікс у Pod spec:**

```yaml
spec:
  dnsConfig:
    options:
      - name: ndots
        value: "1"  # замість дефолтних 5
```

**Висновок:** Alpine — стандарт де-факто для Docker-контейнерів де важливий розмір образу і мінімальна surface attack. Для хост-нод Kubernetes — залишайте Ubuntu або RHEL-клони.

---

### 9. Ansible та крос-дистрибутивна автоматизація

Це щоденний біль при гетерогенному парку серверів.

**Неправильно** — ламається на RHEL:

```yaml
- name: Install nginx
  apt:
    name: nginx
    state: present
```

**Правильно** — через абстрактний модуль `package`:

```yaml
- name: Install nginx
  package:
    name: nginx
    state: present
```

Але навіть `package` не вирішує проблему **різних назв пакетів** між сімействами. Для цього використовуйте `ansible_os_family` або `ansible_distribution`:

```yaml
- name: Install Python dev headers
  package:
    name: "{{ python_dev_pkg[ansible_os_family] }}"
    state: present
  vars:
    python_dev_pkg:
      Debian: python3-dev
      RedHat: python3-devel

- name: Enable EPEL (RedHat only)
  package:
    name: epel-release
    state: present
  when: ansible_os_family == "RedHat"
```

**Файрвол у Ansible — теж різний:**

```yaml
# UFW для Debian/Ubuntu
- community.general.ufw:
    rule: allow
    port: "443"
  when: ansible_os_family == "Debian"

# Firewalld для RHEL
- ansible.posix.firewalld:
    service: https
    permanent: true
    state: enabled
  when: ansible_os_family == "RedHat"
```

---

### 10. Атомарні ОС — наступний крок для Kubernetes-нод

Стаття не буде повною без згадки нової хвилі: **immutable/atomic Linux** спеціально для хмарних та K8s-середовищ.

**Принцип:** ОС оновлюється цілим образом за схемою A/B partition. Немає `apt upgrade` або `dnf update` — є атомарне перемикання між версіями образу. Відкат у разі проблем — одна команда.

| Дистрибутив | Основа | Призначення |
|---|---|---|
| **Talos Linux** | — | Kubernetes-ноди (тільки K8s, без SSH) |
| **Fedora CoreOS** | RHEL | Контейнерні workloads |
| **Flatcar Linux** | CoreOS | Замінник CoreOS від Kinvolk/Microsoft |
| **Bottlerocket** | — | AWS EKS ноди від Amazon |

**Talos Linux** — найрадикальніший: там немає shell доступу взагалі. Управління виключно через API (`talosctl`). Мінімальна surface attack, максимальна відтворюваність. Якщо будуєте новий K8s-кластер з нуля — варто розглянути.

---

## Що обрати під конкретну задачу

| Критерій | Ubuntu Server | Debian | RHEL / Alma / Rocky | Alpine | Talos |
|---|---|---|---|---|---|
| Пакетний менеджер | APT | APT | DNF | APK | — (немає) |
| Init-система | SystemD | SystemD | SystemD | OpenRC | — |
| Підтримка заліза | Відмінна (HWE) | Консервативна | Стабільна | Мінімальна | — |
| SELinux/AppArmor | AppArmor | AppArmor | SELinux | — | — |
| Основна сфера | Хмари, стартапи | Власні сервери | Enterprise | Контейнери | K8s-ноди |
| Snap | Так ⚠️ | Ні | Ні | Ні | Ні |

**Кейс 1: Стартап, хмара, швидкий старт**
→ **Ubuntu Server LTS**. Дефолтний образ у всіх хмарах (AWS, GCP, Azure, Hetzner), найбільша база знань у мережі, ідеальна інтеграція з `cloud-init`. Canonical дає 5 років підтримки безкоштовно. Зверніть увагу на Snap-проблеми при налаштуванні моніторингу.

**Кейс 2: Enterprise, банк, SAP, Oracle**
→ **RHEL**. Комерційний soft від великих вендорів сертифікований виключно під RHEL. Якщо бюджету на підписку немає — **AlmaLinux** або **Rocky Linux** як безкоштовна дроп-ін заміна.

**Кейс 3: Інфраструктура "поставив і забув"**
→ **Debian**. Повністю некомерційний проєкт, не залежить від настроїв ради директорів корпорацій. Без Snap, без surprises, максимально передбачуваний. Ідеал для Proxmox-нод, bare-metal серверів, довгоживучих VPS.

**Кейс 4: Docker-контейнери, мікросервіси**
→ **Alpine Linux** (всередині контейнерів). Для хост-машин Kubernetes-нод — Ubuntu або RHEL-клони через кращу сумісність із системними демонами, eBPF та мережевими драйверами.

**Кейс 5: Kubernetes-кластер з нуля, максимальна безпека**
→ **Talos Linux**. Immutable ОС без shell, управління через API. Мінімальна surface attack, ідеальна відтворюваність нод.

---

## Резюме

Якщо перед вами завдання розгорнути нову інфраструктуру і ви вагаєтесь у виборі — **обирайте Ubuntu**. Вона покриває 90% серверних потреб сучасного IT і прощає найбільше помилок.

Але пам'ятайте кілька речей, які відрізняють досвідченого інженера від початківця:

- **EOL — перше, що перевіряють** перед вибором дистрибутива. Поставити сервер на ОС без підтримки через 2 роки — це технічний борг з відомою датою вибуху.
- **Snap** — вимикайте або враховуйте в моніторингу на Ubuntu-серверах.
- **SELinux не вимикають** на RHEL-продакшені. Його вчать і налаштовують.
- **Alpine** — для контейнерів, не для нод. musl несумісний з glibc-бінарниками.
- **eBPF-фічі** залежать від версії ядра хост-машини, а не контейнера.
- **Ansible-ролі** пишіть одразу з урахуванням `ansible_os_family` — навіть якщо зараз у вас тільки один дистрибутив.

Принципова різниця між дистрибутивами — це зовнішня "обгортка". Якщо ви розумієте як працює Linux на рівні ядра, як функціонують системні виклики та мережевий стек — адаптація до будь-якої системи займе максимум кілька днів.
