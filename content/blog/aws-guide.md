+++
title = "AWS для DevOps"
date = 2025-06-16
draft = false
tags = ["aws"]
categories = ["Cloud"]
description = "Детальний розбір AWS-тем: IAM, EC2, VPC, Route 53, S3, RDS, CloudWatch, Terraform — з прикладами, коментарями."
+++

---

## Що таке AWS Nitro System

Перш ніж переходити до сервісів — важливо розуміти фундамент, на якому вони побудовані.

**AWS Nitro System** — це апаратно-програмна платформа, яку Amazon розробила для своїх EC2-інстансів починаючи з 2017 року. Це не просто «гіпервізор» — це ціла екосистема, що складається з кількох компонентів.

### З чого складається Nitro

**Nitro Hypervisor** — мінімалістичний гіпервізор (на базі KVM), що керує розподілом CPU і пам'яті між гостьовими ОС. На відміну від класичного Xen (який використовувався раніше), Nitro Hypervisor майже не має власного коду в privileged-режимі — більшість функцій перенесена в апаратуру.

**Nitro Cards** — спеціалізовані ASICs (мікросхеми), що виконують роботу, яка раніше лягала на CPU хоста:
- **Nitro Card for VPC** — мережевий I/O: шифрування, маршрутизація, Security Groups
- **Nitro Card for EBS** — блоковий I/O: шифрування томів, обробка операцій запису/читання
- **Nitro Card Controller** — керує картами, моніторингом і прошивкою

**Nitro Security Chip** — апаратна верифікація: перевіряє цілісність прошивки та блокує доступ AWS-персоналу до даних гостьової ОС. Це основа гарантії «no AWS employee access».

### Чому це важливо знати

1. **Продуктивність** — оскільки I/O (мережа, диск) обробляється окремими картами, гостьова ОС отримує майже весь ресурс CPU хоста. Через це нові типи інстансів (m6i, c7g, r7iz) значно швидші за старі.

2. **Безпека** — апаратна ізоляція між VM є фізичною, а не лише програмною.

3. **Bare-metal інстанси** — завдяки Nitro, EC2 може запускати ОС безпосередньо на залізі (`.metal`-типи) без гіпервізора між гостьовою системою та апаратурою.

4. **Nitro Enclaves** — ізольовані середовища всередині EC2 для обробки чутливих даних (криптографія, обробка PII) без доступу навіть з основної ОС інстанса.

```
┌─────────────────────────────────────────────┐
│              EC2 Guest OS (твоя VM)          │
├─────────────────────────────────────────────┤
│           Nitro Hypervisor (тонкий шар)      │
├───────────────┬────────────────┬────────────┤
│  Nitro Card   │  Nitro Card   │  Nitro     │
│  (Network)    │  (EBS/NVMe)   │  Security  │
│  VPC routing  │  Block I/O    │  Chip      │
│  SG enforcement│  Encryption  │  Attestation│
└───────────────┴────────────────┴────────────┘
│              Фізичний сервер AWS             │
└─────────────────────────────────────────────┘
```

---

## 1. IAM — Identity and Access Management

> **Найважливіша тема AWS.** На кожній співбесіді буде хоча б одне питання про IAM.

### Базові концепції

**IAM User** — постійна ідентичність: конкретна людина або застосунок з власними обліковими даними (логін/пароль або access key + secret key). Ключі IAM User довгострокові — якщо витечуть, зловмисник матиме доступ доти, доки ти їх не відкличеш.

**IAM Group** — колекція Users для зручного призначення прав. Групи самі по собі не можуть виконувати дії — тільки їх члени. Наприклад: група `Developers` з правами на EC2 і CodeDeploy.

**IAM Role** — ідентичність без власних статичних ключів. Роль «приймається» (assume) сервісом, користувачем або іншим AWS-акаунтом на певний час. Ключі доступу для ролі — **тимчасові** (STS tokens, від 15 хвилин до 12 годин).

**IAM Policy** — JSON-документ, що описує дозволи: які дії (`Action`) на які ресурси (`Resource`) дозволені або заборонені (`Effect: Allow/Deny`).

### User vs Role — ключова різниця

| Параметр | IAM User | IAM Role |
|---|---|---|
| Ключі доступу | Статичні, довгострокові | Тимчасові (STS) |
| Призначення | Людина або застосунок | AWS-сервіс, федерований юзер, крос-акаунт |
| Ротація секретів | Ручна або автоматична | Автоматична (STS) |
| Рекомендація | Мінімізувати | Використовувати для сервісів |

### AssumeRole

`sts:AssumeRole` — API-виклик, що дозволяє одній ідентичності «стати» іншою роллю. Результат — тимчасові `AccessKeyId`, `SecretAccessKey`, `SessionToken`.

```json
// Trust Policy — хто може прийняти цю роль
// Цей документ прикріплюється до самої РОЛІ (не до юзера)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      // Effect: дозволяємо дію
      "Effect": "Allow",
      // Principal: кому дозволяємо — EC2-сервісу AWS
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      // Action: яку саме дію дозволяємо
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Least Privilege Principle

**Принцип найменших привілеїв** — кожна ідентичність повинна мати рівно стільки прав, скільки потрібно для своєї задачі, і нічого більше.

Практичні правила:
- Ніколи не давай `"Action": "*"` або `"Resource": "*"` у production
- Починай з мінімальних прав, додавай за потребою — не навпаки
- Regularly audit: `IAM Access Analyzer` покаже невикористані права
- Для EC2/Lambda — завжди Role, ніколи не вбудовуй access keys у код

### Managed Policies vs Inline Policies

**AWS Managed Policy** — готові політики від Amazon (`AmazonS3ReadOnlyAccess`, `AdministratorAccess`). Версіонуються AWS, не можна змінити.

**Customer Managed Policy** — твоя власна політика, що може бути прикріплена до багатьох ідентичностей. Рекомендований підхід для організацій.

**Inline Policy** — вбудована безпосередньо в User/Group/Role. Існує тільки разом з нею, не перевикористовується. Використовуй лише для специфічних винятків.

```
Customer Managed Policy (окремий ресурс, можна прикріпити до багатьох)
    │
    ├── прикріплена до → Role: app-server-role
    ├── прикріплена до → Role: ci-cd-role
    └── прикріплена до → Group: DevTeam

Inline Policy (існує ТІЛЬКИ всередині конкретної ролі)
    └── вбудована в → Role: legacy-special-role (особливий виняток)
```

### Типове питання: EC2 читає S3 без ключів

**Питання:** *У тебе є EC2, якій треба читати файли з S3. Як зробиш? Чому не використовувати access keys?*

**Відповідь:**

1. Створити IAM Role з Trust Policy для EC2
2. Прикріпити Permission Policy з правами на S3
3. Прив'язати роль до EC2 через **Instance Profile**

```json
// Крок 1: Permission Policy — що роль може робити
// Зберігаємо як CustomerPolicy: ec2-s3-read-policy
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      // Тільки читання — GetObject і ListBucket, нічого зайвого
      "Action": [
        "s3:GetObject",      // завантажити файл
        "s3:ListBucket"      // перерахувати файли в бакеті
      ],
      // Обмежуємо конкретним бакетом — не зірочкою!
      "Resource": [
        "arn:aws:s3:::my-app-bucket",        // для ListBucket — сам бакет
        "arn:aws:s3:::my-app-bucket/*"       // для GetObject — об'єкти всередині
      ]
    }
  ]
}
```

```bash
# Крок 2: Створити роль з AWS CLI
# --assume-role-policy-document — Trust Policy (хто може прийняти роль)
aws iam create-role \
  --role-name ec2-s3-read-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Прикріпити Permission Policy до ролі
aws iam put-role-policy \
  --role-name ec2-s3-read-role \
  --policy-name s3-read-access \
  --policy-document file://ec2-s3-read-policy.json

# Крок 3: Створити Instance Profile (обгортка для ролі, яку EC2 розуміє)
aws iam create-instance-profile \
  --instance-profile-name ec2-s3-read-profile

# Додати роль до Instance Profile
aws iam add-role-to-instance-profile \
  --instance-profile-name ec2-s3-read-profile \
  --role-name ec2-s3-read-role

# Крок 4: Запустити EC2 з цим Instance Profile
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.micro \
  --iam-instance-profile Name=ec2-s3-read-profile
  # EC2 автоматично отримає тимчасові ключі через instance metadata service
  # http://169.254.169.254/latest/meta-data/iam/security-credentials/ec2-s3-read-role
```

**Чому НЕ access keys:** якщо вбудувати `AWS_ACCESS_KEY_ID` у код або `.env` — вони можуть витекти через git, логи, або злом сервера. Instance Profile видає тимчасові ключі (STS), що автоматично ротуються кожні кілька годин.

---

## 2. EC2 — Elastic Compute Cloud

### Що таке EC2

EC2 — це віртуальні сервери в хмарі AWS. Фізично — VM на Nitro-гіпервізорі, але для тебе це «комп'ютер за хвилину». Ти обираєш тип, AMI, мережу, зберігаєш дані на EBS — і платиш тільки за час роботи.

### Типи інстансів

Назви інстансів читаються за шаблоном: `сімейство + покоління + розмір`. Наприклад, `m7g.xlarge` = сімейство M (General Purpose), 7-е покоління, Graviton-процесор (g), розмір xlarge.

**General Purpose (m, t)** — баланс CPU/RAM/мережа. `t3`, `t4g` — burstable (дешевші, з CPU Credits). `m6i`, `m7g` — стабільний performance. Для більшості веб-застосунків.

**Compute Optimized (c)** — більше CPU на рубль пам'яті. `c7g`, `c6i`. Для batch-обробки, HPC, ігрових серверів, ML inference.

**Memory Optimized (r, x)** — великий RAM. `r7g` (до 768 GB), `x2gd` (до 3.8 TB). Для in-memory БД (Redis, SAP HANA), великих кешів.

**Storage Optimized (i, d, h)** — NVMe SSD або HDD з великою IOPS-пропускністю. `i4i` — NVMe, до 7.5M IOPS. Для NoSQL, OLAP, data warehousing.

**Accelerated Computing (p, g, inf, trn)** — GPU або спеціалізовані чіпи. `p4d` (NVIDIA A100), `inf2` (AWS Inferentia). Для ML training/inference.

### AMI — Amazon Machine Image

**AMI** — шаблон для запуску EC2: містить ОС, пре-встановлене ПЗ, конфігурацію. По суті — snapshot диска + метадані.

```bash
# Знайти актуальний Ubuntu 24.04 AMI у своєму регіоні
aws ec2 describe-images \
  --owners 099720109477 \         # Canonical (Ubuntu) — офіційний publisher
  --filters \
    "Name=name,Values=ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*" \
    "Name=state,Values=available" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text
  # sort_by + [-1] — беремо найновіший образ

# Створити власний AMI з існуючого EC2 (golden image)
aws ec2 create-image \
  --instance-id i-0abc123def456789 \
  --name "my-app-v1.2.0-$(date +%Y%m%d)" \  # ім'я з датою для версіонування
  --no-reboot                                  # не перезавантажувати інстанс
```

### EBS vs Instance Store

**EBS (Elastic Block Store)** — мережевий блоковий диск. Персистентний: дані зберігаються після зупинки або завершення інстанса (якщо не позначений DeleteOnTermination). Можна відчепити і приєднати до іншого EC2.

**Instance Store** — фізичний NVMe на сервері, де запущений твій EC2. Дуже швидкий (мільйони IOPS), але **ефемерний**: дані зникають при зупинці, рестарті, або переїзді на інший хост. Лише для тимчасових даних: кеш, буфери, scratch-диски.

```
EBS:
  EC2 Instance ──(мережа)── EBS Volume (окремий сервер зберігання)
  Тип: gp3 (загального призначення), io2 (IOPS-інтенсивний), st1 (throughput, HDD)
  Можна: snapshot → AMI → відновлення

Instance Store:
  EC2 Instance ──(PCIe/NVMe безпосередньо)── Локальний NVMe
  ЗНИКАЄ при: stop, terminate, host failure
  НЕ ЗНИКАЄ при: reboot (тільки reboot!)
```

### Security Groups

**Security Group** — statefull файрвол для EC2 (або будь-якого ENI). «Stateful» означає: якщо ти дозволив вихідний запит, відповідь на нього буде автоматично дозволена, навіть якщо немає явного inbound-правила.

За замовчуванням: весь inbound трафік заборонений, весь outbound дозволений.

```bash
# Створити Security Group для web-сервера
aws ec2 create-security-group \
  --group-name web-server-sg \
  --description "Web server: HTTP, HTTPS, SSH" \
  --vpc-id vpc-0abc123

# Дозволити HTTPS з будь-якого IP (публічний сайт)
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc123 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0     # IPv4 весь світ

# Дозволити HTTP (редирект на HTTPS)
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc123 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# SSH тільки з твого офісного IP (НЕ відкривай SSH на весь світ!)
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc123 \
  --protocol tcp \
  --port 22 \
  --cidr 203.0.113.10/32   # /32 — конкретна адреса

# Дозволити доступ від іншого Security Group (паттерн: LB → App)
# source-group замість cidr — рекомендовано замість IP-діапазонів всередині VPC
aws ec2 authorize-security-group-ingress \
  --group-id sg-app-servers \
  --protocol tcp \
  --port 8080 \
  --source-group sg-load-balancer   # тільки від Load Balancer SG
```

### User Data (Bootstrap-скрипт)

**User Data** — скрипт або cloud-init конфігурація, що виконується одноразово при першому старті EC2. Використовується для автоматизації початкового налаштування (встановлення пакетів, запуск сервісів, реєстрація в Consul).

```bash
#!/bin/bash
# Цей скрипт виконується від root при першому запуску EC2
# Логи: /var/log/cloud-init-output.log

# Оновлюємо індекс пакетів (завжди першим кроком)
apt-get update -y

# Встановлюємо nginx
apt-get install -y nginx

# Додаємо просту сторінку, що показує hostname інстанса
# TOKEN — отримуємо IMDSv2 токен для безпечного доступу до metadata
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# Отримуємо instance-id через Instance Metadata Service
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)

# Записуємо HTML-сторінку
cat > /var/www/html/index.html << EOF
<h1>Привіт з EC2!</h1>
<p>Instance ID: ${INSTANCE_ID}</p>
EOF

# Вмикаємо nginx і додаємо в автозапуск
systemctl enable --now nginx
```

### Auto Scaling Group + Load Balancer

**Auto Scaling Group (ASG)** — автоматично збільшує або зменшує кількість EC2 залежно від навантаження (CPU, SQS-черга, кастомна метрика CloudWatch). Гарантує мінімальну кількість здорових інстансів.

**Load Balancer** — розподіляє трафік між EC2 в ASG, виконує health checks і автоматично виключає нездорові інстанси.

```
                    ┌─────────────────────────────┐
Internet ──────────▶│   Application Load Balancer  │
                    └──────┬───────────┬───────────┘
                           │           │
                    ┌──────▼──┐   ┌────▼─────┐
                    │  EC2-1  │   │  EC2-2   │   ← Auto Scaling Group
                    │ (nginx) │   │ (nginx)  │     мін: 2, макс: 10
                    └─────────┘   └──────────┘
                         ↑   Auto Scaling додає EC2-3 при CPU > 70%
```

**Питання:** *Твій сервер впав. Як забезпечити відновлення?*

Відповідь — не один сервер, а ASG за ALB:
- ASG замінить впалий інстанс протягом 1-2 хвилин (health check fail → terminate → launch new)
- ALB перестає надсилати трафік на нездоровий інстанс одразу після health check failure
- Multi-AZ: EC2 в різних Availability Zones — захист від збою одного датацентру

---

## 3. VPC — Virtual Private Cloud

> **Одна з найважчих тем.** Розуміння VPC — це розуміння мережі всього AWS.

### Що таке VPC

**VPC** — твоя приватна логічна мережа всередині AWS. Ізольована від інших акаунтів і VPC. В одному регіоні може бути до 5 VPC (ліміт підвищується за запитом).

### CIDR блоки

**CIDR (Classless Inter-Domain Routing)** — спосіб позначити діапазон IP-адрес. `10.0.0.0/16` означає: фіксовані перші 16 бітів (`10.0`), решта 16 бітів — вільні → 65,536 адрес.

```
VPC CIDR: 10.0.0.0/16  (10.0.0.0 — 10.0.255.255, 65534 хостів)
│
├── Public Subnet AZ-a:  10.0.1.0/24   (254 хости)
├── Public Subnet AZ-b:  10.0.2.0/24   (254 хости)
├── Private Subnet AZ-a: 10.0.11.0/24  (254 хости)
└── Private Subnet AZ-b: 10.0.12.0/24  (254 хости)

# AWS резервує 5 адрес у кожній підмережі:
# .0 — network address
# .1 — VPC router
# .2 — DNS resolver
# .3 — зарезервовано
# .255 — broadcast (не використовується, але резервується)
# Тобто /24 дає 256 - 5 = 251 доступну адресу
```

### Public vs Private Subnet

**Public Subnet** — підмережа з маршрутом до Internet Gateway. EC2 у public subnet може отримати Public IP або Elastic IP і спілкуватися з інтернетом напряму.

**Private Subnet** — без прямого маршруту до IGW. EC2 у private subnet не доступна з інтернету. Для вихідного трафіку (apt-get, API calls) використовує NAT Gateway.

### Компоненти VPC — розбір

**Internet Gateway (IGW)** — ворота між VPC і публічним інтернетом. Один на VPC. Stateful (як Security Group). Без IGW — немає інтернету ні в якому напрямку.

**NAT Gateway** — дозволяє EC2 у private subnet ініціювати з'єднання назовні, але блокує вхідні з'єднання ззовні. Розміщується у public subnet. Managed-сервіс (AWS керує ним, не ти). Платний: ~$0.045/годину + $0.045 за GB трафіку.

**Route Table** — таблиця маршрутизації. Кожна підмережа асоційована з одною Route Table. Правила: `destination → target`.

**Network ACL (NACL)** — stateless файрвол на рівні підмережі. «Stateless» означає: потрібно явно дозволяти і вхідний, і вихідний трафік (на відміну від Security Group). Правила обробляються за порядком номерів (100, 200, ...). Перше правило, що спрацювало — фінальне.

```
Різниця Security Group vs NACL:

Security Group (SG):          Network ACL (NACL):
- Рівень: інстанс/ENI         - Рівень: підмережа
- Stateful                    - Stateless
- Тільки Allow                - Allow і Deny
- Всі правила перевіряються   - Правила за порядком номерів
- Захищає конкретний EC2      - Захищає всю підмережу
```

### Повна схема VPC з поясненнями

```
Internet
    │
    ▼
Internet Gateway ──────────────────────────── прикріплений до VPC
    │
    ▼
┌─────────────────── Public Subnet (10.0.1.0/24) ──────────────┐
│  Route Table:                                                  │
│  10.0.0.0/16 → local     (внутрішньо VPC — через router)      │
│  0.0.0.0/0   → igw-xxx   (весь інший трафік → IGW)            │
│                                                                │
│  ┌─────────────────┐    ┌────────────────────┐                │
│  │  NAT Gateway    │    │  Bastion Host EC2  │                │
│  │  (Elastic IP)   │    │  (SSH jump server) │                │
│  └────────┬────────┘    └────────────────────┘                │
└───────────│────────────────────────────────────────────────────┘
            │
            ▼ (вихідний трафік від private EC2 → NAT → IGW → Internet)
┌─────────────────── Private Subnet (10.0.11.0/24) ────────────┐
│  Route Table:                                                  │
│  10.0.0.0/16 → local     (VPC-internal)                       │
│  0.0.0.0/0   → nat-xxx   (вихідний інтернет через NAT GW)     │
│                                                                │
│  ┌──────────────┐    ┌─────────────────────────────────┐      │
│  │  App EC2     │    │  RDS (тільки internal доступ)   │      │
│  │  (no pub IP) │    │  порт 5432, тільки від App EC2  │      │
│  └──────────────┘    └─────────────────────────────────┘      │
└────────────────────────────────────────────────────────────────┘
```

**Питання:** *Чому EC2 у private subnet не має інтернету?*

Відповідь: Бо в Route Table для private subnet немає маршруту `0.0.0.0/0 → igw`. Є тільки `0.0.0.0/0 → nat-gateway-id`. Якщо NAT Gateway видалений або маршрут відсутній — трафік нікуди не йде (дропається на рівні роутера VPC).

```bash
# Перевірити Route Table приватної підмережі
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=subnet-0abc123private" \
  --query 'RouteTables[].Routes[]'

# Очікуваний вивід для private subnet:
# [
#   {"DestinationCidrBlock": "10.0.0.0/16", "GatewayId": "local"},
#   {"DestinationCidrBlock": "0.0.0.0/0",   "NatGatewayId": "nat-0xyz789"}
# ]
```

### VPC Peering і Transit Gateway

**VPC Peering** — пряме з'єднання між двома VPC (в одному або різних акаунтах/регіонах). Не транзитивне: якщо A ↔ B і B ↔ C, то A не бачить C.

**Transit Gateway (TGW)** — хаб для з'єднання багатьох VPC і on-premises мереж. Підтримує транзитивний роутинг. Для великих організацій з >5 VPC.

---

## 4. Route 53 — DNS сервіс

### Як працює DNS

DNS (Domain Name System) — ієрархічна розподілена система перетворення доменних імен у IP-адреси.

```
Клієнт запитує: www.example.com

1. Запит іде до Recursive Resolver (провайдер або 8.8.8.8)
2. Resolver питає Root Nameserver: "хто відповідає за .com?"
3. Root повертає TLD Nameserver для .com
4. Resolver питає TLD NS: "хто відповідає за example.com?"
5. TLD повертає Authoritative NS (Route 53 для твого домену)
6. Route 53 відповідає: "www.example.com = 1.2.3.4"
7. Resolver кешує відповідь на TTL секунд і повертає клієнту
```

### Типи DNS-записів

**A** — ім'я → IPv4 адреса. `www.example.com → 1.2.3.4`

**AAAA** — ім'я → IPv6 адреса. `www.example.com → 2001:db8::1`

**CNAME** — аліас: одне ім'я → інше ім'я. `blog.example.com → example.com`. Не можна на apex домен (`example.com` без www).

**Alias** (тільки Route 53) — як CNAME, але для apex домену та AWS-ресурсів (ALB, CloudFront, S3). Безкоштовні запити, автоматично оновлюються при зміні IP ALB.

**MX** — Mail Exchange: куди доставляти email для домену.

**TXT** — довільний текст: SPF, DKIM, верифікація домену (Google, AWS ACM).

**TTL** — Time To Live: скільки секунд клієнт кешує відповідь. Низький TTL (60с) — швидке перемикання при failover. Високий TTL (86400с) — менше навантаження на DNS, але повільні зміни.

### Routing Policies

**Simple** — один запис, одна або кілька IP (Random order). Немає health checks.

**Weighted** — розподіл трафіку у відсотках між кількома endpoint. `Canary deployment: 90% → старий ALB, 10% → новий ALB`.

**Latency** — відправляє запит до регіону з найменшою затримкою для клієнта (AWS вимірює латентність між регіонами).

**Failover** — Primary + Secondary. Route 53 перемикає на Secondary якщо Primary health check провалюється.

**Geolocation** — відповідь залежить від географічного розташування клієнта. `Клієнти з UA → eu-west-1, з US → us-east-1`.

**Geoproximity** (Traffic Flow) — як Geolocation, але з можливістю «зміщувати» кордони (bias).

**IP-based** — маршрутизація за CIDR-діапазоном IP клієнта.

```
# Питання: Як зробити сайт доступним навіть при падінні регіону?

Route 53 Failover:
  ┌─────────────────────────────────────┐
  │          Route 53                   │
  │  example.com → Failover Routing     │
  │                                     │
  │  PRIMARY:   ALB us-east-1 ←────── Health Check (30s interval)
  │  SECONDARY: ALB eu-west-1 ←────── Health Check
  └─────────────────────────────────────┘

При падінні us-east-1:
  Health check fails → Route 53 автоматично переключає DNS на eu-west-1
  Час переключення: TTL (наприклад 60с) + health check fail threshold
```

### Hosted Zones

**Public Hosted Zone** — для публічних доменів в інтернеті. `example.com` → відповідає на запити з будь-якої точки інтернету.

**Private Hosted Zone** — лише для DNS-резолюції всередині VPC. `internal.company.local` → доступний тільки в твоєму VPC.

---

## 5. Load Balancers

### Три типи Load Balancers в AWS

**Application Load Balancer (ALB)** — Layer 7 (HTTP/HTTPS). Розуміє контент запиту: URL-шлях, заголовки, cookies. Path-based routing: `/api/*` → один Target Group, `/static/*` → інший. Host-based routing: `api.example.com` і `app.example.com` за одним ALB.

**Network Load Balancer (NLB)** — Layer 4 (TCP/UDP/TLS). Не розуміє HTTP, але надзвичайно швидкий: мільйони з'єднань за секунду, затримка ~100 мікросекунд. Зберігає source IP клієнта (ALB підмінює). Для real-time систем, ігор, VoIP.

**Gateway Load Balancer (GWLB)** — Layer 3. Для прозорого проксіювання трафіку через мережеві appliance (Palo Alto, Fortinet, Suricata IDS). Трафік проходить через firewall перш ніж потрапити до EC2.

```
ALB:
  Client → ALB (парсить HTTP) → Target Group A (path: /api)
                              → Target Group B (path: /static)

NLB:
  Client → NLB (TCP passthrough) → EC2 (source IP збережено!)

GWLB:
  Client → GWLB → Firewall EC2 (інспекція) → GWLB → App EC2
```

### Target Groups

**Target Group** — набір бекенд-цілей для балансувальника: EC2-інстанси, IP-адреси, Lambda-функції або інший ALB. Health check визначає, які таргети «здорові».

```bash
# Створити Target Group для ALB
aws elbv2 create-target-group \
  --name app-servers-tg \
  --protocol HTTP \       # протокол між ALB і EC2
  --port 8080 \           # порт застосунку на EC2
  --vpc-id vpc-0abc123 \
  --health-check-path /health \           # endpoint для перевірки
  --health-check-interval-seconds 30 \    # перевіряти кожні 30с
  --healthy-threshold-count 2 \           # 2 успішні → вважати здоровим
  --unhealthy-threshold-count 3           # 3 невдалі → виключити з ротації
```

---

## 6. S3 — Simple Storage Service

### Основи

**S3** — object storage: зберігає файли (objects) у бакетах (buckets). Не файлова система, не блоковий диск — key-value store, де ключ — шлях до файлу, значення — бінарні дані. Нема папок (лише префікси), нема inode, нема append.

Максимальний розмір об'єкта: 5 TB. Завантаження файлів >100 MB — обов'язково Multipart Upload.

### Версіонування

При включеному Versioning S3 зберігає всі версії об'єкта (навіть після перезапису або «видалення»). Delete marker замість справжнього видалення.

```bash
# Включити версіонування на бакеті
aws s3api put-bucket-versioning \
  --bucket my-important-bucket \
  --versioning-configuration Status=Enabled

# Список версій об'єкта
aws s3api list-object-versions \
  --bucket my-important-bucket \
  --prefix configs/app.yaml

# Відновити попередню версію (завантажити конкретну)
aws s3api get-object \
  --bucket my-important-bucket \
  --key configs/app.yaml \
  --version-id "xyz123abc" \     # VersionId з list-object-versions
  restored-app.yaml
```

### Lifecycle Policies

Автоматичне переміщення об'єктів між storage class або видалення.

```json
{
  "Rules": [
    {
      "ID": "move-old-logs-to-glacier",
      "Status": "Enabled",
      "Filter": {"Prefix": "logs/"},   // застосовується до об'єктів з префіксом logs/
      "Transitions": [
        {
          "Days": 30,                   // через 30 днів після створення
          "StorageClass": "STANDARD_IA" // переміщуємо до Infrequent Access (дешевше ~58%)
        },
        {
          "Days": 90,                   // через 90 днів
          "StorageClass": "GLACIER_IR"  // Glacier Instant Retrieval (ще дешевше)
        }
      ],
      "Expiration": {
        "Days": 365                     // видалити через рік
      },
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 30            // видалити старі версії через 30 днів
      }
    }
  ]
}
```

### Storage Classes

| Клас | Призначення | Ціна (відносно) |
|---|---|---|
| STANDARD | Активні дані | 💰💰💰 |
| STANDARD_IA | Рідко читаються | 💰💰 |
| GLACIER_IR | Архів, миттєвий доступ | 💰 |
| GLACIER | Архів, доступ 1-5 хв | 💰 (дуже) |
| DEEP_ARCHIVE | Архів, доступ 12 год | 💰 (найдешевший) |
| INTELLIGENT_TIERING | Авто-переміщення | 💰💰 + моніторинг fee |

### Шифрування

**SSE-S3** — AWS керує ключами. Прозоро для тебе. Безкоштовно. Ключ зберігається в AWS, ротується автоматично. Для більшості випадків — достатньо.

**SSE-KMS** — ключ у AWS KMS (Key Management Service). Ти контролюєш ключ: можеш відкликати, ротувати, аудитувати через CloudTrail (хто коли використовував ключ). Для compliance (SOC2, PCI DSS).

**SSE-C** — ти надаєш ключ при кожному запиті. AWS шифрує, але ключ не зберігає. Якщо втратиш ключ — дані недоступні назавжди.

**Client-side** — шифрування до завантаження в S3. AWS не бачить відкриті дані.

### Bucket Policy

JSON-документ на рівні бакету: контролює доступ незалежно від IAM. Може дозволяти або забороняти доступ конкретним IAM-ідентичностям, іншим акаунтам, або IP-адресам.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      // Дозволити публічне читання для статичного сайту
      "Sid": "PublicReadForStaticWebsite",
      "Effect": "Allow",
      "Principal": "*",           // будь-хто (публічний доступ)
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-static-site/*"
    },
    {
      // Заблокувати завантаження без шифрування (enforce encryption)
      "Sid": "DenyUnencryptedUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-secure-bucket/*",
      "Condition": {
        // Якщо заголовок x-amz-server-side-encryption відсутній або не AES256
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "AES256"
        }
      }
    }
  ]
}
```

### Presigned URLs

Тимчасове посилання для доступу до приватного об'єкта S3 без AWS-ключів. Корисно для: дозволу завантаження файлів безпосередньо в S3 з браузера, тимчасового шерингу файлів.

```python
import boto3
from datetime import datetime

s3 = boto3.client('s3', region_name='eu-central-1')

# Генерація Presigned URL для завантаження файлу (GET)
# Посилання буде дійсне 3600 секунд (1 година)
url = s3.generate_presigned_url(
    ClientMethod='get_object',       # операція: завантажити
    Params={
        'Bucket': 'my-private-bucket',
        'Key': 'reports/q3-2024.pdf'
    },
    ExpiresIn=3600                   # TTL в секундах
)
# Повертає щось на кшталт:
# https://my-private-bucket.s3.amazonaws.com/reports/q3-2024.pdf?X-Amz-Signature=...&X-Amz-Expires=3600

# Presigned URL для завантаження файлу ВІД клієнта (PUT)
# Браузер може завантажити файл напряму в S3, не проходячи через твій бекенд
upload_url = s3.generate_presigned_url(
    ClientMethod='put_object',
    Params={
        'Bucket': 'my-private-bucket',
        'Key': 'uploads/user-photo.jpg',
        'ContentType': 'image/jpeg'   # клієнт повинен надіслати цей Content-Type
    },
    ExpiresIn=300   # 5 хвилин — достатньо для завантаження
)
```

### Cross-Region Replication

Автоматична реплікація об'єктів між бакетами в різних регіонах. Вимагає увімкнення Versioning на обох бакетах.

```bash
# Увімкнути реплікацію з eu-central-1 до us-east-1
aws s3api put-bucket-replication \
  --bucket source-bucket-eu \
  --replication-configuration '{
    "Role": "arn:aws:iam::123456789:role/s3-replication-role",
    "Rules": [{
      "Status": "Enabled",
      "Filter": {"Prefix": ""},   // "" = реплікувати все
      "Destination": {
        "Bucket": "arn:aws:s3:::destination-bucket-us",
        "StorageClass": "STANDARD_IA"  // у destination зберігаємо дешевше
      }
    }]
  }'
```

---

## 7. RDS — Relational Database Service

### Multi-AZ Deployment

**Multi-AZ** — синхронна реплікація між двома Availability Zones. Standby-інстанс — не читається (не приймає запити). При збої Primary → AWS автоматично перемикає на Standby через DNS (CNAME зміна). Час failover: 60-120 секунд.

Призначення: **висока доступність** (HA), не масштабування читань.

### Read Replica

**Read Replica** — асинхронна репліка для розвантаження читань. Може бути в іншому регіоні (cross-region read replica). На відміну від Multi-AZ Standby — читається. Можна промотувати в окремий standalone DB (ручний failover при потребі).

```
Production Traffic:
  App → RDS Primary (reads + writes)
        │
        └─ синхронна реплікація → RDS Standby AZ-b (Multi-AZ, не читається)

Analytics/Reporting:
  BI Tool → Read Replica (тільки читання, не навантажує Primary)
```

### Backup та Snapshot

**Automated Backup** — щоденний snapshot + transaction logs (point-in-time recovery до 5 хвилин). Retention: 1-35 днів. Видаляються разом з інстансом (якщо не збережено).

**Manual Snapshot** — зберігається до явного видалення. Для: перед міграцією, довгострокового архіву, клонування середовища.

```bash
# Створити manual snapshot перед важливою міграцією
aws rds create-db-snapshot \
  --db-instance-identifier prod-postgres-01 \
  --db-snapshot-identifier prod-postgres-01-before-migration-$(date +%Y%m%d)

# Point-in-time restore — відновити стан БД на конкретний момент
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier prod-postgres-01 \
  --target-db-instance-identifier prod-postgres-01-restored \
  --restore-time 2024-11-15T14:30:00Z   # ISO 8601 timestamp
```

### RDS Proxy

**RDS Proxy** — пулінг з'єднань між застосунком і RDS. Замість 1000 Lambda-функцій що відкривають 1000 з'єднань до PostgreSQL → усі через Proxy (pool). Автоматично переключається при failover без зміни connection string у застосунку.

---

## 8. CloudWatch — Моніторинг

### Metrics

**Metric** — часова серія числових даних. Namespace + Metric Name + Dimensions = унікальна метрика. EC2 надсилає базові метрики кожні 5 хвилин безкоштовно. **Detailed Monitoring** — кожну хвилину (платно).

```bash
# Отримати середній CPU за останні 10 хвилин для конкретного EC2
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \                          # namespace: AWS-сервіс
  --metric-name CPUUtilization \                 # назва метрики
  --dimensions Name=InstanceId,Value=i-0abc123 \ # фільтр за інстансом
  --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 \       # агрегація за 300 секунд (5 хвилин)
  --statistics Average # Average, Sum, Min, Max, SampleCount
```

**Custom Metrics** — надсилай свої метрики (RAM, disk usage, кількість черги).

```bash
# EC2 не надсилає RAM за замовчуванням — потрібен CloudWatch Agent або PutMetricData
aws cloudwatch put-metric-data \
  --namespace "MyApp/Production" \
  --metric-data '[{
    "MetricName": "QueueDepth",
    "Value": 42,
    "Unit": "Count",
    "Dimensions": [{"Name": "Environment", "Value": "prod"}]
  }]'
```

### Alarms

**Alarm** — реакція на порогове значення метрики. Стани: `OK`, `ALARM`, `INSUFFICIENT_DATA`.

```bash
# Алерт якщо CPU > 80% протягом 5 хвилин
aws cloudwatch put-metric-alarm \
  --alarm-name "prod-ec2-high-cpu" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --dimensions Name=InstanceId,Value=i-0abc123 \
  --period 300 \           # вікно 5 хвилин
  --evaluation-periods 2 \ # 2 послідовні порушення → ALARM
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:eu-central-1:123456789:ops-alerts   # SNS topic → email/PagerDuty
```

### CloudTrail

**CloudTrail** — аудит-лог всіх API-викликів в AWS: хто, що, коли, звідки. Management Events (безкоштовно за 90 днів). Data Events (S3 GetObject/PutObject, Lambda Invoke) — платні.

```bash
# Хто видалив Security Group sg-0abc123 і коли?
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=sg-0abc123 \
  --query 'Events[?EventName==`DeleteSecurityGroup`].[EventTime,Username,SourceIPAddress]' \
  --output table
```

### CloudWatch Logs Insights

```sql
-- Знайти всі 5xx помилки в логах ALB за останню годину
fields @timestamp, @message
| filter @message like /HTTP\/1.1" 5/
| stats count(*) as error_count by bin(5m)
| sort @timestamp desc
| limit 100
```

---

## 9. Мережі — базові концепції

> AWS — це мережа. Без розуміння TCP/IP, DNS, TLS — неможливо дебажити інфраструктурні проблеми.

### TCP/IP

**TCP (Transmission Control Protocol)** — надійний транспорт з підтвердженням доставки. Трирукостискання (SYN → SYN-ACK → ACK) перед передачею даних. HTTP, HTTPS, SSH, PostgreSQL — використовують TCP.

**UDP** — ненадійний, але швидкий. Без підтвердження. DNS, VoIP, відеострімінг.

```
TCP Handshake:
  Client ──── SYN ────────────────────▶ Server   (я хочу з'єднатися)
  Client ◀─── SYN-ACK ──────────────── Server   (добре, готовий)
  Client ──── ACK ────────────────────▶ Server   (відмінно, починаємо)
  ──── дані передаються в обох напрямках ────
  Client ──── FIN ────────────────────▶ Server   (закінчую)
  Client ◀─── FIN-ACK ──────────────── Server   (прийнято)
```

### TLS/HTTPS

**TLS (Transport Layer Security)** — шифрування поверх TCP. HTTPS = HTTP + TLS. При TLS Handshake:
1. Client Hello: які cipher suites підтримує клієнт, random bytes
2. Server Hello: вибраний cipher suite, сертифікат (з Public Key)
3. Клієнт перевіряє сертифікат (ланцюг до довіреного CA)
4. Key exchange (ECDHE): обидві сторони обчислюють спільний секрет
5. Шифроване з'єднання встановлено

**Certificate Manager (ACM)** в AWS — безкоштовні SSL/TLS сертифікати для ALB, CloudFront, API Gateway. Автоматична ротація.

### NAT — Network Address Translation

**NAT** — перетворення приватних IP на публічні при виході в інтернет. Це те, що робить NAT Gateway в AWS.

```
Private EC2 (10.0.11.5) надсилає пакет до 8.8.8.8:
  1. EC2: src=10.0.11.5:54321, dst=8.8.8.8:53
  2. NAT GW: src=52.29.100.1:12345, dst=8.8.8.8:53  ← підмінює src IP
  3. DNS відповідає на 52.29.100.1:12345
  4. NAT GW знаходить сесію і повертає 10.0.11.5:54321
  5. EC2 отримує відповідь
```

### VPN та AWS

**Site-to-Site VPN** — зашифрований тунель між твоїм on-premises роутером (або Cisco ASA, pfSense) і Virtual Private Gateway в AWS VPC. IPsec/IKEv2. Два тунелі для redundancy.

**Client VPN** — OpenVPN-сумісний managed сервіс. Дозволяє розробникам підключатися до VPC з ноутбука.

---

## 10. Infrastructure as Code

### Terraform — основи

**Terraform** — декларативний IaC-інструмент від HashiCorp. Ти описуєш бажаний стан інфраструктури в `.tf` файлах, Terraform приводить реальний стан до бажаного.

**State** — файл `terraform.tfstate`: відображення між твоїм кодом і реальними ресурсами AWS. **Критично важливий**: без нього Terraform не знає, що вже створено. Ніколи не редагуй вручну.

**Remote Backend** — зберігання state в S3 + DynamoDB (для lock). Обов'язково в команді.

```hcl
# backend.tf — Remote state в S3
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"  # бакет для state файлів
    key            = "prod/vpc/terraform.tfstate"  # шлях всередині бакету
    region         = "eu-central-1"
    encrypt        = true                           # шифрувати state (SSE-S3)
    dynamodb_table = "terraform-state-lock"         # таблиця для lock (запобігає race condition)
  }

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"   # ~> 5.0 означає >= 5.0, < 6.0
    }
  }
}

# provider.tf — налаштування AWS провайдера
provider "aws" {
  region = var.aws_region

  # Теги, що автоматично додаються до всіх ресурсів
  default_tags {
    tags = {
      Environment = var.environment
      ManagedBy   = "terraform"
      Project     = "my-app"
    }
  }
}
```

### Модулі

**Module** — перевикористовувана група ресурсів. `module "vpc"` викликає папку з tf-файлами, передає змінні, отримує outputs.

```hcl
# modules/ec2-instance/main.tf — модуль для EC2
variable "instance_type" {
  description = "Тип EC2 інстанса"
  type        = string
  default     = "t3.micro"
}

variable "subnet_id" {
  description = "ID підмережі для запуску"
  type        = string
}

variable "security_group_ids" {
  description = "Список Security Group IDs"
  type        = list(string)
}

# Ресурс EC2 інстанса
resource "aws_instance" "this" {
  ami           = data.aws_ami.ubuntu.id  # AMI з data source (завжди актуальний)
  instance_type = var.instance_type
  subnet_id     = var.subnet_id

  vpc_security_group_ids = var.security_group_ids

  # IAM Instance Profile для доступу до AWS API без ключів
  iam_instance_profile = aws_iam_instance_profile.this.name

  # Метадані — вимагаємо IMDSv2 (безпечніший)
  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"   # IMDSv2: токен обов'язковий
    http_put_response_hop_limit = 1            # блокує SSRF через metadata
  }

  # EBS root volume
  root_block_device {
    volume_type           = "gp3"
    volume_size           = 20      # GB
    encrypted             = true    # шифрувати диск
    delete_on_termination = true    # видалити з EC2 при terminate
  }

  tags = {
    Name = "${var.environment}-${var.app_name}"
  }
}

# Output — передаємо значення назад до calling module
output "instance_id" {
  value       = aws_instance.this.id
  description = "ID створеного EC2 інстанса"
}

output "private_ip" {
  value       = aws_instance.this.private_ip
  description = "Приватна IP-адреса інстанса"
}
```

```hcl
# main.tf — використання модуля
module "app_server" {
  source = "./modules/ec2-instance"    # відносний шлях до модуля

  instance_type      = "t3.small"
  subnet_id          = module.vpc.private_subnet_ids[0]
  security_group_ids = [aws_security_group.app.id]
  environment        = "prod"
  app_name           = "api-server"
}

# Доступ до output модуля
output "app_server_ip" {
  value = module.app_server.private_ip
}
```

### Drift Detection

**Drift** — розбіжність між state Terraform і реальним станом AWS (хтось змінив ресурс вручну через консоль).

```bash
# Перевірити drift — порівняти state з реальністю
terraform plan
# Якщо є drift — plan покаже зміни, які Terraform хоче зробити

# Оновити state щоб відобразити ручні зміни (прийняти їх)
terraform refresh
# ОБЕРЕЖНО: потім plan може показати інші зміни

# Або імпортувати існуючий ресурс у state
terraform import aws_instance.web i-0abc123def456789
```

### CloudFormation — альтернатива

**CloudFormation** — AWS-рідний IaC. YAML або JSON. Ресурси організовані в **Stack**. **Change Set** — preview змін перед застосуванням (аналог `terraform plan`).

```yaml
# Простий CloudFormation шаблон — EC2 з Security Group
AWSTemplateFormatVersion: '2010-09-09'
Description: Web server stack

Parameters:
  # Параметр дозволяє перевикористати шаблон для різних середовищ
  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues: [t3.micro, t3.small, t3.medium]
    Description: EC2 instance type

Resources:
  # Security Group для web-сервера
  WebServerSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow HTTP and HTTPS
      VpcId: !Ref VpcId                    # !Ref — посилання на параметр або ресурс
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0

  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType      # !Ref — значення параметру InstanceType
      ImageId: ami-0abcdef1234567890
      SecurityGroupIds:
        - !GetAtt WebServerSG.GroupId      # !GetAtt — атрибут іншого ресурсу
      UserData:
        # Fn::Base64 кодує скрипт у base64 (EC2 вимагає base64 для UserData)
        Fn::Base64: |
          #!/bin/bash
          apt-get update -y
          apt-get install -y nginx
          systemctl enable --now nginx

Outputs:
  # CloudFormation виводить значення після деплою
  PublicIP:
    Description: Public IP of web server
    Value: !GetAtt WebServer.PublicIp
```

---

## 11. AWS Leadership Principles — для інтерв'ю в Amazon

Якщо проходиш саме в Amazon (AWS), окрім техніки обов'язково будуть поведінкові питання за **Leadership Principles**.

### Метод STAR

**Situation** — контекст: де працював, яка система, яка команда

**Task** — твоя відповідальність у цій ситуації

**Action** — конкретні кроки, які TИ зробив (не «ми», не «команда»)

**Result** — вимірюваний результат (час, гроші, відсотки, інциденти)

### Приклад відповіді

**Питання:** *Розкажи про ситуацію, коли сервер впав у продакшені.*

**Відповідь:**

**S:** Під час пікового навантаження один з HAProxy-серверів у нашому пулі з 500 bare-metal проксі-машин перестав обробляти трафік.

**T:** Як DevOps-інженер я відповідав за моніторинг і відновлення сервісу. SLA — менше 5 хвилин на реакцію.

**A:**
1. Отримав алерт від Prometheus/Grafana (налаштований алерт на `haproxy_frontend_sessions_total` = 0)
2. За 30 секунд підключився через Headscale VPN і запустив `journalctl -u haproxy -n 100`
3. Виявив: конфліктний IP через помилку в Ansible-плейбуці (duplicate frontend bind)
4. Відкотив проблемний конфіг через `ansible-playbook rollback.yml --limit srv-nj-01`
5. HAProxy перезапустився за 10 секунд, трафік відновлено

**R:** Загальний downtime — 3 хвилини 47 секунд. Після інциденту написав post-mortem і додав pre-flight validation в Ansible: перевірка конфіглів через `haproxy -c -f /etc/haproxy/haproxy.cfg` до деплою.

---

## Швидка шпаргалка для співбесіди

```
IAM:
  Завжди → Role замість User для сервісів
  Ніколи → access keys у коді
  AssumeRole → тимчасові STS-ключі

EC2:
  Відмовостійкість → ASG + ALB + Multi-AZ
  Диск → EBS (persistent) vs Instance Store (ephemeral)
  Bootstrap → User Data + cloud-init

VPC:
  Public subnet → маршрут до IGW
  Private subnet → маршрут до NAT GW
  SG = stateful (інстанс), NACL = stateless (підмережа)

S3:
  Versioning → захист від видалення
  Lifecycle → автоматична економія
  Presigned URL → тимчасовий публічний доступ до приватного файлу

RDS:
  Multi-AZ → HA (failover, не масштабування)
  Read Replica → масштабування читань

CloudWatch:
  Metrics → що відбувається
  Logs → чому відбувається
  Alarms → повідомлення + автоматичні дії
  CloudTrail → хто що зробив (аудит API)

Terraform:
  plan → що зміниться
  apply → застосувати
  state → mapping код ↔ реальність
  Remote backend → S3 + DynamoDB lock
```

---
