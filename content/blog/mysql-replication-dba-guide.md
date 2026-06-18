+++
title = "MySQL Replication і DBA: повний гайд для DevOps — типи реплікації, InnoDB, моніторинг, Redis"
date = 2026-06-18T12:00:00+03:00
draft = false
tags = ["mysql", "dba", "replication", "innodb", "redis", "databases", "devops", "interview", "monitoring"]
categories = ["Databases"]
summary = "Повний розбір MySQL-реплікації: типи, технічна механіка, команди, моніторинг InnoDB, повний траблшутінг-гайд (зламана реплікація, lag, деадлоки, OOM, Galera split-brain), що зараз популярно (Vitess, ProxySQL, Percona), різниці з іншими базами, основи Redis і питання з відповідями для DevOps/DBA."
+++

---

## 1. Що таке репліка і навіщо вона потрібна

Простими словами: **реплікація** — це процес, коли одна база (master / source) постійно передає копію всіх своїх змін на одну або кілька інших баз (replica / slave), і та накатує ці зміни в себе, тримаючи дані синхронізованими.

Навіщо:

- **Відмовостійкість (HA)** — якщо master впав, є реплика, на яку можна перемкнутись (failover).
- **Масштабування читання (read scaling)** — весь SELECT-трафік розкидують по репліках, а на master лишають тільки INSERT/UPDATE/DELETE.
- **Бекапи без даунтайму** — бекап роблять з репліки, не навантажуючи продакшн-master.
- **Аналітика/звітність** — важкі звітні запити йдуть на окрему "аналітичну" репліку, щоб не класти прод.
- **Географічний розподіл** — репліки ближче до користувачів у різних регіонах (latency).
- **Міграції/тестування** — підняти репліку, перевірити на ній нову версію MySQL чи важку міграцію схеми.

Аналогія: master — це бухгалтер, який веде головну книгу і щохвилини диктує секретарці (replica), що саме він записав. Секретарка переписує те ж саме у свою книгу. Якщо бухгалтер захворіє — секретарка вже має майже повну копію і може тимчасово підмінити.

---

## 2. Архітектура: як технічно влаштована реплікація MySQL

### 2.1 Binary Log (binlog) — серце реплікації

`binlog` — це послідовний лог усіх змін даних (і структури), які master записує на диск. Без увімкненого binlog реплікація неможлива в принципі.

```ini
# /etc/mysql/my.cnf — мінімальна конфігурація master
[mysqld]
server-id = 1                      # унікальний ID у топології, ОБОВ'ЯЗКОВО різний на кожному вузлі
log_bin = /var/lib/mysql/binlog     # шлях + префікс файлів binlog (binlog.000001, binlog.000002...)
binlog_format = ROW                 # ROW / STATEMENT / MIXED — детальніше нижче
binlog_expire_logs_seconds = 604800 # автоочистка старих binlog через 7 днів (раніше — expire_logs_days)
sync_binlog = 1                     # 1 = fsync binlog на кожен commit (безпечно, повільніше)
gtid_mode = ON                      # включити GTID (рекомендовано для сучасних версій)
enforce_gtid_consistency = ON       # заборонити транзакції, що ламають GTID-консистентність
```

**Що, де, чому:**
- `server-id` — без унікального ID master/replica не зможуть розрізнити, хто кому що шле; колізія ID — класична причина "реплікація мовчки не працює".
- `sync_binlog=1` — кожен commit примусово фсинкається на диск перед підтвердженням клієнту. Це гарантує durability навіть якщо сервер впаде відразу після commit, але додає I/O навантаження. `sync_binlog=0` швидше, але при краші можна втратити останні транзакції з binlog.
- `binlog_expire_logs_seconds` — якщо забути це налаштувати, binlog-и можуть зʼїсти весь диск за тижні безперервного запису.

### 2.2 Що відбувається технічно, крок за кроком

1. Клієнт виконує `UPDATE`/`INSERT`/`DELETE` на **master**.
2. Зміна записується в **InnoDB redo log** (для durability транзакції) і одночасно формується **binlog event**.
3. На commit транзакції binlog event дописується у файл binlog (атомарно, в одному файлі з InnoDB-комітом через **2-phase commit** між InnoDB та binlog — це і гарантує консистентність даних і логу).
4. На master запускається **Binlog Dump Thread** — окремий тред на кожне підключення репліки, який читає binlog і стрімить події реплікам.
5. На **replica** є **IO Thread** — підключається до master (через `replication user`), отримує binlog events і пише їх **у себе** в **relay log** (це локальна копія тих самих подій, ще не застосована до даних).
6. Окремий **SQL Thread** (в сучасних версіях — **multi-threaded replication / MTS**, кілька паралельних worker-тредів) читає relay log і **застосовує** події до власних таблиць реплики.
7. Прогрес зберігається в `mysql.slave_relay_log_info` / `performance_schema.replication_applier_status` (раніше — у файлах `relay-log.info`, `master.info`).

```
MASTER                                    REPLICA
┌─────────────┐                      ┌─────────────────┐
│ клієнт пише │                      │                  │
│   UPDATE    │                      │                  │
└──────┬──────┘                      │                  │
       ▼                             │                  │
┌─────────────┐   binlog dump thread │  IO Thread       │
│   binlog    ├────────────────────► │  (мережа)        │
│  (на диску) │                      │       │          │
└─────────────┘                      │       ▼          │
                                      │  relay log       │
                                      │  (на диску)      │
                                      │       │          │
                                      │       ▼          │
                                      │  SQL Thread(и)   │
                                      │  (apply events)  │
                                      │       │          │
                                      │       ▼          │
                                      │  власні таблиці  │
                                      └──────────────────┘
```

Чому це важливо розуміти: **затримка реплікації (replication lag)** — це не одне число, а сума двох незалежних затримок: (а) скільки часу IO Thread тягне дані по мережі і (б) скільки часу SQL Thread встигає їх застосувати. Можна мати ідеальну мережу, але повільний disk I/O на репліці — і lag буде рости саме на фазі apply.

### 2.3 Чому одна реплика може відставати, навіть якщо вона потужніша за master

Класична пастка на інтерв'ю: "У нас replica на залізі потужнішому за master, але вона постійно відстає. Чому?"

Історично (до MySQL 5.6) SQL Thread був **один**, і він застосовував binlog events **строго послідовно**, в один потік — навіть якщо на master 50 паралельних з'єднань одночасно писали в різні таблиці. Реплика фізично не могла "розпаралелити" застосування, тому будь-яке потужне залізо не допомагало — бутылочне горлечко було архітектурне, не апаратне.

Рішення — **Multi-Threaded Replication (MTS)**, з'явилось у 5.6+:

```ini
# на replica
slave_parallel_workers = 8          # кількість паралельних SQL worker-тредів (сучасна назва: replica_parallel_workers)
slave_parallel_type = LOGICAL_CLOCK # групує транзакції, що commit-ились одночасно на master, для паралельного apply
```

`LOGICAL_CLOCK` дозволяє паралельно застосовувати транзакції, які на master комітились конкурентно (а значить — не залежали одна від одної). Якщо паралелізм усе ще недостатній — означає, що на master транзакції йдуть переважно послідовно (наприклад, весь трафік у одну "гарячу" таблицю), і тут MTS вже не врятує.

---

## 3. Типи реплікації — класифікація по трьох осях

Це питання "які є типи реплікації" насправді складається з трьох незалежних осей, і їх часто плутають. Розберемо кожну окремо.

### 3.1 Ось №1 — за способом синхронізації (наскільки master чекає на replica)

**Asynchronous replication (за замовчуванням)**
Master комітить транзакцію і відразу відповідає клієнту "OK", **не чекаючи**, поки хоч одна реплика підтвердить отримання даних. Це найшвидший варіант, але якщо master впаде відразу після commit — той commit може **ніколи не дійти** до реплік → втрата даних (data loss) при failover.

**Semi-synchronous replication (semi-sync)**
Master комітить локально, але **перед тим, як підтвердити клієнту**, чекає підтвердження (ACK), що **хоча б одна** реплика отримала і записала транзакцію у свій relay log (саме отримала, не обов'язково застосувала!). Це зменшує (але не зводить до нуля) ймовірність втрати даних і додає невелику затримку (network round-trip).

```sql
-- На master:
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET GLOBAL rpl_semi_sync_master_enabled = 1;
SET GLOBAL rpl_semi_sync_master_timeout = 10000; -- мс; якщо жодна репліка не відповіла за 10с — fallback в async

-- На replica:
INSTALL PLUGIN rpl_semi_sync_replica SONAME 'semisync_replica.so';
SET GLOBAL rpl_semi_sync_replica_enabled = 1;
```

Важливий нюанс: якщо всі semi-sync реплики недоступні і `timeout` минув — master **автоматично перемикається в async режим**, щоб не зупинити продакшн. Це означає, що "semi-sync" — не залізна гарантія, а best-effort з деградацією.

**Synchronous (virtually synchronous) replication**
Тут транзакція **не вважається завершеною**, поки кворум вузлів не підтвердив її застосування (certification). В MySQL це реалізовано не в "класичній" реплікації, а в окремих технологіях:
- **Galera Cluster** (Percona XtraDB Cluster, MariaDB Galera Cluster)
- **MySQL Group Replication** (і InnoDB Cluster на його основі)

Деталі — у розділі 3.4.

### 3.2 Ось №2 — за форматом запису в binlog (що саме реплікується)

**Statement-Based Replication (SBR)** — у binlog пишеться **сам SQL-запит** (текст), і replica повторно виконує цей самий запит у себе.
- Плюс: компактний binlog, добре читається людиною.
- Мінус: **недетерміновані функції** (`NOW()`, `UUID()`, `RAND()`, `CONNECTION_ID()`) можуть дати різний результат на master і на replica → розсинхрон даних. Це класична каверзна тема на інтерв'ю.

**Row-Based Replication (RBR)** — у binlog пишуться **фактичні зміни рядків** (before/after image), а не SQL-текст.
- Плюс: 100% детермінізм — реплікуються вже обчислені значення, без UUID()/NOW() проблем.
- Мінус: binlog більший за розміром (особливо при `UPDATE ... WHERE` на мільйони рядків), важче читати людьми (events бінарні, треба `mysqlbinlog` з декодуванням).
- Це **рекомендований/дефолтний формат** у сучасних версіях MySQL.

**Mixed-Based Replication (MBR)** — MySQL сам вирішує: для "безпечних" детермінованих запитів пише STATEMENT, для недетермінованих — автоматично перемикається на ROW.

```sql
-- Подивитись поточний формат
SHOW VARIABLES LIKE 'binlog_format';

-- Подивитись вміст бінарного лога в людському вигляді
mysqlbinlog --base64-output=decode-rows -v /var/lib/mysql/binlog.000045
```

### 3.3 Ось №3 — за топологією (хто з ким і як з'єднаний)

**Source → Replica (раніше Master-Slave)** — класична топологія "один master, N реплік". Найпростіша, найпоширеніша.

**Master-Master (Dual-Master / Active-Active)** — два сервери, кожен є master для іншого (циклічна реплікація). Технічно працює, але на практиці **дуже ризиковано**: якщо на обох вузлах одночасно прийде write в один і той самий рядок — отримаєш конфлікт, який класична async-реплікація **не вміє автоматично вирішувати** (на відміну від Galera, де є certification-based conflict resolution). Тому Master-Master в "чистому" вигляді зараз вважається анти-паттерном; замість нього використовують Galera/Group Replication, де multi-master зроблено правильно — з conflict detection.

**Chained replication (Relay / Intermediate Master)** — Master → Replica-A (яка одночасно є relay-джерелом) → Replica-B. Replica-A передає далі ті ж binlog events, які отримала від master, не "перезаписуючи" їх власноруч.

```ini
# на Replica-A (intermediate master)
log_slave_updates = ON   # КРИТИЧНО: без цього параметра Replica-A не пише
                          # отримані від master зміни у власний binlog,
                          # і Replica-B нічого не отримає
```

Навіщо chained: розвантажити master від великої кількості Binlog Dump Thread-ів (кожне підключення реплік — це окремий тред і I/O на master), особливо при географічно розкиданих репліках (один "хаб" у регіоні, що далі роздає локальним репліками).

**Multi-Source Replication** — одна replica одночасно отримує дані з **кількох незалежних master**-ів (у різні "канали", `CHANNEL`). Використовується для консолідації даних з кількох шардів/джерел в одну аналітичну базу.

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='master1.internal',
  SOURCE_USER='repl',
  SOURCE_PASSWORD='secret',
  SOURCE_AUTO_POSITION=1
  FOR CHANNEL 'from-master1';

CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='master2.internal',
  SOURCE_USER='repl',
  SOURCE_PASSWORD='secret',
  SOURCE_AUTO_POSITION=1
  FOR CHANNEL 'from-master2';

START REPLICA FOR CHANNEL 'from-master1';
START REPLICA FOR CHANNEL 'from-master2';
```

### 3.4 Galera Cluster та Group Replication — справжній multi-master

Це окремий клас рішень, де всі вузли рівноправні (multi-master), і запис можна робити **в будь-який вузол одночасно**, без класичних binlog dump/IO/SQL тредів.

**Galera Cluster** (Percona XtraDB Cluster, MariaDB Galera):
- Працює через **certification-based replication**: коли транзакція готова до commit на одному вузлі, вона розсилається всім іншим вузлам через груповий комунікаційний шар (wsrep API), і кожен вузол перевіряє (certify) — чи не конфліктує вона з іншими транзакціями, що паралельно виконуються в кластері.
- Якщо конфлікт — одна з транзакцій **відкатується** (rollback) з помилкою deadlock на клієнтській стороні; клієнт має retry-логіку.
- Це фактично synchronous replication: commit вважається успішним лише після certification на всіх вузлах.
- Класична проблема — **flow control**: якщо один вузол повільніший за інших (слабше залізо, повільний диск), кластер штучно сповільнює запис на всіх вузлах, щоб слабкий вузол не відстав критично.

```bash
# приклад статусу Galera-вузла
mysql -e "SHOW STATUS LIKE 'wsrep_%';" | grep -E "cluster_size|local_state_comment|flow_control_paused"
```

**MySQL Group Replication** — офіційна (Oracle) альтернатива Galera, на базі **Paxos-подібного протоколу** для консенсусу. Може працювати в **Single-Primary Mode** (тільки один вузол приймає write, інші — read-only репліки, з автоматичним failover) або **Multi-Primary Mode** (аналог Galera). На основі Group Replication будується **InnoDB Cluster** + **MySQL Router** (проксі, що автоматично направляє трафік на актуальний primary).

**Коротке порівняння:**

| | Async | Semi-sync | Galera / Group Replication |
|---|---|---|---|
| Гарантія доставки | Немає | "Хоч одна репліка отримала" | Консенсус усього кластера |
| Затримка запису | Найменша | +RTT до однієї репліки | +RTT до кворуму вузлів |
| Multi-master "з коробки" | Ні (ризиковано) | Ні | Так, із conflict detection |
| Складність підтримки | Низька | Низька | Висока (flow control, SST/IST, quorum) |

### 3.5 GTID vs Binlog File:Position — як реплікація "знає, де вона зупинилась"

**Класичний (legacy) спосіб** — реплікація відстежує позицію як пару `(назва_файлу_binlog, offset_в_байтах)`:

```sql
CHANGE MASTER TO
  MASTER_HOST='master.internal',
  MASTER_LOG_FILE='binlog.000045',
  MASTER_LOG_POS=15829203;
```

Проблема: при failover (коли стара replica стає новим master) потрібно **вручну** вирахувати правильний файл і позицію для всіх інших реплік — легко помилитись, легко "загубити" або "продублювати" транзакції.

**GTID (Global Transaction Identifier)** — кожній транзакції автоматично присвоюється глобально унікальний ID у форматі `server_uuid:transaction_id`. Реплікація відстежує **набір застосованих GTID**, а не файл/байт.

```sql
-- з GTID все, що треба — це сказати "продовжуй з того місця, де зупинився"
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='new-master.internal',
  SOURCE_USER='repl',
  SOURCE_PASSWORD='secret',
  SOURCE_AUTO_POSITION=1;   -- MySQL сам обчислює потрібну позицію за GTID
```

Переваги GTID: автоматичний failover без ручних обчислень позицій, легше переключати репліку на новий master, вбудована захист від "застосування тієї самої транзакції двічі" (`enforce_gtid_consistency`). Майже всі сучасні failover-інструменти (orchestrator, MHA, ProxySQL) розраховані саме на GTID.

---

## 4. Налаштування реплікації руками — повний покроковий приклад

Класичний сценарій: піднімаємо нову Source → Replica топологію з GTID.

```bash
# === КРОК 1: на MASTER — створюємо користувача для реплікації ===
mysql -u root -p << 'SQL'
CREATE USER 'repl'@'10.0.%' IDENTIFIED WITH mysql_native_password BY 'StrongPassHere';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'10.0.%';
-- REPLICATION SLAVE — спеціальний привілей, що дозволяє ТІЛЬКИ читати binlog,
-- без доступу до самих даних (репліка читає дані вже через сам процес реплікації)
FLUSH PRIVILEGES;
SQL
```

```bash
# === КРОК 2: знімаємо консистентний снепшот даних master ===
# Варіант А (для невеликих баз, з даунтаймом запису):
mysql -u root -p -e "FLUSH TABLES WITH READ LOCK;"   # блокує всі write, але дозволяє SELECT
mysqldump -u root -p --all-databases --master-data=2 --single-transaction > /backup/dump.sql
# --master-data=2 ВПИШЕ у dump.sql коментар з поточною GTID/позицією binlog у момент дампа —
# саме звідси replica дізнається, з якої точки починати реплікацію
mysql -u root -p -e "UNLOCK TABLES;"

# Варіант Б (продакшн без даунтайму, на InnoDB):
xtrabackup --backup --target-dir=/backup/full \
  --user=root --password=xxx
xtrabackup --prepare --target-dir=/backup/full
# Percona XtraBackup робить hot-бекап без FLUSH TABLES WITH READ LOCK для InnoDB-таблиць —
# критично для великих баз, де лок на запис навіть на секунди недопустимий
```

```bash
# === КРОК 3: переносимо дамп/бекап на replica і відновлюємо ===
scp /backup/dump.sql replica1:/tmp/
mysql -u root -p < /tmp/dump.sql
```

```sql
-- === КРОК 4: на REPLICA — конфігуруємо джерело і запускаємо ===
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = 'master.internal.lan',
  SOURCE_PORT = 3306,
  SOURCE_USER = 'repl',
  SOURCE_PASSWORD = 'StrongPassHere',
  SOURCE_AUTO_POSITION = 1,             -- 1 = використовувати GTID замість файл:позиція
  SOURCE_SSL = 1;                        -- реплікація трафіку має йти через TLS, особливо між дата-центрами

START REPLICA;

-- === КРОК 5: перевірка стану ===
SHOW REPLICA STATUS\G
```

Ключові поля з `SHOW REPLICA STATUS` (раніше `SHOW SLAVE STATUS`), на які завжди дивляться першими:

```text
Replica_IO_Running: Yes        -- IO Thread живий і тягне дані з master
Replica_SQL_Running: Yes       -- SQL Thread живий і застосовує relay log
Seconds_Behind_Source: 0       -- НАСКІЛЬКИ репліка відстає (раніше Seconds_Behind_Master)
Last_IO_Error:                 -- якщо тут не пусто — проблема з мережею/правами
Last_SQL_Error:                -- якщо тут не пусто — конфлікт даних/дублікат ключа при apply
Retrieved_Gtid_Set:             -- які GTID отримані по мережі
Executed_Gtid_Set:               -- які GTID реально застосовані
```

Якщо `Replica_IO_Running: No` — проблема на рівні мережі/автентифікації/прав. Якщо `Replica_SQL_Running: No` — проблема на рівні даних (наприклад, дублікат ключа: репліку записали руками, і вона розійшлась з master).

### 4.1 Корисні команди в щоденній роботі

```sql
-- Поточна позиція master (де він зараз пише binlog)
SHOW MASTER STATUS;
SHOW BINARY LOG STATUS;        -- сучасна назва тієї ж команди

-- Список усіх файлів binlog, що зберігаються на диску
SHOW BINARY LOGS;

-- Зупинити/запустити реплікацію (наприклад, перед обслуговуванням)
STOP REPLICA;
START REPLICA;

-- Зупинити тільки SQL Thread, лишивши IO Thread (накопичувати relay log, не застосовуючи —
-- корисно, якщо треба "притримати" репліку, не відключаючи її від мережі)
STOP REPLICA SQL_THREAD;

-- Перезапустити реплікацію "з нуля" (видаляє всю інформацію про джерело!)
STOP REPLICA;
RESET REPLICA ALL;             -- ОБЕРЕЖНО: видаляє SOURCE_HOST/USER/GTID-позицію повністю

-- Пропустити одну "зламану" транзакцію вручну (аварійний костиль, не для продакшн-звички)
STOP REPLICA;
SET GTID_NEXT='aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee:42';
BEGIN; COMMIT;                  -- "імітуємо" виконання проблемної транзакції порожньою
SET GTID_NEXT='AUTOMATIC';
START REPLICA;

-- Перевірити, чи master і replica реально синхронні по даних (а не тільки по позиції)
-- через Percona pt-table-checksum / pt-table-sync (зовнішні утиліти, не вбудовані)
pt-table-checksum --replicate=test.checksums h=master.internal.lan
```

---

## 5. Хто і як займається репліками: ролі, авто-failover, проксі

### 5.1 Хто відповідає за що

- **DBA (Database Administrator)** — власник усієї стратегії: яка топологія, які параметри durability/consistency, capacity planning, performance tuning запитів, бекапи, права доступу.
- **SRE/DevOps** — інфраструктура навколо бази: моніторинг, алертинг, автоматизація провижену (Ansible/Terraform), failover-оркестрація, networking/firewall між вузлами, диски/RAID під дані (тут і твій профіль — моніторинг через Prometheus/Grafana, Ansible-плейбуки для розгортання).
- На практиці в невеликих/середніх компаніях ці дві ролі часто зливаються в одну людину (DevOps-інженер, що "теж трохи DBA").

### 5.2 Автоматичний failover — чому "ручний" failover це погана ідея в 2026

Якщо master впав, потрібно: (1) визначити, яка з реплік найбільш "свіжа" (найменший lag / найбільший Executed_Gtid_Set), (2) промоутнути її в master, (3) переналаштувати інші репліки слухати новий master, (4) переключити трафік застосунку на нову адресу. Робити це руками під час інциденту вночі — рецепт для людської помилки.

**Інструменти для авто-failover:**

- **Orchestrator** (GitHub) — топологія-aware HA менеджер: будує карту master/реплік через `SHOW REPLICA STATUS`, детектує падіння master через множинні незалежні перевірки (uyникнення false positive через мережевий "сплеск"), сам обирає найкращого кандидата на промоушн і виконує `CHANGE REPLICATION SOURCE` на всіх інших репліках.
- **MHA (Master High Availability Manager)** — старіший, але ще зустрічається; робить схожу роботу, фокус на мінімізації втрати даних при failover.
- **ProxySQL** — це не failover-менеджер у чистому вигляді, а **proxy/router** перед MySQL: розрізняє read/write трафік (читання → на репліки, запис → на master), приховує від застосунку факт зміни топології (застосунок підключається до ProxySQL, а не напряму до MySQL-вузла), має вбудовану логіку query routing, query caching, connection pooling.
- **Vitess** — окрема велика тема (розділ 7), включає власний шар оркестрації і шардингу.

### 5.3 Типова продакшн-схема разом

```
                     ┌─────────────┐
  застосунок ───────►│  ProxySQL   │ (read/write split, connection pooling)
                     └──────┬──────┘
              write ────────┼──────── read
                     ▼              ▼
              ┌───────────┐   ┌───────────┐   ┌───────────┐
              │  MASTER   │──►│ REPLICA-1 │   │ REPLICA-2 │
              └───────────┘   └───────────┘   └───────────┘
                     ▲
                     │ детектує падіння, виконує промоушн
              ┌───────────┐
              │Orchestrator│
              └───────────┘
```

---

## 6. InnoDB глибше: що треба знати, щоб впевнено говорити про "движок бази"

InnoDB — це **storage engine** (механізм зберігання даних) MySQL за замовчуванням з версії 5.5+, і саме про нього зараз 95% усіх питань на інтерв'ю по "движках бази".

### 6.1 Ключові внутрішні структури

- **Buffer Pool** — основний кеш у RAM. Тримає сторінки даних та індексів. Чим більший buffer pool (`innodb_buffer_pool_size`), тим менше дискових I/O на читання — головний параметр для тюнінгу пам'яті.
- **Redo Log** (`innodb_log_file_size` / `innodb_redo_log_capacity` у новіших версіях) — circular-буфер на диску, куди записуються **фізичні** зміни сторінок ще ДО того, як вони фізично потраплять у сам файл даних (.ibd). Це і є механізм **durability**: якщо MySQL впаде, при старті він "доганяє" незакомічені зміни з redo log (**crash recovery**).
- **Undo Log** — зберігає "попередні версії" рядків, потрібен для (а) rollback транзакцій і (б) **MVCC** (Multi-Version Concurrency Control) — щоб інша транзакція, яка читає той самий рядок паралельно, бачила консистентний "знімок" даних, не чекаючи на лок.
- **Doublewrite Buffer** — захист від "partial page write": якщо сторінка (зазвичай 16KB) записується на диск, а диск фізично пише блоками меншого розміру (наприклад, 4KB сектор), крах посередині запису може дати "напіврозбиту" сторінку. Doublewrite спочатку пише копію сторінки в спеціальну безпечну область, і якщо основний запис не вдався — відновлює з цієї копії.
- **Adaptive Hash Index** — InnoDB сам, автоматично, будує hash-індекс над "гарячими" частинами B-Tree індексу, якщо бачить, що до них іде багато точкових (equality) запитів.

### 6.2 ACID і як InnoDB його забезпечує

- **Atomicity** — undo log: або всі зміни транзакції застосовані, або жодна (rollback по undo).
- **Consistency** — constraints, foreign keys, тригери — гарантують, що дані не порушують бізнес-правила.
- **Isolation** — рівні ізоляції + MVCC + locking (нижче).
- **Durability** — redo log + `sync_binlog`/`innodb_flush_log_at_trx_commit`.

```ini
innodb_flush_log_at_trx_commit = 1   # 1 = fsync redo log на КОЖЕН commit (максимальна durability, найповільніше)
                                      # 2 = записати в OS cache на commit, fsync раз/сек (швидше, ризик втрати ~1с даних при краху ОС)
                                      # 0 = писати й fsync-ати раз/сек незалежно від commit (найшвидше, найризиковано)
```

Це один з найпопулярніших "практичних" компромісів durability vs throughput, про які питають на інтерв'ю: `1+1` (innodb_flush_log_at_trx_commit=1, sync_binlog=1) — найбезпечніше і рекомендовано для master із критичними даними (наприклад, фінансові транзакції); комбінації з 0/2 — для систем, де допустима втрата останньої секунди даних в обмін на throughput.

### 6.3 Рівні ізоляції транзакцій (isolation levels)

| Рівень | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| READ UNCOMMITTED | можливий | можливий | можливий |
| READ COMMITTED | ні | можливий | можливий |
| REPEATABLE READ (дефолт в InnoDB) | ні | ні | ні* |
| SERIALIZABLE | ні | ні | ні |

\* В InnoDB `REPEATABLE READ` додатково захищає від phantom read через **gap locks** / **next-key locks** — це нестандартна (сильніша за SQL-стандарт) поведінка, і про це теж люблять питати: "Чому в InnoDB REPEATABLE READ вже захищає від phantom read, хоча по стандарту SQL це мав робити лише SERIALIZABLE?"

```sql
SELECT @@transaction_isolation;
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

### 6.4 Locking: row-level vs table-level, gap locks, deadlocks

InnoDB робить **row-level locking** (на відміну від MyISAM, що блокує цілу таблицю на запис) — це і є головна причина, чому InnoDB набагато краще масштабується на конкурентний запис.

- **Record Lock** — лок на конкретний існуючий рядок.
- **Gap Lock** — лок на "проміжок" між значеннями індексу (навіть якщо рядків там немає!) — захищає від вставки нових рядків у цей діапазон іншою транзакцією, поки перша не завершилась.
- **Next-Key Lock** — комбінація Record Lock + Gap Lock одночасно (дефолтна поведінка в REPEATABLE READ).

```sql
-- Подивитись поточні локи й транзакції, що чекають
SELECT * FROM performance_schema.data_locks;
SELECT * FROM performance_schema.data_lock_waits;

-- Класична InnoDB-команда для діагностики (включає й секцію LATEST DETECTED DEADLOCK)
SHOW ENGINE INNODB STATUS\G
```

**Deadlock** — два транзакції взаємно блокують одна одну (A чекає лок, що тримає B; B чекає лок, що тримає A). InnoDB сам детектує цикл і **примусово відкочує** одну з транзакцій (ту, що "дешевша" для rollback), повертаючи клієнту помилку `Deadlock found when trying to get lock`. Правильна поведінка застосунку — retry-логіка на цю конкретну помилку.


## 7. Моніторинг InnoDB і реплікації на практиці (Prometheus/Grafana stack)

Раз ти вже живеш у Prometheus/Grafana — ось конкретно, що й навіщо знімати з MySQL.

### 7.1 Джерела метрик

```bash
# mysqld_exporter — стандартний шлях метрик MySQL у Prometheus
# Створюємо окремого користувача з мінімальними правами саме для моніторингу
mysql -u root -p << 'SQL'
CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'ExporterPass' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
-- PROCESS — бачити SHOW PROCESSLIST/engine status
-- REPLICATION CLIENT — дозволяє виконувати SHOW REPLICA STATUS / SHOW MASTER STATUS
SQL
```

```yaml
# docker-compose фрагмент, типовий для твого стеку
mysqld-exporter:
  image: prom/mysqld-exporter
  environment:
    - DATA_SOURCE_NAME=exporter:ExporterPass@(mysql:3306)/
  command:
    - "--collect.info_schema.innodb_metrics"
    - "--collect.engine_innodb_status"
    - "--collect.slave_status"          # критично для метрик реплікації
    - "--collect.global_status"
    - "--collect.global_variables"
  ports:
    - "9104:9104"
```

### 7.2 Метрики, на які варто ставити алерти

**Реплікація:**
```promql
# Lag реплікації в секундах — найважливіша метрика взагалі
mysql_slave_status_seconds_behind_master > 30

# IO/SQL Thread впали
mysql_slave_status_slave_io_running == 0
mysql_slave_status_slave_sql_running == 0
```

**InnoDB Buffer Pool:**
```promql
# Hit ratio буферного пулу — якщо падає нижче ~95%, означає що
# буфер замалий для робочого набору даних, диск читається частіше ніж треба
(1 - (mysql_global_status_innodb_buffer_pool_reads / mysql_global_status_innodb_buffer_pool_read_requests)) * 100
```

**Locks / Threads:**
```promql
mysql_global_status_threads_running          # активні треди — раптовий стрибок = можливий стопор/lock contention
mysql_global_status_innodb_row_lock_waits     # скільки разів запити чекали на row lock
mysql_global_status_innodb_row_lock_time_avg  # середній час очікування на лок (мс)
```

**Connections / Slow queries:**
```promql
rate(mysql_global_status_slow_queries[5m])
mysql_global_status_threads_connected / mysql_global_variables_max_connections
```

### 7.3 Performance Schema і sys schema — вбудована "обсерваторія"

```sql
-- Топ запитів за сумарним часом виконання (найкорисніший запит для performance tuning)
SELECT digest_text, count_star, sum_timer_wait/1000000000000 AS total_sec
FROM performance_schema.events_statements_summary_by_digest
ORDER BY sum_timer_wait DESC LIMIT 10;

-- sys schema — людиночитабельні view над performance_schema
SELECT * FROM sys.statement_analysis ORDER BY total_latency DESC LIMIT 10;
SELECT * FROM sys.schema_unused_indexes;     -- індекси, які ніхто не використовує (баласт на write)
SELECT * FROM sys.innodb_lock_waits;          -- хто кого блокує прямо зараз
```

### 7.4 Percona Monitoring and Management (PMM)

Якщо хочеш готове рішення "з коробки" замість збирання власного дашборду — **PMM** від Percona розгортається одним Docker-контейнером, сам тягне метрики через `mysqld_exporter`, додає готові Grafana-дашборди саме під MySQL/InnoDB/реплікацію, плюс **Query Analytics (QAN)** — окремий UI для аналізу повільних запитів з розбивкою по часу. Для твого випадку (вже є власний Prometheus/Grafana) логічніше інтегрувати `mysqld_exporter` в існуючий стек, а PMM розглядати, якщо MySQL стає основним фокусом, а не одним з компонентів великої інфраструктури.

---

## 8. Що зараз найбільш популярне в екосистемі MySQL (2026)

- **InnoDB** — беззаперечний стандарт для будь-яких транзакційних навантажень; `MyISAM` лишився хіба що в legacy-системах (немає транзакцій, table-level locking, але трохи компактніший для read-only/архівних таблиць).
- **MySQL vs MariaDB vs Percona Server** — три відгалуження одного коріння:
  - **MySQL (Oracle)** — офіційна версія, найновіші фічі (Group Replication, Document Store з JSON).
  - **MariaDB** — форк від оригінальних розробників MySQL після покупки Oracle; має власні фічі (інший Galera-інтеграція з коробки, інший optimizer в деяких версіях). Несумісність з MySQL поступово зростає з кожною major-версією.
  - **Percona Server for MySQL** — drop-in заміна MySQL з додатковими enterprise-фічами безкоштовно (розширена діагностика, XtraDB engine історично, краща Performance Schema деталізація). Часто вибір для тих, хто хоче MySQL-сумісність + краще тулінг.
- **ProxySQL** — фактичний стандарт для read/write split і connection pooling перед MySQL-кластером, незалежно від того InnoDB Cluster, Galera чи звичайна async-реплікація позаду нього.
- **Vitess** — створений у YouTube, тепер CNCF graduated проєкт. Розв'язує проблему **шардингу** на рівні, де "звичайна" реплікація вже не масштабується горизонтально (мільярди рядків в одній таблиці). Vitess представляє кластер шардованих MySQL-інстансів як **одну логічну базу** для застосунку (через VTGate proxy), сам керує resharding-ом без даунтайму. Активно використовується там, де Kubernetes-native управління базою — вимога (Vitess Operator для k8s).
- **Group Replication / InnoDB Cluster** — офіційний шлях Oracle до multi-master HA без сторонніх Galera-патчів.

---

## 9. Бекапи і Point-in-Time Recovery (PITR) — must-know для DBA

```bash
# Логічний бекап (повільніше, але портативний між версіями/платформами)
mysqldump --single-transaction --master-data=2 --routines --triggers --all-databases > full.sql

# Фізичний бекап (швидше відновлення на великих базах, специфічний до версії/engine)
xtrabackup --backup --target-dir=/backup/full

# PITR — відновлення "на конкретний момент часу між бекапами":
# 1. Відновлюємо останній full backup
# 2. Накатуємо binlog від позиції бекапа до потрібного моменту (наприклад, ПЕРЕД помилковим DELETE)
mysqlbinlog --start-position=15829203 --stop-datetime="2026-06-18 14:23:00" binlog.000045 | mysql -u root -p
```

**Правило 3-2-1**, яке варто згадати на інтерв'ю: 3 копії даних, на 2 різних типах носіїв, 1 копія off-site (інший дата-центр/регіон). Для MySQL це типово: локальний xtrabackup + копія в S3/Glacier-подібне сховище + репліка в іншому регіоні (яка сама по собі НЕ замінює бекап — людська помилка `DROP TABLE` реплікується теж).

---

## 10. MySQL vs PostgreSQL vs MongoDB — короткі практичні різниці

- **MySQL vs PostgreSQL** — обидві реляційні/ACID, але PostgreSQL історично сильніший у складних запитах (window functions, CTE з'явились раніше), розширюваності (custom types, extensions типу PostGIS), точнішій відповідності SQL-стандарту. MySQL історично простіший в адмініструванні, легший replication setup "з коробки", дуже широка індустріальна підтримка (більшість CMS/web-стеків). InnoDB робить MVCC через undo log; PostgreSQL — через зберігання кількох версій рядка прямо в таблиці (і потребує `VACUUM` для прибирання "мертвих" версій — концептуально інший підхід до того ж завдання).
- **MySQL vs MongoDB** — реляційна модель (таблиці, JOIN, жорстка схема, ACID-транзакції) проти документної моделі (JSON-подібні документи, гнучка схема, traditionally слабші гарантії консистентності, хоча сучасний MongoDB вже підтримує multi-document ACID-транзакції). MongoDB природньо шардиться "з коробки" (на відміну від MySQL, де шардинг — це або Vitess, або ручна робота).

---

## 11. Redis — коротко, що треба знати поруч з MySQL

Redis — це **in-memory** key-value хранилище (структури даних: string, hash, list, set, sorted set, stream), а не реляційна СУБД. Використовують як кеш перед MySQL, для сесій, rate-limiting, черг (через Streams/Lists), pub/sub.

### 11.1 Persistence — як Redis взагалі не втрачає все при перезапуску

- **RDB (snapshot)** — періодичний повний дамп усієї бази в файл (`save 900 1` — зробити снепшот, якщо за 900с змінився хоча б 1 ключ). Швидке відновлення, але можлива втрата змін між снепшотами.
- **AOF (Append Only File)** — лог кожної write-команди (схоже на binlog у MySQL). `appendfsync everysec` (компромісний дефолт) / `always` (максимальна durability, повільніше) / `no` (швидко, ризиковано).
- Можна (і часто варто) комбінувати RDB + AOF одночасно.

### 11.2 Реплікація в Redis

- **Master-Replica** — асинхронна за замовчуванням, концептуально схожа на MySQL async replication: replica підключається, отримує RDB-снепшот, потім стрімить подальші команди.
- **Redis Sentinel** — окремий процес моніторингу для авто-failover у простих master-replica топологіях (аналог ролі orchestrator для MySQL, тільки вбудований у сам Redis-екосистему).
- **Redis Cluster** — горизонтальний шардинг (16384 hash slots, розподілені між вузлами), кожен шард має власні репліки. Це вже не просто HA, а й горизонтальне масштабування запису.

### 11.3 Найважливіша практична різниця від MySQL

Redis — однопотоковий (в основному циклі обробки команд, хоча є I/O threads для мережі в нових версіях) і тримає весь робочий набір **у RAM**. Це дає мікросекундну латентність, але означає: (а) дані обмежені обʼємом RAM, (б) це не заміна транзакційної реляційної бази для складних запитів з JOIN/агрегаціями — Redis оптимізований під O(1)/O(log n) точкові операції зі структурами даних, а не під реляційну аналітику.

---

## 12. Каверзні питання для інтерв'ю — з відповідями

**Питання: Master впав одразу після того, як клієнту повернулась відповідь "OK" на commit. Чи гарантовано ця транзакція є на репліці?**
Відповідь: Ні, якщо реплікація асинхронна — "OK" клієнту повертається відразу після локального commit на master, без очікування підтвердження від реплік. Транзакція могла фізично не встигнути потрапити навіть у binlog dump thread. Це і є фундаментальна причина існування semi-sync і Galera/Group Replication.

**Питання: Чим `Seconds_Behind_Master`/`Seconds_Behind_Source` НЕ є?**
Відповідь: Це не "різниця в часі між годинниками серверів", і це не точна метрика затримки мережі. Це різниця між часом останньої події, яку SQL Thread застосував, і поточним часом на репліці. Якщо реплікація просто "стоїть" (наприклад, IO Thread не може підключитись), ця метрика може показувати `NULL`, а не зростаюче число — і це окрема пастка для алертів (треба алертити і на `NULL`/Running=No, не лише на порогове значення секунд).

**Питання: Чому STATEMENT-based реплікація може "тихо" розсинхронити дані без жодної помилки в логах?**
Відповідь: Через недетерміновані вирази в SQL — `UPDATE t SET x = NOW()` або `INSERT ... VALUES (UUID())` виконані на master і повторно виконані (а не скопійовані як результат) на репліці можуть дати різні значення часу/UUID. Жодної помилки не буде — просто дані розійдуться непомітно, і це виявиться лише через checksums (`pt-table-checksum`) набагато пізніше.

**Питання: У Galera Cluster клієнт отримав помилку deadlock на простому одиночному `UPDATE`, без жодних паралельних транзакцій з боку застосунку. Як це можливо?**
Відповідь: Через certification-based replication конфлікт перевіряється на рівні всього кластера, включно з транзакціями, які прилетіли від реплікації з інших вузлів (а не лише локальні конкурентні запити). Сертифікація може "не пройти" навіть для одиночного UPDATE, якщо в той самий момент на іншому вузлі комітилась конфліктна транзакція — застосунок повинен мати retry-логіку саме на цей клас помилок як норму роботи з Galera, а не як аномалію.

**Питання: Чи захищає InnoDB REPEATABLE READ від phantom read без явного SERIALIZABLE?**
Відповідь: Так, на відміну від "чистого" SQL-стандарту — InnoDB реалізує REPEATABLE READ із gap locks/next-key locks саме для захисту від phantom read, тоді як стандарт ANSI SQL формально допускає phantom read на цьому рівні ізоляції. Це нюанс конкретної реалізації InnoDB, а не загальна властивість РБД.

**Питання: На сервері InnoDB Buffer Pool Hit Ratio показує 99.9%, але запити все одно повільні. Чому буфер тут не допомагає?**
Відповідь: Hit ratio показує лише ефективність кешування сторінок, а не якість запитів. Повільність може бути через відсутність потрібного індексу (full table scan навіть у пам'яті — повільний для великих таблиць), lock contention (запити чекають на row/gap locks, а не на диск), або погану структуру запиту (N+1, відсутність LIMIT, сортування великого набору без індексу). Перший крок діагностики тут — `EXPLAIN`/`EXPLAIN ANALYZE`, не buffer pool метрики.

**Питання: Чим відрізняється `STOP REPLICA SQL_THREAD` від просто `STOP REPLICA`?**
Відповідь: Перше лишає IO Thread живим — репліка продовжує тягнути binlog events з master і накопичувати їх у relay log на диску, просто не застосовує. Друге зупиняє обидва треди повністю. Перший варіант корисний, якщо потрібно "притримати" застосування змін (наприклад, для аналізу стану на конкретний момент), але дані продовжують прибувати і відповідно займати дисковий простір під relay log.

**Питання: GTID-набір на репліці містить транзакцію, якої немає на новому master після failover. Що станеться при спробі підключити цю репліку до нового master?**
Відповідь: MySQL виявить "errant transaction" — GTID, виконаний на репліці, але не існуючий в історії нового master, — і реплікація зупиниться з помилкою, замість того, щоб тихо щось зіпсувати. Це власне і є одна з переваг GTID над файл:позиція — явна детекція розбіжності історій замість мовчазного розсинхрону. Виправляється або `RESET MASTER` на репліці з повним ресинком, або інʼєкцією порожньої транзакції з тим самим GTID (`SET GTID_NEXT` + порожній `BEGIN/COMMIT`, як показано вище).

**Питання: Redis Sentinel промоутнув репліку в master під час split-brain мережевого розділення. Що могло піти не так?**
Відповідь: Якщо мережевий розрив ізолював старий master від Sentinel-кворуму, але клієнти застосунку (з іншого боку розриву) досі можуть писати в старий master — отримуємо два "masters" одночасно (split-brain), і дані, записані в "ізольований" master після промоушну нового, будуть втрачені, коли старий master зрештою повернеться і стане replica (full resync затирає його локальні дані). Захист — `min-replicas-to-write` на master (відмовляти у write, якщо немає достатньої кількості "живих" реплік) і коректний кворум Sentinel (мінімум 3, нечисле число вузлів-спостерігачів).


---

## 13. Траблшутінг бази даних — повний покроковий гайд "коли все погано"

Це окремий найважливіший блок для DevOps/SRE: на інтерв'ю питають не лише "що таке X", а "у тебе продакшн лежить, що ти робиш". Нижче — алгоритм по кожному типовому інциденту, з конкретними командами в порядку, в якому їх реально треба виконувати.

### 13.1 Загальний алгоритм для будь-якого інциденту з базою

Перш ніж лізти в конкретні симптоми — завжди один і той самий порядок дій:

1. **Стабілізувати, перш ніж діагностувати.** Якщо є read-репліки і трафік можна перемкнути — перемкни, щоб зняти тиск з проблемного вузла, поки розбираєшся.
2. **Зняти "знімок стану" одразу**, перш ніж щось рестартувати — після рестарту `SHOW ENGINE INNODB STATUS`, `SHOW PROCESSLIST` і `performance_schema` дані зникають назавжди, а саме вони потрібні для post-mortem.
3. **Подивитись логи в хронології, а не вгадувати.** `journalctl -u mysql --since "-15 min"` + `/var/log/mysql/error.log` — перше місце, куди дивимось, до будь-яких SQL-команд.
4. Тільки після збору даних — приймати рішення: рестарт, kill процесу, failover, тощо.

```bash
# ШВИДКИЙ ЗНІМОК СТАНУ перед будь-якими діями (зберегти кудись окремо для post-mortem)
mysql -e "SHOW ENGINE INNODB STATUS\G" > /tmp/incident_innodb_status.txt
mysql -e "SHOW FULL PROCESSLIST;" > /tmp/incident_processlist.txt
mysql -e "SHOW REPLICA STATUS\G" > /tmp/incident_replica_status.txt
mysql -e "SHOW GLOBAL STATUS;" > /tmp/incident_global_status.txt
journalctl -u mysql --since "-15 min" --no-pager > /tmp/incident_journal.txt
tail -n 200 /var/log/mysql/error.log > /tmp/incident_error_log.txt
```

### 13.2 Реплікація зламалась: `Replica_IO_Running` або `Replica_SQL_Running` = No

```sql
SHOW REPLICA STATUS\G
```

Дивимось саме в такому порядку:

1. **`Replica_IO_Running: No`** → проблема на рівні мережі/автентифікації, дані не доходять.
   - `Last_IO_Error` — текст помилки. Найчастіше: `Access denied`, `Can't connect to MySQL server`, `error reconnecting to master`.
   - Перевірка мережі і портів: `nc -zv master.internal.lan 3306` (або `telnet` за відсутності `nc`)
   - Перевірка прав: `SHOW GRANTS FOR 'repl'@'10.0.%';` на master
   - Перевірка, чи не зʼїв master binlog, на якому "зупинилась" репліка: якщо `binlog_expire_logs_seconds` спрацював раніше, ніж репліка встигла прочитати потрібний файл — IO Thread не знайде потрібну позицію взагалі. **Виправлення тут тільки одне — повний resync (новий snapshot + CHANGE REPLICATION SOURCE з нуля)**, тому це і є головна причина тримати `expire_logs` достатньо великим відносно максимально допустимого даунтайму репліки.

2. **`Replica_SQL_Running: No`** → дані дійшли, але apply зламався — це проблема **даних**, не мережі.
   - `Last_SQL_Error` — найчастіше: дублікат ключа (`Duplicate entry`), `Table doesn't exist`, foreign key violation.
   - Типова причина: хтось писав напряму в репліку (репліка мала бути read-only, але не була), і її дані розійшлись з master.

```sql
-- ОБОВ'ЯЗКОВЕ профілактичне налаштування, щоб уберегтись від п.2 назавжди:
SET GLOBAL read_only = 1;
SET GLOBAL super_read_only = 1;  -- захищає навіть від root-користувача, лишає виключення тільки для самого SQL Thread
```

```sql
-- Аварійний "пропуск" одної помилки (костиль, не звичка):
STOP REPLICA;
SET GLOBAL sql_replica_skip_counter = 1;  -- для legacy не-GTID реплікації
START REPLICA;

-- Для GTID-реплікації — інʼєкція порожньої транзакції з тим самим GTID (показано в розділі 4.1)
```

> **WARNING:** пропуск помилки (`skip_counter` або порожня GTID-транзакція) означає, що репліка **свідомо** розійшлась з master на цьому рядку. Завжди після такого фіксу запускати `pt-table-checksum`, щоб знати масштаб розбіжності, і планувати повний resync, якщо розбіжностей багато.

### 13.3 Реплікація відстає (lag росте, треди живі)

Якщо `Replica_IO_Running`/`Replica_SQL_Running` обидва `Yes`, а `Seconds_Behind_Source` росте — шукаємо бутылочне горлечко окремо на кожній фазі:

```sql
-- Фаза 1: чи встигає IO Thread тягнути дані з мережі?
-- Порівняй позицію, яку РЕПЛІКА вже ОТРИМАЛА (Retrieved_Gtid_Set) з позицією на MASTER (SHOW MASTER STATUS).
-- Якщо розрив тут — проблема мережі/bandwidth, не диска репліки.
SHOW REPLICA STATUS\G   -- дивимось Retrieved_Gtid_Set vs Executed_Gtid_Set

-- Фаза 2: чи встигає SQL Thread застосовувати relay log?
-- Якщо Retrieved ≈ Master, але Executed сильно позаду Retrieved — проблема саме в apply (диск/locks/одна гаряча таблиця)
SHOW PROCESSLIST;  -- шукаємо state системних SQL/worker тредів (System Lock, Waiting for table flush і т.д.)
```

Найчастіші причини повільного apply:
- **Single-threaded apply** на гарячу таблицю навіть з увімкненим MTS (всі транзакції йдуть в одну таблицю → паралелити нема чим, дивись 2.3).
- **Повільний диск на репліці** — `iostat -x 1` під час лагу: високий `%util`, високий `await`.
- **Великі неіндексовані UPDATE/DELETE** на master, що в ROW-форматі генерують величезні binlog events (наприклад, `DELETE FROM big_table WHERE old_date < ...` без індексу — на master теж повільно, а на репліці ще й однопотоково накатується).
- **Конкурентний read-навантаження на самій репліці** (важкі звітні SELECT займають CPU/I/O, який потрібен SQL Thread).

```bash
# Diск — головний підозрюваний при apply lag
iostat -x 1 5
iotop -o
```

### 13.4 База "висить" / усі запити повільні

```sql
-- Крок 1: скільки активних з'єднань і що вони роблять ПРЯМО ЗАРАЗ
SHOW FULL PROCESSLIST;
-- або зручніше через sys schema:
SELECT * FROM sys.session ORDER BY time DESC LIMIT 20;
```

Дивимось на колонку `State` — це і дає напрямок:

| State | Що означає |
|---|---|
| `Waiting for table metadata lock` | хтось тримає DDL-лок (ALTER TABLE), всі інші чекають |
| `Sending data` | повільний full scan / сортування великого набору без індексу |
| `Locked` / `Waiting for row lock` | row-level lock contention |
| `Copying to tmp table` | сортування/groupby не вкладається в `tmp_table_size`, пишеться на диск |
| `Waiting for table flush` | хтось зробив `FLUSH TABLES` / бекап і тримає глобальний лок |

```sql
-- Крок 2: хто кого блокує (актуальні запити для сучасних версій MySQL)
SELECT * FROM performance_schema.data_lock_waits;
SELECT * FROM sys.innodb_lock_waits;

-- Крок 3: InnoDB-специфічна секція — транзакції, що довго не комітяться, і поточні деадлоки
SHOW ENGINE INNODB STATUS\G
-- дивимось секції TRANSACTIONS і LATEST DETECTED DEADLOCK
```

```sql
-- Аварійна дія: вбити конкретний "зависший" запит/транзакцію (за Id з PROCESSLIST)
KILL QUERY 12345;     -- вбити тільки запит, з'єднання лишається
KILL 12345;           -- вбити все з'єднання повністю
```

> **WARNING:** `KILL` на транзакції, що вже частково записала зміни — викликає **rollback**, який сам може зайняти значний час і додатково навантажити сервер. Не панікувати, якщо після `KILL` навантаження ще кілька секунд/хвилин лишається високим.

### 13.5 "Too many connections" / неможливо підключитись

```sql
SHOW VARIABLES LIKE 'max_connections';
SHOW STATUS LIKE 'Threads_connected';
SHOW PROCESSLIST;   -- шукати, чи це один застосунок "зʼїв" усі конекшени (connection leak)
```

Швидкий рятувальний доступ, якщо ліміт зайнятий повністю і навіть адмін не може зайти:

```sql
-- MySQL резервує окремий слот понад max_connections саме для SUPER-користувача
SET GLOBAL max_connections = 500;   -- runtime-зміна без рестарту, якщо вистачає пам'яті на нові буфери з'єднань
```

Корінна причина майже завжди — або connection leak у застосунку (немає `close()`/pool не повертає конекшени), або відсутність connection pooling (ProxySQL/ProxySQL pooling саме для цього і існує — розділ 5.2).

### 13.6 Диск заповнюється

```bash
# Що саме їсть місце — перші підозрювані:
du -sh /var/lib/mysql/*           # binlog файли, ibdata1, окремі .ibd таблиці
du -sh /var/lib/mysql/binlog.*    # якщо binlog_expire_logs_seconds не налаштований/реплікація відстає
```

- **binlog накопичився** — якщо є відстаюча репліка, MySQL **не видалить** binlog файли, які ще потрібні цій репліці, навіть якщо `expire_logs_seconds` минув. Перевір `SHOW REPLICA STATUS` на всіх репліках, перш ніж форсити видалення.
- **ibdata1 росте і не зменшується після DELETE** — це нормальна поведінка InnoDB за замовчуванням (`innodb_file_per_table` зазвичай ON у сучасних версіях, але старий `ibdata1` міг накопичити "сміття" роками; зменшити можна лише повним `mysqldump` + перестворенням instance).
- **tmp-таблиці на диску** — `tmp_table_size`/`max_heap_table_size` замалі для важких GROUP BY/ORDER BY → MySQL переключається на дискові temp-таблиці в `tmpdir`.

```sql
-- Безпечне видалення старих binlog (перевір спочатку, що жодна репліка їх не потребує!)
PURGE BINARY LOGS BEFORE '2026-06-10 00:00:00';
```

### 13.7 Деадлоки повторюються постійно

```sql
SHOW ENGINE INNODB STATUS\G   -- секція LATEST DETECTED DEADLOCK — показує ОБИДВІ транзакції, їх запити і локи
```

Типові причини й фікси:
- **Різний порядок блокування рядків** у різних частинах застосунку (transaction A блокує rows у порядку 1→2, transaction B — 2→1) → завжди уніфікувати порядок (наприклад, сортувати ID перед UPDATE).
- **Відсутність індексу** на колонці в WHERE → InnoDB блокує більше рядків/gaps, ніж насправді потрібно, через full scan з локами.
- **Довгі транзакції**, що тримають локи довше необхідного (забутий `autocommit=0` без явного commit) → перевірка `information_schema.innodb_trx` на "старі" транзакції.

```sql
-- Знайти транзакції, що відкриті довше, ніж розумно (потенційні джерела deadlock/lock contention)
SELECT trx_id, trx_started, trx_mysql_thread_id, trx_query
FROM information_schema.innodb_trx
WHERE trx_started < NOW() - INTERVAL 60 SECOND
ORDER BY trx_started;
```

### 13.8 MySQL не стартує після краху

```bash
# Перше місце для діагностики — error log, не намагатись щось "фіксити" наосліп
tail -n 100 /var/log/mysql/error.log
journalctl -u mysql -n 100 --no-pager
```

- Якщо в логах `InnoDB: Database was not shutdown normally` — це нормально, InnoDB сам запускає **crash recovery** через redo log при старті; просто дай йому час (на великих базах може зайняти хвилини).
- Якщо crash recovery сам падає (рідко, але буває при пошкодженому диску) — `innodb_force_recovery` (від 1 до 6, зростаюча "агресивність" відмови від частини функціоналу заради можливості хоча б підняти сервер і витягти дамп):

```ini
[mysqld]
innodb_force_recovery = 1   # почати з 1, піднімати поступово ТІЛЬКИ якщо менше значення не допомогло
```

> **WARNING:** `innodb_force_recovery >= 4` забороняє будь-який запис у базу — використовується ВИКЛЮЧНО для того, щоб встигнути зробити `mysqldump` і перенести дані на чисту нову інсталяцію. Це режим "врятувати дані", не режим "продовжити роботу".

### 13.9 OOM killer вбив mysqld

```bash
# Підтвердження, що це саме OOM, а не crash самого MySQL
dmesg -T | grep -i "killed process"
journalctl -k --since "-1 hour" | grep -i oom
```

Найчастіша причина — `innodb_buffer_pool_size` виставлений занадто близько до фізичної RAM без запасу на OS, connection-буфери (`max_connections × per-connection буфери`), і файлову кеш-пам'ять. Класична математика, яку перевіряють на інтерв'ю:

```
RAM на mysqld ≈ innodb_buffer_pool_size
            + (max_connections × (sort_buffer_size + join_buffer_size + read_buffer_size + ...))
            + innodb_log_buffer_size
            + запас для ОС (мінімум 10-20%)
```

Якщо сервер ще й хостить інші процеси (як у твоєму випадку з HAProxy/Squid поряд) — `innodb_buffer_pool_size` треба рахувати з урахуванням **усієй** машини, не лише MySQL. Профілактика — `MemoryHigh`/`MemoryMax` через systemd drop-in (той самий підхід, що ти вже застосовував для Squid при OOM-інциденті), щоб MySQL не зʼїдав RAM в інших процесів, а отримував контрольовану відмову раніше.

### 13.10 Galera-специфічний траблшутінг

```sql
SHOW STATUS LIKE 'wsrep_%';
```

- **`wsrep_local_state_comment: Donor/Desynced`** — вузол зараз віддає **SST (State Snapshot Transfer)** новому вузлу (повна копія даних) і тимчасово не приймає трафік на повну. Нормально під час додавання нового вузла, проблема — якщо це триває годинами на проді.
- **`wsrep_flow_control_paused` зростає до 1** — кластер штучно гальмує запис, бо один з вузлів не встигає (слабший диск/мережа). Шукати "найслабшу ланку" саме там, а не на тому вузлі, куди прийшов запит.
- **SST провалився** (`rsync`/`xtrabackup`-based) — найчастіше через firewall між вузлами (SST йде по окремому порту, не 3306) або недостатньо місця на диску приймаючого вузла.
- **Split-brain / кластер втратив кворум** (`wsrep_cluster_status: non-Primary`) — кластер сам себе переводить у read-only, щоб не допустити розбіжності даних без кворуму; ручне втручання `SET GLOBAL wsrep_provider_options='pc.bootstrap=true';` потрібне **лише** на ОДНОМУ вузлі, обраному як джерело істини, і тільки після того, як підтверджено, що інші вузли реально мертві (а не просто мережево недосяжні — інакше отримаєш справжній split-brain).

### 13.11 Швидка шпаргалка команд для інциденту (порядок виконання)

```sql
SHOW REPLICA STATUS\G                                  -- 1. реплікація жива?
SHOW FULL PROCESSLIST;                                 -- 2. що зараз виконується
SHOW ENGINE INNODB STATUS\G                             -- 3. транзакції, локи, деадлоки
SELECT * FROM information_schema.innodb_trx;            -- 4. довгі/підозрілі транзакції
SELECT * FROM sys.innodb_lock_waits;                     -- 5. хто кого блокує
SHOW GLOBAL STATUS LIKE 'Threads_%';                      -- 6. навантаження по тредах
SHOW VARIABLES LIKE '%timeout%';                           -- 7. перевірка таймаутів, якщо щось "висить" довго
```

```bash
iostat -x 1 5           # 8. диск — bottleneck чи ні
free -h                  # 9. памʼять — чи близько до OOM
dmesg -T | tail -50      # 10. чи був OOM-кілл нещодавно
journalctl -u mysql -n 200 --no-pager   # 11. хронологія подій на рівні systemd/ОС
```

---

## 14. Чек-лист "що треба знати, щоб впевнено пройти DBA/DevOps інтерв'ю по MySQL"

- [ ] Пояснити різницю async / semi-sync / sync (Galera, Group Replication) і ціну кожного варіанту
- [ ] Намалювати на дошці шлях binlog → IO Thread → relay log → SQL Thread
- [ ] Пояснити, чому MTS (multi-threaded replication) з'явився і яку проблему вирішує
- [ ] GTID vs binlog file:position — переваги при failover
- [ ] STATEMENT vs ROW vs MIXED binlog format — і конкретний приклад, де STATEMENT ламається
- [ ] InnoDB: buffer pool, redo log, undo log, doublewrite — навіщо кожен
- [ ] ACID — як саме кожна буква забезпечується механізмами InnoDB
- [ ] Isolation levels + чому InnoDB REPEATABLE READ "сильніший" за стандарт SQL
- [ ] Row lock vs gap lock vs next-key lock, і що таке deadlock detection
- [ ] Як читати `SHOW REPLICA STATUS` і `SHOW ENGINE INNODB STATUS`
- [ ] Які метрики виносити в Prometheus/Grafana і на що ставити алерти (lag, threads_running, lock waits, hit ratio)
- [ ] PITR через binlog + бекап, правило 3-2-1
- [ ] Коли вибрати ProxySQL, коли Vitess, коли просто orchestrator
- [ ] Базові відмінності MySQL/PostgreSQL/MongoDB/Redis на рівні моделі даних і consistency-гарантій
- [ ] Знати алгоритм дій при інциденті: знімок стану ДО рестарту → логи → SHOW PROCESSLIST/INNODB STATUS → рішення
- [ ] Вміти діагностувати: зламана реплікація (IO vs SQL thread), lag, "база висить", too many connections, диск заповнюється, повторювані деадлоки, краш-рекавері, OOM, Galera split-brain

---

