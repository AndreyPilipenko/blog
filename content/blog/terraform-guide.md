+++
title = "Terraform від А до Я: як він працює, state, модулі, best practices і каверзні питання на співбесіді"
date = 2026-10-16T00:00:00+03:00
draft = false
tags = ["terraform", "iac", "devops", "aws", "interview"]
categories = ["DevOps"]
description = "Повний мануал по Terraform для DevOps: навіщо потрібен IaC, як Terraform працює всередині, HCL, state і locking, модулі, середовища, імпорт і рефакторинг, секрети, CI/CD, тестування, best practices, траблшутінг і понад 50 каверзних питань зі співбесід з відповідями."
+++

# Terraform від А до Я: повний мануал для DevOps-інженера

> Terraform є майже в кожній вакансії DevOps/SRE/Platform Engineer. Але більшість туторіалів закінчуються на «ось `resource "aws_instance"`, тепер натисни `apply`». Ця стаття про інше: **навіщо** Terraform узагалі потрібен, **як він працює всередині**, чому state — найважливіше, що в ньому є, як організувати код для команди, як не знести продакшен однією командою, і що питають на співбесідах, коли хочуть перевірити, чи ти справді з ним працював.

Приклади в статті переважно для AWS, бо це найпоширеніший випадок. Але все, що стосується самого Terraform (state, модулі, lifecycle, CI/CD), так само працює з GCP, Azure, Kubernetes, Cloudflare, Proxmox чи будь-чим іншим, що має API.

---

## Зміст

1. [Навіщо потрібен Terraform: життя без IaC](#why)
2. [Terraform серед інших інструментів](#compare)
3. [Як Terraform працює всередині](#internals)
4. [Встановлення і перший проєкт без хмари](#first-project)
5. [Мова HCL: блоки, змінні, вирази, функції](#hcl)
6. [Meta-arguments: count, for_each, depends_on, lifecycle](#meta)
7. [State — найважливіша тема](#state)
8. [Провайдери, версії і lock-файл](#providers)
9. [Модулі](#modules)
10. [Структура проєкту і середовища: workspaces, каталоги, Terragrunt](#structure)
11. [Рефакторинг та імпорт: moved, import, removed](#refactoring)
12. [Секрети і безпека](#security)
13. [Terraform у CI/CD](#cicd)
14. [Тестування Terraform-коду](#testing)
15. [Best practices — зведений список](#best-practices)
16. [Команди, які треба знати напам'ять](#commands)
17. [Траблшутінг: типові помилки](#troubleshooting)
18. [Каверзні питання на співбесіді — з відповідями](#interview)
19. [Чек-лист](#checklist)

---

## 1. Навіщо потрібен Terraform: життя без IaC {#why}

### 1.1 Типова історія

Уяви компанію, у якої за два роки в AWS накопичилось: 3 VPC, 40 security groups, 2 бази RDS, кластер EKS, десяток S3-бакетів і купа IAM-ролей. Усе це створювали руками в консолі троє різних людей, двоє з яких уже звільнились. Такий підхід має назву **ClickOps**.

А тепер кілька простих питань:

- **Треба підняти stage, «такий самий, як прод».** Який саме? Ніхто не пам'ятає всіх галочок, які ставили в консолі два роки тому.
- **Хто відкрив порт 22 на `0.0.0.0/0` у продакшен security group і навіщо?** CloudTrail, можливо, покаже хто. Але «навіщо» він не покаже ніколи.
- **Акаунт скомпрометували або хтось видалив половину ресурсів.** Скільки займе відновлення? Тижні: доведеться вручну відтворювати сотні налаштувань по пам'яті й по скриншотах.
- **Новий інженер питає, як влаштована мережа.** Єдина відповідь: «відкрий консоль і подивись».

Кожен такий сервер чи бакет — **snowflake** (сніжинка): унікальний, ніде не задокументований, точно відтворити його неможливо.

### 1.2 Що таке Infrastructure as Code

**Infrastructure as Code (IaC)** — підхід, коли інфраструктура описується текстовими файлами, які лежать у Git, а створює і змінює її інструмент, а не людина в консолі.

| Проблема ClickOps | Як її вирішує IaC |
|---|---|
| Ніхто не знає, як усе налаштовано | Код і є документацією, причому завжди актуальною |
| «Хто і навіщо це змінив?» | `git log` і `git blame`: автор, дата, PR з обговоренням |
| Помилки помічають уже на проді | Code review **до** змін: колеги бачать і код, і `plan` |
| Stage не схожий на prod | Ті самі модулі з різними параметрами |
| Відновлення після катастрофи — тижні | `terraform apply` піднімає інфраструктуру заново за години (дані, звісно, з бекапів) |
| Нове середовище — тиждень ручної роботи | Скопіював каталог, змінив змінні, запустив |
| Незрозуміло, за що ми платимо | Видно всі ресурси з тегами; Infracost рахує вартість змін ще в PR |

> **Головна ідея:** IaC — не про те, щоб «швидше клікати». Він дає інфраструктурі **єдине джерело правди** (single source of truth), яке можна читати, рев'ювити, версіонувати й відтворювати.

### 1.3 Декларативний vs імперативний підхід

Автоматизувати інфраструктуру можна і bash-скриптом з `aws cli`:

```bash
# Імперативно: "ЗРОБИ ось це"
aws ec2 run-instances --image-id ami-0abc --instance-type t3.micro --count 2
```

Запусти скрипт двічі — отримаєш **4** сервери. Хочеш 3 замість 2 — пиши логіку «порахуй, скільки вже є, і додай різницю». Хочеш видалити — окремий скрипт. Це **імперативний** підхід: ти описуєш **кроки**.

Terraform **декларативний**: ти описуєш **кінцевий результат**, а як до нього дійти, вирішує інструмент.

```hcl
# Декларативно: "Я ХОЧУ, щоб існувало ось це"
resource "aws_instance" "web" {
  count         = 2
  ami           = "ami-0abc"
  instance_type = "t3.micro"
}
```

Запусти `terraform apply` десять разів — серверів буде рівно **2**. Зміниш на `count = 3` — Terraform створить ще один. Зміниш на `1` — видалить два. Ця властивість називається **ідемпотентністю**: повторне застосування того самого коду не змінює результат.

### 1.4 Аналогія: креслення і прораб

- **Код (`.tf`-файли)** — креслення будинку. Описує, яким будинок має бути.
- **State** — журнал прораба: що вже збудовано і де саме (адреса кожної стіни — це ID ресурсу в хмарі).
- **Terraform** — прораб. Він приходить, звіряє креслення з журналом і з тим, що реально стоїть на ділянці, і складає кошторис: «добудувати дві кімнати, перефарбувати фасад, знести сарай». Це **plan**.
- Ти кажеш «так», і прораб виконує роботи. Це **apply**.

Важливий нюанс, який часто питають на співбесідах: **прораб не живе на будмайданчику**. Terraform — не демон і не контролер, як Kubernetes чи Crossplane. Він працює лише тоді, коли ти його запускаєш. Якщо між запусками хтось руками добудував сарай, Terraform побачить це тільки під час наступного `plan` і запропонує сарай знести, бо в кресленні його немає. Така розбіжність називається **drift**.

### 1.5 Чим Terraform НЕ є

- **Не система керування конфігурацією.** Встановити пакети всередині VM, розкласти конфіги, перезапустити сервіси — це Ansible, cloud-init, Packer. Terraform створює VM, але не адмініструє те, що всередині.
- **Не інструмент деплою застосунків.** Викатувати нову версію застосунку кожні 20 хвилин через Terraform — погана ідея. Для цього є Helm, ArgoCD, CI/CD.
- **Не бекап.** Якщо Terraform видалить базу даних, дані зникнуть разом із нею. Код відтворює **інфраструктуру**, а не **дані**.
- **Не постійний reconciler.** Між запусками він ні за чим не стежить (див. drift вище).

### 1.6 Чим Terraform може керувати

Будь-чим, що має API і для чого існує **провайдер**: AWS, GCP, Azure, Hetzner, DigitalOcean, Kubernetes і Helm, Cloudflare і DNS, GitHub (репозиторії, команди, branch protection), Datadog, Grafana, PagerDuty, Vault, Proxmox, vSphere. У Terraform Registry тисячі провайдерів. Одна мова і один інструмент для всієї платформи: однією конфігурацією можна створити кластер, DNS-запис у Cloudflare і дашборд у Grafana.

---

## 2. Terraform серед інших інструментів {#compare}

| | Terraform | OpenTofu | CloudFormation | Pulumi | Ansible | Crossplane |
|---|---|---|---|---|---|---|
| Мова | HCL | HCL | YAML / JSON | Python, TS, Go, C#, Java | YAML (playbooks) | YAML (Kubernetes CRD) |
| Підхід | Декларативний | Декларативний | Декларативний | Декларативний результат, імперативна мова | Процедурний (задачі по черзі) | Декларативний, постійна reconciliation |
| State | Свій файл (локально або в backend) | Те саме + вбудоване шифрування | Зберігає AWS (stack) | Pulumi Cloud або свій backend | Немає | etcd кластера Kubernetes |
| Хмари | Будь-які | Будь-які | Лише AWS | Будь-які | Будь-які, але слабко з видаленням | Будь-які (через свої провайдери) |
| Rollback при помилці | Немає | Немає | Автоматичний rollback стеку | Немає | Немає | Постійно доводить до бажаного стану |
| Ліцензія | BSL 1.1 | MPL 2.0 (open source) | Сервіс AWS | Apache 2.0 | GPL-3.0 | Apache 2.0 |
| Коли брати | Стандарт індустрії, мультихмара | Потрібен справжній open source | Тільки AWS, хочеш нативний сервіс | Команда любить мови програмування, складна логіка | Конфігурація серверів | Платформа на Kubernetes, self-service через CRD |

**Terraform vs Ansible** — класичне питання на співбесіді. Ansible теж уміє створювати ресурси в хмарі, але **в нього немає state**. Якщо видалити з плейбука задачу «створи бакет», бакет залишиться в хмарі назавжди: Ansible про нього просто забуде. Terraform пам'ятає все, що створив, тож якщо ресурс зник із коду, він буде видалений. На практиці їх поєднують: **Terraform створює інфраструктуру** (VPC, VM, балансувальники), **Ansible налаштовує те, що всередині VM**. Ще краще, коли Packer заздалегідь збирає готовий образ (immutable infrastructure), а Terraform лише запускає VM з нього.

**Terraform vs OpenTofu.** У серпні 2023 року HashiCorp змінила ліцензію Terraform з open source MPL 2.0 на Business Source License (BSL 1.1). Для компаній, які просто користуються Terraform, нічого не змінилось, але BSL забороняє будувати на ньому конкурентні комерційні продукти. У відповідь спільнота зробила форк **OpenTofu** під егідою Linux Foundation (тепер це проєкт CNCF). Він сумісний з Terraform за мовою і форматом state, працює з тими самими провайдерами і має кілька власних фіч (наприклад, шифрування state на боці клієнта). У 2025 році IBM завершила купівлю HashiCorp. На співбесіді достатньо знати: команди майже ідентичні (`tofu plan` замість `terraform plan`), а вибір зазвичай про ліцензію, enterprise-підтримку і конкретні фічі.

**Terraform vs CloudFormation.** CloudFormation — нативний сервіс AWS: state зберігає сам AWS, а при невдалому деплої стек автоматично відкочується. Але він працює тільки з AWS. Terraform — одна мова для AWS, Cloudflare, GitHub і Datadog одночасно, зате rollback'у в ньому немає (про це в розділі 17).

**Terraform vs Pulumi.** Pulumi використовує ту саму модель (state, preview, up), але код пишеться звичайною мовою програмування з циклами, класами і тестами. Сила Pulumi — гнучкість, слабкість — та сама гнучкість: інфраструктурний код перетворюється на звичайний застосунок зі складною логікою, який важче рев'ювити.

---

## 3. Як Terraform працює всередині {#internals}

### 3.1 Core і провайдери

Terraform складається з двох частин:

- **Terraform Core** — бінарник `terraform`. Читає код, будує граф залежностей, порівнює стани, складає план, керує state. **Core нічого не знає ні про AWS, ні про Kubernetes, ні про будь-яку іншу систему.**
- **Провайдери** — окремі бінарники-плагіни (`terraform-provider-aws`, `terraform-provider-kubernetes`...), які вміють говорити з конкретним API. Кожен тип ресурсу в провайдері реалізує операції CRUD (Create, Read, Update, Delete) і описує схему: які в ресурсу є атрибути і зміна яких вимагає перестворення.

Core запускає провайдер як окремий процес і спілкується з ним через **gRPC** (локально, не по мережі). Провайдери завантажуються під час `terraform init` з Terraform Registry у каталог `.terraform/providers/`.

```
                    ┌──────────────────── Terraform Core ─────────────────────┐
  *.tf (код) ──────▶│ 1. парсить HCL                                          │
                    │ 2. будує граф залежностей (DAG)                         │
  state ◀──────────▶│ 3. порівнює код, state і реальність                     │
                    │ 4. складає plan і виконує його, обходячи граф паралельно│
                    └─────────┬──────────────────┬──────────────────┬─────────┘
                              │ gRPC             │ gRPC             │ gRPC
                     ┌────────▼───────┐ ┌────────▼───────┐ ┌────────▼───────┐
                     │ provider: aws  │ │ provider: k8s  │ │ provider:      │
                     │                │ │                │ │ cloudflare     │
                     └────────┬───────┘ └────────┬───────┘ └────────┬───────┘
                              │ HTTPS            │ HTTPS            │ HTTPS
                           AWS API         K8s API server     Cloudflare API
```

Хто пише провайдери: HashiCorp (**Official**: `aws`, `azurerm`, `google`, `kubernetes`), компанії-партнери (**Partner**: `cloudflare`, `datadog`) і спільнота (**Community**). Ці позначки видно в Registry.

Важливий наслідок: **якість роботи з конкретним сервісом залежить від провайдера, а не від Terraform.** Помилки на кшталт «plan постійно показує зміни» або `Provider produced inconsistent result` — майже завжди баги провайдера.

### 3.2 Робочий цикл

```
  write ──▶ init ──▶ plan ──▶ apply ──▶ (зміни коду) ──▶ plan ──▶ apply ... ──▶ destroy
```

| Команда | Що робить |
|---|---|
| `terraform init` | Завантажує провайдери й модулі, налаштовує backend (де лежить state), створює або оновлює `.terraform.lock.hcl`. Запускати можна скільки завгодно разів |
| `terraform plan` | Оновлює знання про реальні ресурси (refresh), порівнює їх із кодом і показує, що буде зроблено. **Нічого не змінює в інфраструктурі** |
| `terraform apply` | Складає план (або бере збережений), питає підтвердження і виконує його |
| `terraform destroy` | Видаляє все, що є в цьому state. По суті це `apply` з планом «видалити все» |

### 3.3 Три стани: код, state і реальність

Ключ до розуміння Terraform: він завжди працює з **трьома** версіями правди.

```
   Код (*.tf)              State (tfstate)                Реальність (API хмари)
   "що я ХОЧУ"             "що я ПАМ'ЯТАЮ"                "що Є насправді"
        │                        │                                │
        │                        │◀──────────── refresh ──────────┤
        │                        │ (оновити пам'ять з реальності) │
        └───────────┬────────────┘                                │
                    ▼                                             │
              diff = plan                                         │
                    │                                             │
                    └────────────────── apply ───────────────────▶│
                           (змінити реальність і записати це в state)
```

1. **Refresh.** Для кожного ресурсу зі state провайдер робить запит Read до API: «який зараз стан ресурсу з ID `i-0abc`?». Якщо ресурс змінили руками, Terraform про це дізнається. Якщо ресурс видалили, він зникне зі state, і план запропонує створити його знову.
2. **Diff.** Оновлений state порівнюється з кодом.
3. **Plan.** Для кожного ресурсу визначається дія: створити, змінити, перестворити, видалити або нічого не робити.

### 3.4 Як читати plan

```
Terraform will perform the following actions:

  # aws_instance.web must be replaced
-/+ resource "aws_instance" "web" {
      ~ ami                          = "ami-0a1b2c3d" -> "ami-0e4f5a6b" # forces replacement
      ~ id                           = "i-0abc123def456" -> (known after apply)
      ~ public_ip                    = "3.120.15.10" -> (known after apply)
        tags                         = {
            "Name" = "web"
        }
        # (28 unchanged attributes hidden)
    }

  # aws_s3_bucket.logs will be updated in-place
  ~ resource "aws_s3_bucket" "logs" {
        id   = "acme-logs"
      ~ tags = {
          ~ "Environment" = "dev" -> "prod"
        }
        # (10 unchanged attributes hidden)
    }

  # aws_iam_user.old will be destroyed
  # (because aws_iam_user.old is not in configuration)
  - resource "aws_iam_user" "old" {
      - name = "old-user" -> null
        # (5 unchanged attributes hidden)
    }

Plan: 1 to add, 1 to change, 2 to destroy.
```

| Символ | Дія | Коли |
|---|---|---|
| `+` | create | Ресурс є в коді, але немає в state |
| `-` | destroy | Ресурс є в state, але зник із коду |
| `~` | update in-place | Змінився атрибут, який можна оновити без перестворення |
| `-/+` | replace: спершу видалити, потім створити | Змінився атрибут із позначкою `# forces replacement` |
| `+/-` | replace: спершу створити, потім видалити | Те саме, але з `create_before_destroy = true` |
| `<=` | read | Data source, який можна прочитати лише під час apply |

`(known after apply)` — значення, яке стане відомим тільки після створення ресурсу (ID, IP, ARN). Тому план — це **прогноз**, а не гарантія: apply ще може впасти через помилку API, ліміти чи права.

> **Правило виживання:** завжди шукай у плані `must be replaced` і `forces replacement`. Заміна бази, диска чи балансувальника означає втрату даних або даунтайм. Рядок `Plan: 0 to add, 3 to change, 0 to destroy` читається за секунду — читай його **завжди**.

### 3.5 Граф залежностей

Terraform не виконує код зверху вниз. Порядок блоків у файлах і навіть назви файлів не мають значення: Terraform зчитує всі `*.tf` у каталозі (без підкаталогів), об'єднує їх і будує **DAG** (directed acyclic graph — орієнтований граф без циклів).

Залежності визначаються **посиланнями**:

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "private" {
  vpc_id     = aws_vpc.main.id # посилання = залежність: subnet після VPC
  cidr_block = "10.0.1.0/24"
}

resource "aws_instance" "app" {
  ami           = "ami-0abc"
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.private.id # instance після subnet
}

resource "aws_s3_bucket" "logs" { # ні від чого не залежить,
  bucket = "acme-logs"            # тому створюється паралельно з VPC
}
```

```
        aws_vpc.main          aws_s3_bucket.logs
             │
             ▼
     aws_subnet.private
             │
             ▼
      aws_instance.app
```

- Незалежні ресурси створюються **паралельно**: за замовчуванням до 10 операцій одночасно (`-parallelism=10`).
- Видалення відбувається у **зворотному** порядку: спершу instance, потім subnet, потім VPC.
- Подивитися граф: `terraform graph | dot -Tsvg > graph.svg` (потрібен Graphviz).

Якщо залежність існує, але Terraform не може побачити її з посилань, використовують `depends_on` (розділ 6.3).

---

## 4. Встановлення і перший проєкт без хмари {#first-project}

### 4.1 Встановлення

Ubuntu/Debian з офіційного репозиторію HashiCorp:

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt install terraform
terraform version
```

Різні проєкти часто вимагають різних версій Terraform, тому зручніше користуватися менеджером версій:

```bash
# tfenv
tfenv install 1.16.6
tfenv use 1.16.6
echo "1.16.6" > .terraform-version   # tfenv сам підхопить версію в цьому каталозі

# або mise (універсальний менеджер версій)
mise use terraform@1.16.6
```

Корисні дрібниці:

```bash
terraform -install-autocomplete   # автодоповнення для bash/zsh
alias tf=terraform                # так роблять майже всі
```

### 4.2 Перший проєкт

Щоб спробувати Terraform, хмара не потрібна. Провайдери `random` і `local` працюють просто на твоїй машині.

```bash
mkdir tf-hello && cd tf-hello
```

```hcl
# main.tf
terraform {
  required_version = ">= 1.6"

  required_providers {
    random = {
      source  = "hashicorp/random"
      version = "~> 3.9"
    }
    local = {
      source  = "hashicorp/local"
      version = "~> 2.9"
    }
  }
}

# Генерує випадкове ім'я на кшталт "brave-otter"
resource "random_pet" "server" {
  length = 2
}

# Створює файл на диску. Посилання на random_pet = залежність
resource "local_file" "hello" {
  filename = "${path.module}/hello.txt"
  content  = "Привіт від ${random_pet.server.id}!\n"
}

output "pet_name" {
  value = random_pet.server.id
}
```

Проходимо весь цикл:

```bash
terraform init      # завантажить провайдери random і local
terraform plan      # Plan: 2 to add, 0 to change, 0 to destroy.
terraform apply     # спитає "Enter a value:" — введи yes
cat hello.txt       # Привіт від brave-otter!
```

А тепер найкорисніші експерименти, щоб відчути, як Terraform думає:

```bash
# 1. Ідемпотентність: ще раз apply
terraform apply
# No changes. Your infrastructure matches the configuration.

# 2. Drift: видали файл руками, ніби "хтось зайшов у консоль"
rm hello.txt
terraform plan
# local_file.hello will be created — Terraform помітив, що реальність розійшлася зі state

# 3. Зміни content у main.tf і подивись на план.
#    local_file не вміє оновлюватися на місці, тому побачиш "-/+ ... forces replacement"

# 4. Подивись, що Terraform пам'ятає
terraform state list
terraform state show random_pet.server

# 5. Прибери за собою
terraform destroy
```

### 4.3 Що з'явилося в каталозі

```
tf-hello/
├── main.tf
├── .terraform/                # завантажені провайдери і модулі — НЕ комітимо
│   └── providers/registry.terraform.io/hashicorp/...
├── .terraform.lock.hcl        # точні версії і хеші провайдерів — КОМІТИМО
├── terraform.tfstate          # state — НЕ комітимо (у команді — remote backend)
└── terraform.tfstate.backup   # попередня версія state
```

Стандартний `.gitignore` для Terraform:

```text
# Локальні каталоги з провайдерами і модулями
.terraform/

# State і його бекапи — можуть містити секрети
*.tfstate
*.tfstate.*

# Збережені плани — теж можуть містити секрети
*.tfplan
tfplan

# Crash-логи
crash.log
crash.*.log

# Override-файли — зазвичай локальні експерименти
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Локальний конфіг CLI (може містити токени)
.terraformrc
terraform.rc
```

> `*.tfvars` комітити можна і треба, якщо в них параметри середовища (розміри інстансів, CIDR). Секретів там не повинно бути взагалі, про них — у розділі 12.

### 4.4 Структура файлів

Terraform склеює всі `*.tf` з каталогу, тож розбиття на файли потрібне лише людям. Загальноприйнята конвенція:

| Файл | Що в ньому |
|---|---|
| `versions.tf` | Блок `terraform {}`: `required_version`, `required_providers`, часто `backend` |
| `providers.tf` | Налаштування провайдерів (регіон, `assume_role`, `default_tags`) |
| `main.tf` | Основні ресурси (у великих проєктах ділять за темою: `network.tf`, `iam.tf`, `rds.tf`) |
| `variables.tf` | Вхідні змінні |
| `outputs.tf` | Вихідні значення |
| `locals.tf` | Локальні значення, якщо їх багато |
| `data.tf` | Data sources (за бажанням) |
| `terraform.tfvars`, `prod.tfvars` | Значення змінних |

---

## 5. Мова HCL: блоки, змінні, вирази, функції {#hcl}

HCL (HashiCorp Configuration Language) — декларативна мова, щось середнє між JSON і мовою програмування. Вона навмисно обмежена: є змінні, умови, цикли-вирази і вбудовані функції, але немає власних функцій, класів і довільної логіки. Це фіча, а не баг: інфраструктурний код має читатися як опис, а не як програма.

### 5.1 Анатомія блоку

```hcl
resource "aws_instance" "web" { # <тип блоку> "<тип ресурсу>" "<локальне ім'я>"
  ami           = "ami-0abc123" # аргумент = вираз
  instance_type = "t3.micro"

  root_block_device { # вкладений блок
    volume_size = 20
    encrypted   = true
  }

  tags = { # аргумент типу map
    Name = "web"
  }
}
```

- **Тип ресурсу** `aws_instance`: префікс до першого `_` (`aws`) вказує, який провайдер його обслуговує.
- **Локальне ім'я** `web` існує лише в Terraform-коді. В AWS ресурс називатиметься так, як ти вкажеш у `tags.Name` чи `name`.
- **Адреса ресурсу** — `aws_instance.web`. Через неї на ресурс посилаються інші ресурси (`aws_instance.web.id`) і команди (`terraform state show aws_instance.web`).

Основні типи блоків верхнього рівня:

| Блок | Призначення |
|---|---|
| `terraform` | Налаштування самого Terraform: версії, провайдери, backend |
| `provider` | Налаштування провайдера (регіон, облікові дані, alias) |
| `resource` | Ресурс, яким Terraform **керує** (створює, змінює, видаляє) |
| `data` | Data source: **читає** інформацію про те, що вже існує |
| `variable` | Вхідний параметр |
| `locals` | Локальні обчислювані значення |
| `output` | Вихідне значення |
| `module` | Виклик модуля |
| `moved`, `import`, `removed` | Рефакторинг та імпорт (розділ 11) |
| `check` | Перевірки інфраструктури, що дають warning (розділ 6.6) |
| `ephemeral` | Тимчасові ресурси, які не записуються в state (1.10+, розділ 12) |
| `action` | Дії поза CRUD: інвалідація CloudFront, виклик Lambda (1.14+, розділ 6.7) |

### 5.2 Типи даних

| Тип | Приклад | Нотатка |
|---|---|---|
| `string` | `"hello"` | |
| `number` | `42`, `3.14` | І цілі, і дробові |
| `bool` | `true` | |
| `list(type)` | `["a", "b", "a"]` | Упорядкований, з дублікатами, доступ за індексом `[0]` |
| `set(type)` | `toset(["a", "b"])` | Без порядку і без дублікатів, за індексом не звернешся |
| `map(type)` | `{ env = "prod", team = "core" }` | Ключі — рядки, значення одного типу |
| `object({...})` | `{ name = "web", port = 80 }` | Фіксований набір полів різних типів |
| `tuple([...])` | `["web", 80, true]` | Фіксована довжина, різні типи |
| `any` | | «Будь-який»: Terraform виведе тип сам |

Окремо — `null`. Це не «порожній рядок», а «аргумент не задано»: провайдер візьме значення за замовчуванням. Зручно в умовах: `key_name = var.enable_ssh ? var.key_name : null`.

### 5.3 Змінні (variables)

```hcl
variable "environment" {
  description = "Назва середовища"
  type        = string

  validation {
    condition     = contains(["dev", "stage", "prod"], var.environment)
    error_message = "environment має бути одним із: dev, stage, prod."
  }
}

variable "instance_type" {
  description = "Тип EC2-інстансу"
  type        = string
  default     = "t3.micro" # є default, отже змінна необов'язкова
}

variable "db_password" {
  description = "Пароль до БД"
  type        = string
  sensitive   = true # ховати значення у виводі plan/apply
}

variable "subnets" {
  description = "Підмережі: ключ — назва, значення — параметри"
  type = map(object({
    cidr   = string
    az     = string
    public = optional(bool, false) # необов'язкове поле зі значенням за замовчуванням (1.3+)
  }))
}
```

Звідки Terraform бере значення змінних — **у порядку зростання пріоритету** (кожне наступне перекриває попереднє):

```
 найнижчий ─▶  1. default у блоці variable
               2. змінні оточення TF_VAR_<name>        (export TF_VAR_environment=prod)
               3. terraform.tfvars
               4. terraform.tfvars.json
               5. *.auto.tfvars / *.auto.tfvars.json   (в алфавітному порядку імен файлів)
 найвищий  ─▶  6. -var і -var-file у командному рядку  (у тому порядку, як вказані)
```

Якщо значення немає ніде, а `default` не задано, Terraform спитає його інтерактивно. У CI це означає зависання, тому там завжди `-input=false` (або `TF_INPUT=false`).

Файл зі значеннями:

```hcl
# prod.tfvars
environment   = "prod"
instance_type = "m6i.large"

subnets = {
  public-a  = { cidr = "10.0.1.0/24", az = "eu-central-1a", public = true }
  private-a = { cidr = "10.0.11.0/24", az = "eu-central-1a" }
  private-b = { cidr = "10.0.12.0/24", az = "eu-central-1b" }
}
```

```bash
terraform plan -var-file=prod.tfvars
```

> **Пастка:** `terraform.tfvars` і `*.auto.tfvars` підхоплюються автоматично, а `prod.tfvars` — тільки через `-var-file`. Забув прапорець, і план порахувався з дефолтними значеннями, наприклад для dev.

### 5.4 Locals

```hcl
locals {
  name_prefix = "${var.project}-${var.environment}" # "shop-prod"

  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
  }

  is_prod = var.environment == "prod"
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = local.is_prod ? "m6i.large" : "t3.micro"
  tags          = merge(local.common_tags, { Name = "${local.name_prefix}-app" })
}
```

Аналогія з програмуванням: модуль — це функція. **Variables** — її параметри, **locals** — локальні змінні всередині функції, **outputs** — значення, яке вона повертає.

Locals не можна перевизначити ззовні, і в цьому їхня відмінність від змінних. Якщо значення не повинен налаштовувати користувач модуля, роби його local, а не variable.

### 5.5 Outputs

```hcl
output "vpc_id" {
  description = "ID створеної VPC"
  value       = aws_vpc.main.id
}

output "db_endpoint" {
  description = "Адреса бази"
  value       = aws_db_instance.main.endpoint
  sensitive   = true
}
```

```bash
terraform output                 # усі outputs (sensitive показані як <sensitive>)
terraform output -raw vpc_id     # тільки значення, без лапок — зручно в скриптах
terraform output -json           # JSON для jq (тут sensitive видно відкритим текстом!)
```

Outputs потрібні для трьох речей: показати важливі значення людині після apply, передати значення з дочірнього модуля в батьківський (`module.vpc.vpc_id`) і поділитися значеннями між різними state (розділ 7.9).

### 5.6 Data sources

Data source **читає** дані, що вже існують: ресурси, створені поза цим кодом, або довідкову інформацію від провайдера.

```hcl
# Найсвіжіший образ Ubuntu 24.04 від Canonical
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # офіційний AWS-акаунт Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*"]
  }
}

# Поточний акаунт
data "aws_caller_identity" "current" {}

# VPC, створена іншою командою: шукаємо її за тегом
data "aws_vpc" "shared" {
  tags = {
    Name = "shared-prod"
  }
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
}

locals {
  account_id = data.aws_caller_identity.current.account_id
}
```

| | `resource` | `data` |
|---|---|---|
| Що робить | Створює, змінює, видаляє | Тільки читає |
| Чи є в state | Так, з повним життєвим циклом | Так, але лише як кеш останнього прочитаного |
| Якщо прибрати з коду | Ресурс буде **видалено** | Нічого не станеться |
| Коли виконується | Під час apply | Зазвичай під час plan; якщо залежить від ще не створених значень — під час apply |

> **Міна з `most_recent = true`:** Canonical регулярно випускає оновлені AMI. Data source щоразу повертатиме новий ID, а зміна `ami` в `aws_instance` вимагає **перестворення** інстансу. Одного дня звичайний `apply` запропонує перестворити всі твої сервери. Рішення: зафіксувати AMI змінною, додати `lifecycle { ignore_changes = [ami] }` або використовувати Auto Scaling Group з launch template і поступовим оновленням.

### 5.7 Вирази

```hcl
# Інтерполяція рядків
name = "${var.project}-${var.environment}-web"

# Умовний (тернарний) оператор
instance_type = var.environment == "prod" ? "m6i.large" : "t3.micro"

# Багаторядковий рядок (heredoc); <<- прибирає відступи
user_data = <<-EOT
  #!/bin/bash
  echo "Hello from ${var.environment}" > /etc/motd
EOT

# for-вираз: список → список
upper_names = [for name in var.users : upper(name)]

# for-вираз: map → map з фільтром
public_subnets = { for k, s in var.subnets : k => s.cidr if s.public }

# splat: усі ID зі списку ресурсів, створених через count
instance_ids = aws_instance.web[*].id
```

**`dynamic`** генерує вкладені блоки в циклі:

```hcl
variable "ingress_ports" {
  type    = list(number)
  default = [80, 443]
}

resource "aws_security_group" "web" {
  name   = "web"
  vpc_id = aws_vpc.main.id

  dynamic "ingress" {
    for_each = var.ingress_ports
    content {
      description = "Allow ${ingress.value}"
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```

> Не зловживай `dynamic`. Якщо з коду неможливо зрозуміти, які блоки згенеруються, краще написати їх явно. HashiCorp радить використовувати `dynamic` лише там, де без нього не обійтися, переважно в модулях. А для правил security group в AWS-провайдері краще взагалі окремі ресурси `aws_vpc_security_group_ingress_rule`: так кожне правило має власну адресу в state, і зміна одного правила не перезаписує решту.

### 5.8 Функції, які використовуються щодня

Власних функцій у HCL немає, зате вбудованих понад сотню. Найпотрібніші:

| Функція | Приклад | Результат |
|---|---|---|
| `merge` | `merge({a = 1}, {b = 2})` | `{a = 1, b = 2}` |
| `lookup` | `lookup({prod = "m6i.large"}, "dev", "t3.micro")` | `"t3.micro"` |
| `coalesce` | `coalesce(null, "", "default")` | `"default"` |
| `try` | `try(var.config.port, 8080)` | `8080`, якщо поля немає |
| `can` | `can(regex("^ami-", var.ami))` | `true` / `false`, зручно у validation |
| `length` | `length(["a", "b"])` | `2` |
| `contains` | `contains(["dev", "prod"], "prod")` | `true` |
| `toset` | `toset(["a", "b", "a"])` | `["a", "b"]` (set) |
| `flatten` | `flatten([["a"], ["b", "c"]])` | `["a", "b", "c"]` |
| `zipmap` | `zipmap(["a", "b"], [1, 2])` | `{a = 1, b = 2}` |
| `join` / `split` | `join(",", ["a", "b"])` | `"a,b"` |
| `format` | `format("%s-%03d", "web", 7)` | `"web-007"` |
| `cidrsubnet` | `cidrsubnet("10.0.0.0/16", 8, 1)` | `"10.0.1.0/24"` |
| `jsonencode` | `jsonencode({Version = "2012-10-17"})` | Рядок JSON |
| `templatefile` | `templatefile("${path.module}/init.sh.tftpl", {port = 8080})` | Відрендерений шаблон |
| `file` | `file("${path.module}/policy.json")` | Вміст файлу |

Функції зручно перевіряти в інтерактивній консолі:

```bash
$ terraform console
> cidrsubnet("10.0.0.0/16", 8, 2)
"10.0.2.0/24"
> [for i in range(3) : "web-${i}"]
[
  "web-0",
  "web-1",
  "web-2",
]
> exit
```

---

## 6. Meta-arguments: count, for_each, depends_on, lifecycle {#meta}

Meta-arguments — аргументи, які є в **будь-якого** ресурсу незалежно від провайдера: `count`, `for_each`, `depends_on`, `provider`, `lifecycle`.

### 6.1 count

```hcl
resource "aws_instance" "web" {
  count         = 3
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  tags = {
    Name = "web-${count.index}" # web-0, web-1, web-2
  }
}
```

Адреси в state: `aws_instance.web[0]`, `aws_instance.web[1]`, `aws_instance.web[2]`.

Найчастіше `count` використовують як **вимикач**:

```hcl
resource "aws_instance" "bastion" {
  count         = var.create_bastion ? 1 : 0 # створити або ні
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.nano"
}

output "bastion_ip" {
  value = one(aws_instance.bastion[*].public_ip) # null, якщо bastion не створено
}
```

### 6.2 Головна пастка count, і чому for_each

Створимо користувачів IAM зі списку через `count`:

```hcl
variable "users" {
  default = ["anna", "bohdan", "carl"]
}

resource "aws_iam_user" "this" {
  count = length(var.users)
  name  = var.users[count.index]
}
```

У state вони записані **за індексами**:

```
aws_iam_user.this[0] → anna
aws_iam_user.this[1] → bohdan
aws_iam_user.this[2] → carl
```

Анна звільнилась, прибираємо її зі списку: `["bohdan", "carl"]`. Що зробить Terraform?

```
aws_iam_user.this[0]: anna   → bohdan   (перейменувати!)
aws_iam_user.this[1]: bohdan → carl     (перейменувати!)
aws_iam_user.this[2]: carl   → DESTROY
```

Ти хотів видалити одного користувача, а Terraform зачіпає **всіх трьох** і видаляє не того: фізично зникне об'єкт, який був користувачем carl. Ресурси, які не можна перейменувати на місці (сервер, бакет, база), взагалі будуть перестворені, а це даунтайм і втрата даних.

Те саме через `for_each`:

```hcl
resource "aws_iam_user" "this" {
  for_each = toset(var.users)
  name     = each.key
}
```

```
aws_iam_user.this["anna"]   → anna
aws_iam_user.this["bohdan"] → bohdan
aws_iam_user.this["carl"]   → carl
```

Прибираємо anna, і Terraform видаляє рівно `aws_iam_user.this["anna"]`, нічого більше. Ключ стабільний, бо залежить від самого значення, а не від позиції в списку.

`for_each` з map, коли в кожного елемента є власні параметри:

```hcl
variable "buckets" {
  type = map(object({
    versioning = bool
  }))
  default = {
    logs    = { versioning = false }
    backups = { versioning = true }
  }
}

resource "aws_s3_bucket" "this" {
  for_each = var.buckets
  bucket   = "acme-${each.key}" # each.key = "logs" / "backups", each.value = об'єкт
}

resource "aws_s3_bucket_versioning" "this" {
  for_each = { for k, v in var.buckets : k => v if v.versioning }

  bucket = aws_s3_bucket.this[each.key].id
  versioning_configuration {
    status = "Enabled"
  }
}
```

| | `count` | `for_each` |
|---|---|---|
| Приймає | Число | `map` або `set(string)` |
| Адреса | `res[0]`, `res[1]` | `res["logs"]`, `res["backups"]` |
| Видалення елемента з середини | Зсуває індекси, зачіпає сусідів | Зачіпає тільки цей елемент |
| Коли використовувати | Вимикач `0/1`, N повністю однакових ресурсів | Колекції з унікальними іменами — майже завжди |

Обмеження `for_each`:

- **Ключі мають бути відомі на етапі plan.** Значення, що з'являться лише після apply (ID, ARN, IP), ключами бути не можуть: отримаєш помилку `Invalid for_each argument` (як її обійти — у розділі 17.3).
- Ключі не можуть бути sensitive.
- `list` треба перетворити на `set` через `toset()`, і дублікати зникнуть.

### 6.3 depends_on

Зазвичай Terraform сам виводить залежності з посилань. `depends_on` потрібен, коли залежність є, але **в коді її не видно**:

```hcl
resource "aws_iam_role_policy" "app_s3" {
  role   = aws_iam_role.app.id
  policy = data.aws_iam_policy_document.app_s3.json
}

resource "aws_instance" "app" {
  ami                  = data.aws_ami.ubuntu.id
  instance_type        = "t3.micro"
  iam_instance_profile = aws_iam_instance_profile.app.name

  # Застосунок під час старту читає S3. Посилання на профіль є,
  # а на саму політику з правами — ні. Без depends_on інстанс
  # може стартувати раніше, ніж роль отримає права.
  depends_on = [aws_iam_role_policy.app_s3]
}
```

> **Пастка:** `depends_on` на **модулі** або на **data source** робить залежність грубою. Data source усередині такого модуля вже не може прочитатися під час plan: читання відкладається до apply, і все, що від нього залежить, стає `(known after apply)`. Результат — план, який при кожному запуску показує зміни, а то й перестворення ресурсів. Коли можливо, посилайся на конкретні outputs замість `depends_on = [module.x]`.

### 6.4 lifecycle

```hcl
resource "aws_db_instance" "main" {
  # ...

  lifecycle {
    prevent_destroy = true                 # будь-який план із видаленням цього ресурсу впаде з помилкою
    ignore_changes  = [tags["LastBackup"]] # не зважати на зміни цього тегу ззовні
  }
}
```

**`create_before_destroy`** змінює порядок заміни: спершу створити новий ресурс, потім видалити старий (`+/-` у плані). Це потрібно для оновлень без простою:

```hcl
resource "aws_security_group" "app" {
  name_prefix = "app-" # не name: два SG з однаковим name в одній VPC існувати не можуть
  vpc_id      = aws_vpc.main.id

  lifecycle {
    create_before_destroy = true
  }
}
```

Дві речі, які треба знати про `create_before_destroy`:

1. **Унікальні імена.** Якщо в ресурсу фіксоване ім'я, новий ресурс не створиться, поки існує старий з таким самим ім'ям. Тому разом із `create_before_destroy` використовують `name_prefix` або суфікс з `random_id`.
2. **Він «заразний».** Якщо ресурс A з `create_before_destroy` залежить від ресурсу B, Terraform застосує таку саму поведінку і до B, інакше правильний порядок операцій неможливий.

**`ignore_changes`** ігнорує зміни атрибутів, які робить хтось інший:

```hcl
resource "aws_autoscaling_group" "app" {
  min_size         = 2
  max_size         = 10
  desired_capacity = 2
  # ...

  lifecycle {
    ignore_changes = [desired_capacity] # кількість інстансів змінює autoscaler, а не Terraform
  }
}
```

Без цього кожен apply повертав би `desired_capacity` до 2 і вбивав би інстанси, які autoscaler додав під навантаженням.

**`replace_triggered_by`** перестворює ресурс, коли змінився інший ресурс:

```hcl
# terraform_data — вбудований "порожній" ресурс, який просто зберігає значення (1.4+)
resource "terraform_data" "config_version" {
  input = var.config_version
}

resource "aws_instance" "app" {
  # ...

  lifecycle {
    # змінилась версія конфігурації → перестворити інстанс
    replace_triggered_by = [terraform_data.config_version]
  }
}
```

**`prevent_destroy`** має два обмеження, про які часто не знають (і які люблять питати на співбесідах):

1. Він захищає лише поки блок ресурсу **є в коді**. Якщо видалити блок ресурсу разом із його `lifecycle`, Terraform спокійно видалить ресурс. Від `terraform destroy` і від випадкової заміни захист є, від видалення коду — немає.
2. Це захист лише на рівні Terraform. Хтось у консолі чи через CLI видалить базу без жодних проблем. Тому для критичних ресурсів вмикають ще й захист на боці хмари: `deletion_protection = true` для RDS, `disable_api_termination = true` для EC2, MFA Delete чи Object Lock для S3.

> З Terraform 1.16 є ще один інструмент: `lifecycle { destroy = false }`. Ресурс, який мав би бути видалений, Terraform просто прибирає зі state, а в хмарі залишає (детальніше в розділі 11.3).

### 6.5 Провайдер з alias: кілька регіонів і акаунтів

```hcl
provider "aws" {
  region = "eu-central-1" # провайдер за замовчуванням
}

provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}

# Сертифікат для CloudFront зобов'язаний бути в us-east-1
resource "aws_acm_certificate" "cdn" {
  provider          = aws.us_east_1
  domain_name       = "example.com"
  validation_method = "DNS"
}
```

Інший акаунт — через assume role:

```hcl
provider "aws" {
  alias  = "prod"
  region = "eu-central-1"

  assume_role {
    role_arn = "arn:aws:iam::111111111111:role/terraform"
  }
}
```

> В AWS-провайдері версії 6 більшість ресурсів отримали аргумент `region`, тож для кількох регіонів в одному акаунті alias уже не обов'язковий. Але для різних акаунтів і для інших провайдерів alias, як і раніше, основний механізм.

### 6.6 Перевірки: validation, precondition, postcondition, check

| Механізм | Де | Що перевіряє | Результат при провалі |
|---|---|---|---|
| `validation` | `variable` | Вхідне значення змінної | Помилка, plan зупиняється |
| `precondition` | `lifecycle` ресурсу, `output` | Припущення **до** створення ресурсу | Помилка |
| `postcondition` | `lifecycle` ресурсу | Гарантію **після** створення чи читання | Помилка |
| `check` | Окремий блок (1.5+) | Стан інфраструктури загалом | Тільки **warning**, apply не зупиняється |

```hcl
resource "aws_instance" "app" {
  ami           = data.aws_ami.selected.id
  instance_type = var.instance_type

  lifecycle {
    precondition {
      condition     = data.aws_ami.selected.architecture == "x86_64"
      error_message = "AMI має бути x86_64: instance_type розрахований на Intel/AMD."
    }
  }
}

check "app_health" {
  data "http" "health" {
    url = "https://${aws_lb.app.dns_name}/healthz"
  }

  assert {
    condition     = data.http.health.status_code == 200
    error_message = "Застосунок не повертає 200 на /healthz"
  }
}
```

### 6.7 Provisioners і actions

**Provisioners** (`local-exec`, `remote-exec`, `file`) запускають команди під час створення або видалення ресурсу:

```hcl
resource "aws_instance" "web" {
  # ...

  provisioner "local-exec" {
    command = "echo ${self.private_ip} >> inventory.ini"
  }
}
```

HashiCorp прямо називає їх **крайнім заходом** (last resort). Їх не видно в плані, вони не ідемпотентні, результат їхньої роботи не зберігається в state, а якщо команда впаде, ресурс стане tainted, і наступний apply його перестворить. Майже завжди є краще рішення: `user_data`/cloud-init для початкового налаштування VM, готовий образ із Packer, Ansible після Terraform або нативний ресурс провайдера.

Для операцій, які не вкладаються в модель CRUD (скинути кеш CloudFront, викликати Lambda, зупинити інстанс), з Terraform 1.14 є **actions** — дії, які описує сам провайдер:

```hcl
action "aws_cloudfront_create_invalidation" "all" {
  config {
    distribution_id = aws_cloudfront_distribution.site.id
    paths           = ["/*"]
  }
}

resource "terraform_data" "site_version" {
  input = var.site_version

  lifecycle {
    action_trigger {
      events  = [after_update]                                  # змінилася версія сайту...
      actions = [action.aws_cloudfront_create_invalidation.all] # ...скинути кеш CDN
    }
  }
}
```

На відміну від provisioner'а, action видно в плані, і це перевірена частина провайдера, а не довільна shell-команда. Які actions доступні, дивись у документації провайдера.

---

## 7. State — найважливіша тема {#state}

Якщо з усієї статті запам'ятати одну річ, нехай це буде вона: **state — серце Terraform, і більшість серйозних аварій з Terraform пов'язані саме з ним.**

### 7.1 Що лежить у state

`terraform.tfstate` — це JSON-файл. Спрощено він виглядає так:

```json
{
  "version": 4,
  "terraform_version": "1.16.6",
  "serial": 42,
  "lineage": "3f1c9e0a-7b2d-4c1e-9a8f-1d2e3f4a5b6c",
  "outputs": {
    "vpc_id": { "value": "vpc-0a1b2c3d4e5f", "type": "string" }
  },
  "resources": [
    {
      "mode": "managed",
      "type": "aws_db_instance",
      "name": "main",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "attributes": {
            "identifier": "prod-db",
            "endpoint": "prod-db.c9akciq32.eu-central-1.rds.amazonaws.com:5432",
            "username": "app",
            "password": "SuperSecret123!",
            "...": "..."
          },
          "dependencies": ["aws_db_subnet_group.main", "aws_security_group.db"]
        }
      ]
    }
  ]
}
```

Що тут важливо:

- **Зв'язок «адреса в коді ↔ реальний об'єкт»**: `aws_db_instance.main` — це база `prod-db`.
- **Усі атрибути ресурсів**, зокрема ті, що повернув API.
- **Пароль лежить відкритим текстом.** Так, `"password": "SuperSecret123!"`. Те саме стосується приватних ключів з `tls_private_key`, токенів і всього, що провайдер зберігає як атрибут. `sensitive = true` ховає значення лише з виводу в терміналі, а в state воно лежить як є.
- **`serial`** — лічильник версій, збільшується з кожним записом state.
- **`lineage`** — унікальний ID, що генерується при створенні state. Разом із `serial` вони захищають від помилок: Terraform не дасть перезаписати state «чужим» (інший lineage) або старішим (менший serial) файлом без `-force`.
- **`dependencies`** — залежності. Вони потрібні, щоб правильно видалити ресурс, навіть коли його вже немає в коді.

### 7.2 Навіщо state взагалі потрібен

Класичне питання на співбесіді: «Чому Terraform не може просто подивитися в хмару, що там є?»

1. **Зіставлення коду з реальністю.** У хмарі немає універсального способу дізнатися, що «цей ресурс створив ось цей блок коду». Теги є не в усіх ресурсів, і їх можна змінити руками. State — надійний словник `aws_instance.web → i-0abc123`.
2. **Видалення.** Якщо ти прибрав блок із коду, лише state пам'ятає, що такий ресурс існував і його треба видалити. Без state Terraform просто б про нього не знав, як Ansible.
3. **Залежності і метадані.** Порядок видалення, провайдер (і alias), який обслуговує ресурс, — усе це зберігається в state.
4. **Продуктивність.** Для великої інфраструктури state працює як кеш: можна запустити `plan -refresh=false` і не опитувати API про тисячі ресурсів.

### 7.3 Чому локальний state — погана ідея для команди

- Він живе на одному ноутбуці. Ноутбук помер — state зник.
- Двоє людей одночасно запускають `apply` зі своїх копій, state розходиться з реальністю, і наступний apply може видаляти чи дублювати ресурси.
- Секрети зі state лежать на дисках розробників.
- Комітити state у Git ще гірше: секрети залишаються в історії репозиторію назавжди, а конфлікти злиття state-файлів практично нерозв'язні.

Рішення — **remote backend** зі **state locking**.

### 7.4 Remote backend: S3

```hcl
terraform {
  backend "s3" {
    bucket       = "acme-terraform-state"
    key          = "prod/network/terraform.tfstate" # окремий key для кожного state
    region       = "eu-central-1"
    encrypt      = true
    use_lockfile = true # блокування через lock-файл у самому S3
  }
}
```

> **Як змінилося блокування в S3.** Роками стандартом була DynamoDB-таблиця для lock (`dynamodb_table = "..."`), і ти ще зустрінеш її в багатьох туторіалах і старих проєктах. У Terraform 1.10 з'явилося нативне блокування через S3 (`use_lockfile = true`): Terraform кладе поруч зі state об'єкт `.tflock`, використовуючи conditional writes у S3. З версії 1.11 воно стабільне, а блокування через DynamoDB позначене як deprecated. Для нових проєктів DynamoDB-таблиця більше не потрібна. А для міграції старих можна на якийсь час залишити обидва параметри: тоді Terraform бере lock і в DynamoDB, і в S3, і нікого не заблокує під час переходу.

**Проблема курки і яйця:** бакет для state має існувати до того, як ним користуватися. Зазвичай роблять маленьку окрему конфігурацію `bootstrap/` з локальним state (або створюють бакет через CLI), яка один раз створює бакет:

```hcl
# bootstrap/main.tf
resource "aws_s3_bucket" "tf_state" {
  bucket = "acme-terraform-state"

  lifecycle {
    prevent_destroy = true
  }
}

# Versioning — головний рятівник: будь-яку зіпсовану версію state можна відкотити
resource "aws_s3_bucket_versioning" "tf_state" {
  bucket = aws_s3_bucket.tf_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "tf_state" {
  bucket = aws_s3_bucket.tf_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "tf_state" {
  bucket                  = aws_s3_bucket.tf_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

Інші популярні backends:

| Backend | Блокування | Нотатки |
|---|---|---|
| `s3` | `use_lockfile` (або застаріла DynamoDB) | Стандарт для AWS |
| `gcs` | Вбудоване | Стандарт для GCP |
| `azurerm` | Через blob lease | Стандарт для Azure |
| HCP Terraform (`cloud {}`) | Вбудоване | Плюс remote runs, політики, UI, історія state |
| `pg` | Advisory locks PostgreSQL | Зручно on-prem |
| `kubernetes` | Через Lease | State у Secret, а розмір Secret обмежений ~1 МБ |
| `http` | Опціонально | Наприклад, GitLab-managed Terraform state |
| `local` | Лише в межах однієї машини | Використовується за замовчуванням |

### 7.5 Чому в backend не можна використати змінні

Ще одне улюблене питання. Такий код не працює:

```hcl
terraform {
  backend "s3" {
    bucket = var.state_bucket # Error: Variables not allowed
    key    = "${var.env}/terraform.tfstate"
  }
}
```

Причина в тому, що backend ініціалізується під час `terraform init`, **раніше**, ніж Terraform обробляє змінні. Щоб працювати зі змінними, йому потрібен state, а щоб знайти state, потрібен backend.

Рішення — **часткова конфігурація** (partial configuration): у коді залишаєш порожній або неповний блок, а решту передаєш під час `init`:

```hcl
terraform {
  backend "s3" {}
}
```

```hcl
# backend-prod.hcl
bucket       = "acme-terraform-state"
key          = "prod/network/terraform.tfstate"
region       = "eu-central-1"
encrypt      = true
use_lockfile = true
```

```bash
terraform init -backend-config=backend-prod.hcl
# або окремими параметрами:
terraform init -backend-config="key=stage/network/terraform.tfstate"
```

До речі, OpenTofu з версії 1.8 дозволяє використовувати змінні й locals у backend. Це одна з помітних відмінностей між ними.

### 7.6 State locking

Перед кожною операцією, яка працює зі state (`plan` теж бере lock), Terraform захоплює блокування. Якщо його вже тримає хтось інший:

```
│ Error: Error acquiring the state lock
│
│ Lock Info:
│   ID:        6b4c1c1e-5a3b-0f2e-8f2a-3d9b7c1a2e4f
│   Path:      acme-terraform-state/prod/network/terraform.tfstate
│   Operation: OperationTypeApply
│   Who:       runner@gha-runner-7
│   Version:   1.16.6
│   Created:   2026-10-14 09:12:33 +0000 UTC
```

Що робити:

1. **Подивитися на `Who` і `Created`.** Можливо, саме зараз справді йде apply у CI. Тоді просто зачекай (або запускай з `-lock-timeout=10m`, щоб Terraform чекав сам).
2. Якщо процес, який тримав lock, точно помер (CI-джоб убили, ноутбук заснув посеред apply), зняти блокування:

```bash
terraform force-unlock 6b4c1c1e-5a3b-0f2e-8f2a-3d9b7c1a2e4f
```

> **Ніколи не роби `force-unlock`, поки не переконався, що інший процес справді не працює.** Два одночасні apply в один state — найнадійніший спосіб його зіпсувати.

### 7.7 Команди для роботи зі state

```bash
terraform state list                             # усі ресурси в state
terraform state list module.vpc                  # тільки всередині модуля
terraform state show aws_instance.web            # усі атрибути одного ресурсу
terraform state show 'aws_iam_user.this["anna"]' # лапки обов'язкові через [" "]

terraform state mv aws_instance.web aws_instance.app  # змінити адресу (краще — блок moved)
terraform state rm aws_instance.legacy                # "забути" ресурс, НЕ видаляючи його в хмарі

terraform state pull > backup-$(date +%F).tfstate     # завантажити state локально (бекап!)
terraform state push backup.tfstate                   # залити state назад — ОБЕРЕЖНО

terraform show                                   # state у читабельному вигляді
terraform show -json | jq '.values.root_module.resources[].address'
```

> **Правило:** перед будь-якою ручною операцією зі state роби `terraform state pull > backup.tfstate`. Ручне редагування JSON state — крайній захід, коли інших варіантів не залишилося.

### 7.8 Розбиття state і blast radius

Найгірший варіант — **один state на всю компанію**:

- `plan` триває 20 хвилин, бо треба опитати API про кожен із 3000 ресурсів;
- будь-яка помилка може зачепити все одночасно, тобто величезний **blast radius** (радіус ураження);
- поки одна людина робить apply, всі інші чекають на lock;
- кожному, хто хоче змінити DNS-запис, потрібні права на всю інфраструктуру.

State ділять за двома осями:

1. **За середовищами**: `dev`, `stage`, `prod` — завжди окремо. В ідеалі ще й в окремих AWS-акаунтах.
2. **За шарами (компонентами)**: за тим, як часто вони змінюються і хто за них відповідає.

```
  ┌───────────────────────────┐   змінюється часто
  │  apps: ECS, k8s addons    │   (кілька разів на тиждень)
  ├───────────────────────────┤
  │  compute: EKS, ASG, LB    │
  ├───────────────────────────┤
  │  data: RDS, S3, Redis     │   рідко, але помилка = втрата даних
  ├───────────────────────────┤
  │  network: VPC, subnets    │   дуже рідко
  ├───────────────────────────┤
  │  foundation: IAM, DNS     │
  └───────────────────────────┘
```

Наприклад: `prod/network/terraform.tfstate`, `prod/data/terraform.tfstate`, `prod/eks/terraform.tfstate`. Помилка в шарі `apps` фізично не може зачепити базу даних, бо база в іншому state.

### 7.9 Як передавати дані між state

Шару `eks` потрібні ID підмереж із шару `network`. Варіанти:

**1. `terraform_remote_state`** читає outputs іншого state:

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "acme-terraform-state"
    key    = "prod/network/terraform.tfstate"
    region = "eu-central-1"
  }
}

resource "aws_eks_cluster" "main" {
  # ...
  vpc_config {
    subnet_ids = data.terraform_remote_state.network.outputs.private_subnet_ids
  }
}
```

Мінуси: щоб прочитати outputs, потрібен доступ на читання до **всього** state-файлу, разом із секретами в ньому. До того ж шари починають залежати від внутрішньої структури один одного.

**2. Data sources за тегами або іменами**: шукати ресурси просто в хмарі.

```hcl
data "aws_vpc" "main" {
  tags = { Name = "prod" }
}

data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }
  tags = { Tier = "private" }
}
```

**3. Явний «контракт» через SSM Parameter Store** (або Consul, outputs у HCP Terraform): шар network записує `/prod/network/vpc_id` в `aws_ssm_parameter`, а шар eks читає його через `data "aws_ssm_parameter"`. Найменша зв'язність і зрозумілий інтерфейс між командами.

---

## 8. Провайдери, версії і lock-файл {#providers}

### 8.1 Фіксація версій

```hcl
terraform {
  required_version = ">= 1.10, < 2.0" # версія самого Terraform

  required_providers {
    aws = {
      source  = "hashicorp/aws" # = registry.terraform.io/hashicorp/aws
      version = "~> 6.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.9"
    }
  }
}
```

Оператори обмежень версій:

| Запис | Означає | Які версії підходять |
|---|---|---|
| `= 6.2.0` або `6.2.0` | Рівно ця версія | 6.2.0 |
| `>= 6.0` | Ця або новіша | 6.0.0, 6.5.1, 7.0.0... |
| `>= 6.0, < 7.0` | Діапазон | 6.0.0 – 6.x.x |
| `~> 6.0` | Можна збільшувати лише **останню вказану** цифру | 6.0.0 – 6.x.x, але не 7.0 |
| `~> 6.2.0` | Те саме, але останнім вказано patch | 6.2.0 – 6.2.x, але не 6.3 |
| `!= 6.3.0` | Будь-яка, крім цієї | Виключити версію з відомим багом |

> **Підступне питання:** `~> 6.0` і `~> 6.0.0` — дуже різні речі. Перше дозволяє 6.99, друге — тільки 6.0.x.

Best practice:

- у **root-модулях** (тих, що ти запускаєш) для провайдерів пиши `~> X.Y`, а точну версію зафіксує lock-файл;
- у **перевикористовуваних модулях** вказуй лише мінімальну версію (`>= 6.0`), щоб не конфліктувати з вимогами інших модулів. Terraform шукає одну версію провайдера, що задовольняє обмеження **всіх** модулів одночасно.

### 8.2 `.terraform.lock.hcl`

Після `terraform init` з'являється файл з **точною** версією та контрольними сумами кожного провайдера:

```hcl
provider "registry.terraform.io/hashicorp/aws" {
  version     = "6.69.1"
  constraints = "~> 6.0"
  hashes = [
    "h1:Jt0Pr5C3Xq4ZcG...",
    "zh:0a2b4c6d8e0f...",
    # ...
  ]
}
```

- **Комітити обов'язково.** Інакше в тебе на ноутбуці 6.69, а в CI завтра підтягнеться 6.75 з іншою поведінкою.
- `terraform init` бере версію з lock-файлу, навіть якщо вже вийшла новіша.
- `terraform init -upgrade` оновлює провайдери до найновішої версії в межах `constraints` і переписує lock.
- **Lock-файл фіксує тільки провайдери, не модулі.** Версії модулів фіксуються в самому коді (`version = "..."` або `?ref=v1.2.3`). Про це часто питають.

**Різні операційні системи.** Під час `init` Terraform записує `h1:`-хеш лише для твоєї поточної платформи. Якщо в CI інша ОС і там використовується кеш провайдерів або дзеркало, `init` може впасти з помилкою про невідповідність контрольних сум. Рішення — записати хеші для всіх потрібних платформ:

```bash
terraform providers lock \
  -platform=linux_amd64 \
  -platform=darwin_arm64 \
  -platform=darwin_amd64
```

### 8.3 Як оновлювати провайдер

Особливо уважно з **major-версіями** (наприклад, AWS v5 → v6), де є breaking changes:

1. Прочитати **upgrade guide** провайдера: у великих провайдерів він є для кожної major-версії.
2. В окремій гілці підняти `version` і зробити `terraform init -upgrade`.
3. Запустити `plan` **у кожному** state, що використовує цей провайдер. Ідеальний результат — `No changes`. Будь-які зміни треба розібрати й зрозуміти.
4. Виправити deprecated-аргументи і попередження.
5. Котити поступово: dev → stage → prod.

### 8.4 Кеш і дзеркала провайдерів

AWS-провайдер важить сотні мегабайтів, і кожен `init` у кожному каталозі завантажує його заново. Кеш вмикається в `~/.terraformrc` (каталог треба створити заздалегідь):

```hcl
# ~/.terraformrc
plugin_cache_dir = "$HOME/.terraform.d/plugin-cache"
```

```bash
# або змінною оточення
mkdir -p "$HOME/.terraform.d/plugin-cache"
export TF_PLUGIN_CACHE_DIR="$HOME/.terraform.d/plugin-cache"
```

Для закритих мереж без доступу до інтернету роблять дзеркало:

```bash
terraform providers mirror /opt/terraform-mirror   # завантажити провайдери для поточної конфігурації
```

```hcl
# ~/.terraformrc на машинах без інтернету
provider_installation {
  filesystem_mirror {
    path    = "/opt/terraform-mirror"
    include = ["registry.terraform.io/*/*"]
  }
  direct {
    exclude = ["registry.terraform.io/*/*"]
  }
}
```

---

## 9. Модулі {#modules}

### 9.1 Що таке модуль

**Модуль — це будь-який каталог із `.tf`-файлами.** Каталог, у якому ти запускаєш `terraform`, — це **root module**. Модулі, які він викликає через блок `module`, — **child modules**.

Навіщо вони:

- **DRY**: VPC для dev, stage і prod описується один раз.
- **Стандарти**: «правильний S3-бакет у нашій компанії» (шифрування, versioning, заблокований публічний доступ, теги) зашитий у модуль. Команди не можуть «забути галочку».
- **Інкапсуляція**: користувачу модуля не треба знати про 15 ресурсів усередині, йому вистачає 5 змінних.
- **Версіонування**: модуль v1.4 стабільний, у v2.0 є breaking changes, і кожен переходить, коли готовий.

### 9.2 Пишемо свій модуль: безпечний S3-бакет

```
modules/s3-bucket/
├── versions.tf      # required_providers — БЕЗ блоку provider
├── variables.tf
├── main.tf
├── outputs.tf
├── README.md        # генерується terraform-docs
└── examples/
    └── basic/
        └── main.tf
```

```hcl
# modules/s3-bucket/versions.tf
terraform {
  required_version = ">= 1.6"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 6.0" # у модулі — тільки мінімальна версія
    }
  }
}
```

```hcl
# modules/s3-bucket/variables.tf
variable "name" {
  description = "Ім'я бакета (глобально унікальне в AWS)"
  type        = string

  validation {
    condition     = can(regex("^[a-z0-9][a-z0-9.-]{1,61}[a-z0-9]$", var.name))
    error_message = "Ім'я бакета: 3–63 символи, малі літери, цифри, крапки та дефіси."
  }
}

variable "versioning" {
  description = "Чи вмикати versioning"
  type        = bool
  default     = true # безпечне значення за замовчуванням
}

variable "tags" {
  description = "Теги для всіх ресурсів модуля"
  type        = map(string)
  default     = {}
}
```

```hcl
# modules/s3-bucket/main.tf
resource "aws_s3_bucket" "this" {
  bucket = var.name
  tags   = var.tags
}

resource "aws_s3_bucket_versioning" "this" {
  count  = var.versioning ? 1 : 0
  bucket = aws_s3_bucket.this.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket                  = aws_s3_bucket.this.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

```hcl
# modules/s3-bucket/outputs.tf
output "id" {
  description = "Ім'я (ID) бакета"
  value       = aws_s3_bucket.this.id
}

output "arn" {
  description = "ARN бакета"
  value       = aws_s3_bucket.this.arn
}
```

Ресурси всередині модуля названі `this`. Це загальноприйнята конвенція, коли ресурс такого типу в модулі один.

Використання модуля:

```hcl
module "logs_bucket" {
  source = "../../modules/s3-bucket"

  name = "acme-prod-logs"
  tags = local.common_tags
}

module "app_buckets" {
  source   = "../../modules/s3-bucket"
  for_each = toset(["uploads", "exports"]) # модулі теж підтримують for_each і count

  name = "acme-prod-${each.key}"
  tags = local.common_tags
}

output "logs_bucket_arn" {
  value = module.logs_bucket.arn
}
```

Адреси ресурсів: `module.logs_bucket.aws_s3_bucket.this`, `module.app_buckets["uploads"].aws_s3_bucket.this`.

### 9.3 Джерела модулів

Варіанти значення `source` (у справжньому блоці `module` буде лише один із них):

```hcl
# Локальний шлях — модулі в тому самому репозиторії
source = "../../modules/vpc"

# Публічний Terraform Registry — завжди з version!
source  = "terraform-aws-modules/vpc/aws"
version = "~> 6.0"

# Git-репозиторій з тегом; // відокремлює підкаталог у репозиторії
source = "git::https://github.com/acme/terraform-modules.git//modules/vpc?ref=v1.4.2"

# Git по SSH
source = "git::ssh://git@github.com/acme/terraform-modules.git//modules/vpc?ref=v1.4.2"

# Приватний реєстр HCP Terraform
source  = "app.terraform.io/acme/vpc/aws"
version = "1.4.2"
```

> `version` працює тільки для модулів з реєстру. Для Git версію фіксують через `?ref=`: тег або, ще надійніше, SHA коміту (тег можна перемістити, коміт — ні). Модуль без зафіксованої версії — це код, який зміниться під тобою без жодного PR.

Після зміни `source` або `version` потрібен `terraform init` (або `terraform get`).

З Terraform 1.15 у `source` і `version` можна використовувати змінні, але тільки оголошені з `const = true`: їхні значення мають бути відомі вже під час `init`.

```hcl
variable "vpc_module_version" {
  type    = string
  default = "6.7.3"
  const   = true # без цього: "Only literal values and const variables can be evaluated during init"
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = var.vpc_module_version
  # ...
}
```

```bash
terraform init -var vpc_module_version=6.7.2   # init теж приймає -var
```

### 9.4 Готові модулі

Не пиши все з нуля. Для AWS є колекція **terraform-aws-modules** (VPC, EKS, RDS, IAM, S3, ALB...), якою користуються тисячі компаній:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 6.0"

  name = "prod"
  cidr = "10.0.0.0/16"

  azs             = ["eu-central-1a", "eu-central-1b", "eu-central-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true # один NAT дешевший — ок для dev; у prod краще по одному на AZ
}
```

Але зі сторонніми модулями — **довіряй, але перевіряй**. Модуль — це чужий код, який виконується з твоїми обліковими даними. Фіксуй версію, читай CHANGELOG перед оновленням і дивись у `plan`, що модуль насправді створює, особливо якщо це невеликий маловідомий модуль.

### 9.5 Провайдери і модулі

**Правило: у перевикористовуваному модулі не має бути блоку `provider`.** Модуль лише оголошує, які провайдери йому потрібні (`required_providers`), а налаштовані провайдери передає root-модуль. Причини:

- модуль із власним `provider` не можна використовувати з `count`, `for_each` і `depends_on`;
- якщо видалити з коду виклик такого модуля, Terraform не зможе видалити його ресурси: разом із модулем зник і провайдер, через який це треба робити;
- регіон, акаунт і облікові дані — рішення того, хто викликає модуль, а не автора модуля.

Якщо модулю потрібні **два** різні AWS-провайдери (наприклад, основний регіон і `us-east-1` для CloudFront):

```hcl
# modules/cdn/versions.tf
terraform {
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      version               = ">= 6.0"
      configuration_aliases = [aws.us_east_1] # модуль очікує провайдер з таким alias
    }
  }
}
```

```hcl
# root module
module "cdn" {
  source = "./modules/cdn"

  providers = {
    aws           = aws           # провайдер за замовчуванням
    aws.us_east_1 = aws.us_east_1 # alias із root → alias усередині модуля
  }
}
```

### 9.6 Як писати хороші модулі

- **Один модуль — одна відповідальність.** «VPC», «RDS-кластер», «сервіс на ECS». Не «вся інфраструктура компанії».
- **Не роби тонких обгорток.** Модуль, який просто передає 30 змінних в один ресурс, нічого не додає, крім ще одного шару, крізь який доведеться продиратися.
- **Безпечні значення за замовчуванням**: шифрування ввімкнене, публічний доступ вимкнений.
- **Типи, `description` і `validation`** для кожної змінної.
- **Щедрі outputs**: ID, ARN, імена — усе, що може знадобитися користувачам модуля.
- **Не більше 2–3 рівнів вкладеності** модулів, інакше налагодження перетворюється на археологію.
- **Semver і CHANGELOG**: breaking change = нова major-версія.
- **Документація генерується**: `terraform-docs markdown table . > README.md`.
- **Блоки `moved` при рефакторингу** всередині модуля, щоб після оновлення в користувачів нічого не перестворювалося (розділ 11).
- **М'яке видалення змінних і outputs.** З 1.15 у блоках `variable` і `output` є атрибут `deprecated = "повідомлення"`. Користувачі модуля бачать попередження, коли передають значення застарілій змінній або посилаються на застарілий output, і мають час перейти, перш ніж ти видалиш їх у наступній major-версії.

---

## 10. Структура проєкту і середовища {#structure}

Як тримати dev, stage і prod? Є три основні підходи.

### 10.1 CLI workspaces

```bash
terraform workspace new stage
terraform workspace new prod
terraform workspace select prod
terraform workspace show        # prod
terraform workspace list
```

Код один, але для кожного workspace окремий state (у S3 вони лежать під префіксом `env:/stage/...`, `env:/prod/...`). Поточний workspace доступний у коді як `terraform.workspace`:

```hcl
locals {
  instance_type = terraform.workspace == "prod" ? "m6i.large" : "t3.micro"
}
```

Проблеми:

- **Дуже легко помилитися.** `terraform apply` у «не тому» workspace — і ти щойно змінив prod, думаючи, що це dev. Ні каталог, ні код про це не нагадують.
- **Один backend і одні облікові дані** для всіх середовищ. Ізолювати prod в окремому акаунті з окремими правами неможливо.
- Відмінності між середовищами перетворюються на ланцюжки `terraform.workspace == "prod" ? ... : ...` по всьому коду.

HashiCorp сама пише, що CLI workspaces не підходять для ізоляції середовищ, яким потрібні різні облікові дані й контроль доступу. Зате вони добре підходять для **тимчасових однакових копій**: preview-середовища для кожного PR, тестових стендів.

> Не плутай **CLI workspaces** з **workspaces в HCP Terraform**. В HCP Terraform workspace — окрема одиниця з власним state, змінними, правами й історією запусків, тобто фактично окремий root-модуль. Назва однакова, концепції різні.

### 10.2 Каталог на кожне середовище (рекомендовано)

```
infra/
├── modules/                    # перевикористовувані модулі
│   ├── network/
│   ├── eks/
│   └── rds/
└── envs/
    ├── dev/
    │   ├── network/            # окремий root-модуль = окремий state
    │   │   ├── main.tf         #   module "network" { source = "../../../modules/network" ... }
    │   │   ├── backend.tf      #   key = "dev/network/terraform.tfstate"
    │   │   ├── providers.tf
    │   │   └── terraform.tfvars
    │   ├── eks/
    │   └── rds/
    ├── stage/
    │   └── ...
    └── prod/
        ├── network/
        ├── eks/
        └── rds/
```

- Кожен листовий каталог — окремий root-модуль з окремим state.
- Шлях у терміналі сам каже, що саме ти змінюєш: `envs/prod/rds` важко сплутати з `envs/dev/rds`.
- Prod може жити в окремому AWS-акаунті з окремою роллю для apply.
- Відмінності між середовищами описані в `terraform.tfvars`, а не в умовах по всьому коду.

Мінус — повторюваний boilerplate: `backend.tf`, `providers.tf` і `versions.tf` майже однакові в кожному каталозі. Саме цю проблему вирішує Terragrunt.

### 10.3 Terragrunt

Terragrunt — обгортка над Terraform/OpenTofu від Gruntwork. Головне, що він дає: генерацію backend-конфігурації (DRY), явні залежності між state і запуск багатьох модулів однією командою.

```hcl
# infra/root.hcl — спільне для всіх
remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    bucket       = "acme-terraform-state"
    key          = "${path_relative_to_include()}/terraform.tfstate" # envs/prod/eks/terraform.tfstate
    region       = "eu-central-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

```hcl
# infra/envs/prod/eks/terragrunt.hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

terraform {
  source = "../../../modules/eks"
}

dependency "network" {
  config_path = "../network" # беремо outputs із шару network
}

inputs = {
  cluster_name = "prod"
  vpc_id       = dependency.network.outputs.vpc_id
  subnet_ids   = dependency.network.outputs.private_subnet_ids
}
```

```bash
cd infra/envs/prod/eks && terragrunt plan       # один модуль
cd infra/envs/prod && terragrunt run --all plan  # усі модулі середовища в порядку залежностей
                                                 # (у старих версіях: terragrunt run-all plan)
```

### 10.4 Що обрати

| Ситуація | Підхід |
|---|---|
| Пет-проєкт, одне середовище | Один каталог, локальний або S3 state |
| Невелика команда, 2–3 середовища | Каталоги на середовище + модулі в тому самому репозиторії |
| Однакові тимчасові копії (preview на кожен PR) | CLI workspaces |
| Багато акаунтів, десятки state, залежності між шарами | Terragrunt, HCP Terraform (зокрема Stacks), Spacelift, env0 |
| Організація з багатьма командами | Модулі в окремих репозиторіях із semver + приватний реєстр + окремі акаунти на команди і середовища |

---

## 11. Рефакторинг та імпорт: moved, import, removed {#refactoring}

Колись усю роботу зі state робили командами `terraform state mv`, `terraform import`, `terraform state rm`. Їхня проблема в тому, що вони **одразу змінюють state**, оминаючи plan, code review і CI. Помилився в адресі — дізнаєшся вже потім. Сучасний підхід — **блоки в коді**, які проходять через `plan` і PR, як будь-яка інша зміна.

| Задача | Старий спосіб (одразу змінює state) | Сучасний спосіб (у коді, через plan) |
|---|---|---|
| Перейменувати або перемістити ресурс | `terraform state mv` | Блок `moved` (1.1+) |
| Взяти існуючий ресурс під керування | `terraform import` | Блок `import` (1.5+) |
| Перестати керувати, не видаляючи | `terraform state rm` | Блок `removed` (1.7+) |
| Примусово перестворити | `terraform taint` (deprecated) | `terraform apply -replace=ADDR` |
| Прийняти зміни з реальності у state | `terraform refresh` (deprecated) | `terraform apply -refresh-only` |

### 11.1 moved: перейменування без перестворення

Ти перейменував `aws_instance.web` на `aws_instance.app`. Без підказки Terraform побачить, що `web` зник із коду (видалити), а `app` з'явився (створити). Тобто **знищить сервер і створить новий**.

```hcl
resource "aws_instance" "app" { # було "web"
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
}

moved {
  from = aws_instance.web
  to   = aws_instance.app
}
```

```
  # aws_instance.web has moved to aws_instance.app
    resource "aws_instance" "app" {
        id = "i-0abc123def456"
        # (30 unchanged attributes hidden)
    }

Plan: 0 to add, 0 to change, 0 to destroy.
```

`moved` працює і для складніших випадків:

```hcl
# Міграція з count на for_each
moved {
  from = aws_iam_user.this[0]
  to   = aws_iam_user.this["anna"]
}

# Винесення ресурсу в модуль
moved {
  from = aws_s3_bucket.logs
  to   = module.logs_bucket.aws_s3_bucket.this
}

# Перейменування виклику модуля: переїжджають усі ресурси всередині
moved {
  from = module.web_server
  to   = module.app_server
}
```

Блоки `moved` можна видалити, коли вони застосовані в усіх state. У модулях, якими користуються інші команди, їх залишають надовго: ти не знаєш, коли оновиться кожен споживач.

### 11.2 import: беремо під керування існуючу інфраструктуру

Найчастіша реальна задача: «у нас усе створено руками, треба перевести на Terraform».

```hcl
import {
  to = aws_s3_bucket.legacy
  id = "acme-legacy-bucket" # формат ID для кожного ресурсу описаний у документації провайдера
}
```

Писати код ресурсу руками не обов'язково, Terraform може згенерувати його сам:

```bash
terraform plan -generate-config-out=generated.tf
```

Terraform прочитає реальний ресурс і запише його опис у `generated.tf`. Далі:

1. Переглянути згенерований код: прибрати атрибути зі значеннями за замовчуванням, замінити захардкоджені ID посиланнями на інші ресурси.
2. Перенести код у потрібний файл.
3. `terraform plan` має показати `1 to import, 0 to add, 0 to change, 0 to destroy`. Якщо є зміни, код ще не відповідає реальності — доводь до нуля.
4. `terraform apply` записує ресурс у state.
5. Блок `import` після цього можна видалити.

Масовий імпорт через `for_each` (1.7+):

```hcl
locals {
  legacy_buckets = toset(["acme-logs", "acme-backups", "acme-exports"])
}

import {
  for_each = local.legacy_buckets
  to       = aws_s3_bucket.legacy[each.key]
  id       = each.key
}

resource "aws_s3_bucket" "legacy" {
  for_each = local.legacy_buckets
  bucket   = each.key
}
```

> Чому блок `import`, а не команда `terraform import`? Команда одразу пише в state, не показує план і не проходить review. Блок — звичайна зміна коду: колеги бачать у PR, що саме імпортується, а CI показує план. До того ж команда не вміє генерувати код.

**Пошук ресурсів для імпорту: `terraform query` (1.14+).** Коли ресурсів сотні, виписувати `import`-блок для кожного ID довго. Тепер ресурси можна знайти запитом: у файлі `*.tfquery.hcl` описується блок `list`.

```hcl
# legacy.tfquery.hcl
list "aws_instance" "platform" {
  provider = aws

  config {
    filter {
      name   = "tag:Team"
      values = ["platform"]
    }
  }
}
```

```bash
terraform query                                    # показати знайдені ресурси
terraform query -generate-config-out=imported.tf   # згенерувати для них import- і resource-блоки
```

Далі все так само: почистити згенерований код, довести план до нуля, застосувати. Які типи ресурсів підтримують `list`, залежить від провайдера (в AWS-провайдері їх уже понад 250).

І ще два уточнення: з 1.12 `import` може знайти ресурс через `identity` (структурований ідентифікатор) замість рядка `id`, якщо провайдер це підтримує, а з 1.16 блоки `import` можна писати не лише в root-модулі, а й усередині модулів.

### 11.3 removed: забути ресурс, не видаляючи його

Ресурс більше не повинен керуватися цим кодом (наприклад, переїжджає в state іншої команди), але видаляти його в хмарі не можна:

```hcl
removed {
  from = aws_instance.legacy

  lifecycle {
    destroy = false # прибрати зі state, а ресурс у хмарі залишити
  }
}
```

Блок ресурсу `aws_instance.legacy` прибираєш із коду, а `removed` додаєш. План покаже `will no longer be managed by Terraform`, і нічого не буде видалено.

З Terraform 1.16 те саме можна вказати прямо в `lifecycle` ресурсу:

```hcl
resource "aws_iam_user" "this" {
  for_each = toset(var.users)
  name     = each.key

  lifecycle {
    destroy = false # прибрані зі списку і навіть terraform destroy → забути, а не видалити
  }
}
```

Тепер, якщо прибрати когось зі списку або навіть запустити `terraform destroy`, план покаже `will no longer be managed by Terraform, but will not be destroyed`. Але якщо видалити з коду весь блок ресурсу, разом із ним зникне і `destroy = false`, і ресурси буде знищено, так само як із `prevent_destroy`.

### 11.4 Примусове перестворення і refresh-only

```bash
# Інстанс "зламався" зсередини — перестворити лише його
terraform apply -replace="aws_instance.web"

# Подивитися drift: що змінилося в реальності порівняно зі state
terraform plan -refresh-only

# Прийняти реальність: оновити state, нічого не змінюючи в інфраструктурі
terraform apply -refresh-only
```

> `apply -refresh-only` не змінює ресурси, але змінює state. Якщо ресурс видалили руками, після refresh-only він зникне зі state, і наступний звичайний apply створить його знову. Розумій, що саме ти приймаєш.

---

## 12. Секрети і безпека {#security}

### 12.1 Де НЕ можна тримати секрети

- У `.tf`-файлах і в `*.tfvars`, що лежать у Git: вони залишаться в історії назавжди.
- У `default` змінних.
- В outputs без `sensitive = true`.

### 12.2 Що насправді робить `sensitive = true`

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

output "db_password" {
  value     = random_password.db.result
  sensitive = true
}
```

`sensitive` ховає значення з виводу `plan` і `apply` (там буде `(sensitive value)`) і не дає випадково вивести його в незахищений output. **І все.**

- У **state** значення лежить відкритим текстом.
- `terraform output db_password` (з конкретним іменем) і `terraform output -json` показують його відкритим текстом.
- У збереженому plan-файлі (`-out=tfplan`) воно теж є.

Тобто `sensitive` захищає від випадкового підглядання в логах CI, але це не шифрування.

### 12.3 Як правильно працювати із секретами

**Варіант 1: нехай секретом керує сервіс, а не Terraform.** Найпростіше і найнадійніше:

```hcl
resource "aws_db_instance" "main" {
  identifier                  = "prod-db"
  engine                      = "postgres"
  instance_class              = "db.t4g.medium"
  allocated_storage           = 50
  username                    = "app"
  manage_master_user_password = true # пароль генерує, зберігає в Secrets Manager і ротує сам RDS

  deletion_protection       = true
  skip_final_snapshot       = false
  final_snapshot_identifier = "prod-db-final"
}
```

Terraform пароля не бачить, і в state його немає. Застосунок читає пароль із Secrets Manager.

**Варіант 2: ephemeral-ресурси і write-only атрибути** (Terraform 1.10+ і 1.11+). Ephemeral-ресурс існує лише під час виконання команди і **ніколи не записується ні в state, ні в plan**. Write-only атрибут (із суфіксом `_wo`) приймає значення, передає його в API і теж ніде не зберігає:

```hcl
# Читаємо секрет, не записуючи його в state
ephemeral "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/db/master-password"
}

resource "aws_db_instance" "main" {
  # ...
  password_wo         = ephemeral.aws_secretsmanager_secret_version.db.secret_string
  password_wo_version = 1 # Terraform не бачить старого значення, тож щоб змінити пароль, збільш версію
}
```

Ephemeral-значення можна використовувати лише там, де це підтримується: у write-only атрибутах, конфігурації провайдерів, інших ephemeral-ресурсах, `locals`. Наявність `_wo`-атрибутів залежить від провайдера і ресурсу, тож дивись документацію. Вхідну змінну теж можна оголосити з `ephemeral = true` (1.10+): тоді її значення не потрапить ні в plan, ні в state, а Terraform не дасть використати її там, де значення довелося б зберегти.

**Варіант 3: секрети ззовні через змінні оточення:** `export TF_VAR_db_password=...` із secret store твого CI або з Vault. У Git секрет не потрапить, але якщо значення йде у звичайний (не write-only) атрибут, воно все одно опиниться в state.

**Варіант 4: Vault із динамічними обліковими даними.** Terraform отримує короткоживучі облікові дані до хмари з Vault (або через OIDC, див. 12.5), і довгоживучих ключів не зберігається ніде.

### 12.4 Захист самого state

- Backend із шифруванням at rest (SSE-KMS для S3) і TLS.
- Доступ до бакета зі state — **мінімальному колу**: ролям CI і кільком адміністраторам. Хто може читати state, той читає всі секрети з нього.
- Versioning увімкнений, щоб можна було відкотитися.
- Аудит доступу (CloudTrail data events або S3 access logs).
- OpenTofu вміє шифрувати state на боці клієнта (з версії 1.7), і тоді навіть доступ до бакета не розкриває вмісту.

### 12.5 Облікові дані для Terraform

- **Ніяких довгоживучих access keys у CI.** Використовуй **OIDC**: GitHub Actions чи GitLab CI отримують короткоживучий токен і через `AssumeRoleWithWebIdentity` міняють його на тимчасові облікові дані AWS.
- **Дві ролі.** `terraform-plan` з read-only правами на ресурси (плюс доступ до state і lock-файлу) — для PR. `terraform-apply` з правами на зміни — тільки для захищеної гілки і з ручним підтвердженням.
- Prod-роль може прийняти лише prod-середовище CI: умова на `sub` у trust policy, наприклад `repo:acme/infra:environment:prod`.

### 12.6 Ланцюжок постачання (supply chain)

- Провайдери з реєстру перевіряються за підписами і хешами з lock-файлу, тому **комітити lock обов'язково**.
- Сторонні модулі — це код, що виконується з твоїми обліковими даними. Модуль може містити `local-exec` чи data source `external`, які запустять будь-яку команду на машині, де працює Terraform.
- **`terraform plan` не є безпечною read-only операцією.** Під час plan виконуються data sources (зокрема `external`, що запускає довільну програму) і ініціалізуються провайдери. Тому не можна запускати plan із prod-обліковими даними на PR із форків чи від неперевірених авторів: шкідливий PR може вкрасти облікові дані ще до будь-якого review.

### 12.7 Статичний аналіз і policy as code

| Інструмент | Що робить |
|---|---|
| `tflint` | Лінтер: неіснуючі типи інстансів, deprecated-синтаксис, конвенції іменування; є плагіни для AWS/GCP/Azure |
| `trivy config` (тепер у ньому живе tfsec) | Помилки безпеки: відкриті security groups, нешифровані диски, публічні бакети |
| `checkov` | Те саме, сотні готових політик, уміє перевіряти й plan у JSON |
| `conftest` (OPA/Rego) | Власні політики компанії, які перевіряють plan |
| Sentinel / OPA в HCP Terraform | Політики, вбудовані в платформу запуску |

Приклад власної політики, яка забороняє видаляти бази даних через CI:

```rego
# policy/deny_db_delete.rego
package main

import rego.v1

deny contains msg if {
  some rc in input.resource_changes
  rc.type in {"aws_db_instance", "aws_rds_cluster"}
  "delete" in rc.change.actions
  msg := sprintf("Видалення бази заборонене політикою: %s", [rc.address])
}
```

```bash
terraform plan -out=tfplan
terraform show -json tfplan > tfplan.json
conftest test tfplan.json        # exit code != 0, якщо є порушення
```

Заміна (`-/+`) — це теж `delete` у списку дій, тож політика зловить і перестворення бази.

---

## 13. Terraform у CI/CD {#cicd}

Загальні принципи пайплайнів розібрані в статті [CI/CD: guide](/blog/ci-cd/). Тут — те, що специфічно для Terraform.

### 13.1 Чому apply має робити CI, а не ноутбук

- Одна точка запуску з однаковими версіями Terraform і провайдерів.
- Кожна зміна проходить через PR з видимим планом.
- Облікові дані з правом змінювати prod не лежать на ноутбуках.
- Повна історія: хто, коли і що застосував.

### 13.2 Типовий pipeline

```
 feature-гілка ──PR──▶ fmt -check ─▶ validate ─▶ tflint ─▶ trivy/checkov ─▶ plan
                                                                             │
                                                   план публікується в PR ◀──┘
                                                              │
                                                     review + approve
                                                              │
 main ◀─────────────────────── merge ─────────────────────────┘
   │
   └──▶ plan -out=tfplan ─▶ ручне підтвердження (environment: prod) ─▶ apply tfplan
```

Ключова ідея: **apply застосовує саме той план, який переглянули**. `terraform apply tfplan` не перераховує план заново. Якщо за цей час state змінився (хтось інший щось застосував), Terraform відмовиться застосовувати застарілий план (`Saved plan is stale`), і це правильно.

### 13.3 Приклад: GitHub Actions з OIDC

```yaml
# .github/workflows/terraform-prod-network.yml
name: terraform / prod / network

on:
  pull_request:
    paths: ["infra/envs/prod/network/**", "infra/modules/**"]
  push:
    branches: [main]
    paths: ["infra/envs/prod/network/**", "infra/modules/**"]

permissions:
  id-token: write        # потрібно для OIDC
  contents: read

concurrency:
  group: tf-prod-network     # не більше одного запуску на цей state одночасно
  cancel-in-progress: false  # НІКОЛИ не переривати apply посередині

env:
  TF_IN_AUTOMATION: "true"   # вивід без підказок на кшталт "тепер запусти terraform apply"
  TF_INPUT: "false"          # не чекати інтерактивного вводу
  WORKDIR: infra/envs/prod/network

jobs:
  plan:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ${{ env.WORKDIR }}
    steps:
      - uses: actions/checkout@v7

      - uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: 1.16.6     # точна версія, однакова для всіх
          terraform_wrapper: false

      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::123456789012:role/gha-terraform-plan   # read-only роль
          aws-region: eu-central-1

      - run: terraform fmt -check -recursive
      - run: terraform init
      - run: terraform validate
      - run: terraform plan -out=tfplan -lock-timeout=5m

      - name: Plan у summary джоба (sensitive-значення замасковані)
        run: |
          {
            echo '```'
            terraform show -no-color tfplan
            echo '```'
          } >> "$GITHUB_STEP_SUMMARY"

      - uses: actions/upload-artifact@v7
        with:
          name: tfplan
          path: ${{ env.WORKDIR }}/tfplan
          retention-days: 1          # у плані можуть бути секрети, тож не зберігаємо довго

  apply:
    needs: plan
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: prod                # GitHub Environment з обов'язковим approve
    defaults:
      run:
        working-directory: ${{ env.WORKDIR }}
    steps:
      - uses: actions/checkout@v7

      - uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: 1.16.6
          terraform_wrapper: false

      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::123456789012:role/gha-terraform-apply  # роль із правом змін
          aws-region: eu-central-1

      - uses: actions/download-artifact@v8
        with:
          name: tfplan
          path: ${{ env.WORKDIR }}

      - run: terraform init
      - run: terraform apply tfplan     # рівно той план, що згенерований вище
```

Публікацію плану коментарем у PR зазвичай додають окремим кроком (через `actions/github-script` чи tfcmt) або беруть інструмент, який уміє це з коробки (див. нижче).

### 13.4 Готові інструменти

| Інструмент | Що це |
|---|---|
| **Atlantis** | Self-hosted бот для PR: коментарі `atlantis plan` / `atlantis apply`, блокування каталогу на час PR |
| **HCP Terraform / Terraform Enterprise** | Платформа HashiCorp: remote runs, state, політики Sentinel/OPA, приватний реєстр модулів, drift detection |
| **Spacelift, env0, Scalr** | Комерційні платформи з підтримкою Terraform, OpenTofu і Terragrunt, політиками і залежностями між стеками |
| **Digger, Terrateam** | Запуск у твоєму власному CI (GitHub Actions) з PR-коментарями, як в Atlantis |

### 13.5 Drift detection за розкладом

```bash
terraform plan -detailed-exitcode -lock=false -input=false
echo $?
# 0 — змін немає
# 1 — помилка
# 2 — є зміни: drift або незастосований код
```

Нічний джоб у CI, який проходить по всіх state і надсилає в Slack список тих, де код виходу 2, — дешевий і дуже корисний. `-lock=false` тут допустимий, бо plan нічого не записує в state, а нічний запуск не блокуватиме нікому роботу.

### 13.6 Правила для CI

- Версія Terraform зафіксована і збігається з `required_version`.
- `-input=false` скрізь.
- `terraform apply tfplan` не питає підтвердження: підтвердженням є review плану. `apply -auto-approve` без збереженого плану для prod неприпустимий.
- `concurrency` або черга: не більше одного apply на state одночасно.
- Apply не можна переривати. Обірваний apply залишає state заблокованим, а частину ресурсів — у проміжному стані.
- Plan-файли містять секрети: зберігай їх недовго і з обмеженим доступом.

---

## 14. Тестування Terraform-коду {#testing}

Рівні перевірок — від дешевих і швидких до дорогих і повільних:

| Рівень | Інструменти | Що ловить | Час |
|---|---|---|---|
| 1. Форматування й синтаксис | `terraform fmt -check`, `terraform validate` | Синтаксис, типи, неіснуючі атрибути | Секунди |
| 2. Лінтинг | `tflint` | Неіснуючі типи інстансів, невикористані змінні, конвенції | Секунди |
| 3. Безпека | `trivy config`, `checkov` | Відкриті порти, нешифровані диски, публічні бакети | Секунди |
| 4. Unit-тести модулів | `terraform test` на plan з `mock_provider` | Логіку модуля: умови, `for_each`, validation | Секунди |
| 5. Політики на plan | `conftest`, Sentinel | Порушення правил компанії в конкретних змінах | Секунди |
| 6. Інтеграційні тести | `terraform test` з `command = apply`, Terratest | Чи справді працює створена інфраструктура | Хвилини, і це коштує грошей |
| 7. Вартість | Infracost | Скільки коштуватиме зміна | Секунди |

### 14.1 `terraform validate` і `tflint`

`validate` перевіряє синтаксис і відповідність схемі провайдера, але не знає, що `t3.mikro` — неіснуючий тип інстансу. Це вже робота tflint із плагіном для провайдера:

```hcl
# .tflint.hcl
plugin "terraform" {
  enabled = true
  preset  = "recommended"
}

plugin "aws" {
  enabled = true
  version = "0.49.0" # перевір актуальну версію плагіна
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}
```

```bash
tflint --init        # завантажити плагіни
tflint --recursive
```

### 14.2 `terraform test` (1.6+)

Тести пишуться на HCL у файлах `*.tftest.hcl` (у каталозі модуля або в `tests/`). Тест для модуля S3-бакета з розділу 9:

```hcl
# modules/s3-bucket/tests/s3_bucket.tftest.hcl

# Мок-провайдер: тести не ходять в AWS і не потребують облікових даних (1.7+)
mock_provider "aws" {}

variables {
  name = "acme-test-bucket"
}

run "bucket_uses_given_name" {
  command = plan

  assert {
    condition     = aws_s3_bucket.this.bucket == "acme-test-bucket"
    error_message = "Ім'я бакета не збігається зі змінною name"
  }
}

run "public_access_blocked" {
  command = plan

  assert {
    condition     = aws_s3_bucket_public_access_block.this.restrict_public_buckets
    error_message = "Публічний доступ має бути заблокований"
  }
}

run "versioning_can_be_disabled" {
  command = plan

  variables {
    versioning = false
  }

  assert {
    condition     = length(aws_s3_bucket_versioning.this) == 0
    error_message = "При versioning = false ресурс versioning не має створюватися"
  }
}

run "rejects_invalid_name" {
  command = plan

  variables {
    name = "Acme_Bucket"
  }

  expect_failures = [var.name] # тест проходить, якщо validation змінної впала
}
```

```bash
cd modules/s3-bucket
terraform init
terraform test
# tests/s3_bucket.tftest.hcl... in progress
#   run "bucket_uses_given_name"... pass
#   run "public_access_blocked"... pass
#   run "versioning_can_be_disabled"... pass
#   run "rejects_invalid_name"... pass
# tests/s3_bucket.tftest.hcl... tearing down
# tests/s3_bucket.tftest.hcl... pass
#
# Success! 4 passed, 0 failed.
```

Важлива деталь: з `command = apply` (це значення за замовчуванням, якщо `command` не вказано) Terraform **справді створює ресурси**, перевіряє assert-и, а наприкінці все знищує. Це вже інтеграційний тест, і запускати його треба в окремому акаунті-пісочниці.

### 14.3 Terratest

Terratest — Go-бібліотека від Gruntwork для інтеграційних тестів: `terraform apply` → перевірити, що сервіс справді відповідає по HTTP або що файл читається з бакета → `terraform destroy`. Потужно, але вимагає знання Go і коштує грошей і часу, тому зазвичай нею покривають лише ключові модулі.

```go
func TestS3Bucket(t *testing.T) {
	opts := &terraform.Options{
		TerraformDir: "../examples/basic",
		Vars: map[string]interface{}{
			"name": fmt.Sprintf("acme-test-%s", strings.ToLower(random.UniqueId())),
		},
	}
	defer terraform.Destroy(t, opts) // прибрати за собою, навіть якщо тест упав
	terraform.InitAndApply(t, opts)

	bucketID := terraform.Output(t, opts, "id")
	aws.AssertS3BucketExists(t, "eu-central-1", bucketID)
}
```

### 14.4 pre-commit

Щоб базові перевірки запускалися ще до коміту:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.109.2
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint
      - id: terraform_docs
      - id: terraform_trivy
```

### 14.5 Вартість змін

```bash
infracost breakdown --path . --format json --out-file base.json   # вартість до змін (на main)
infracost diff --path . --compare-to base.json                    # різниця після змін у гілці
```

Infracost може писати в PR коментар на кшталт «ця зміна додасть +$420/міс». Іноді це найкорисніший рядок у всьому review.

---

## 15. Best practices — зведений список {#best-practices}

### State

1. **Remote backend із блокуванням** — завжди, коли з кодом працює більше ніж одна людина або CI.
2. **Versioning і шифрування** бакета зі state, доступ — мінімальному колу.
3. **Окремий state на кожне середовище і кожен шар.** Маленький state = маленький blast radius, швидкий plan, менше конфліктів за lock.
4. **Жодних ручних правок state.** Тільки `moved`/`import`/`removed`; команди `state` — у крайньому разі і після `state pull` у бекап.

### Код

5. **Фіксуй версії**: `required_version`, провайдери через `~>`, модулі — точною версією або `?ref=`. Коміть `.terraform.lock.hcl`.
6. **`for_each` замість `count`** для всього, що має унікальну ідентичність. `count` — тільки для `0/1` і повністю однакових копій.
7. **Типи, `description` і `validation`** для змінних. Не роби змінною те, що не повинно змінюватися: для цього є `locals`.
8. **Іменування**: snake_case; не повторюй тип у назві (`aws_s3_bucket.logs`, а не `aws_s3_bucket.logs_bucket`); `this` — для єдиного ресурсу свого типу в модулі.
9. **Порядок у блоці**: спершу `count`/`for_each`, далі звичайні аргументи, потім вкладені блоки, наприкінці `lifecycle` і `depends_on`.
10. **`terraform fmt`** — завжди, а в CI — перевірка `fmt -check`.
11. **`default_tags` у провайдері**: `Environment`, `Project`, `Owner`, `ManagedBy = "terraform"`. Тег `ManagedBy` одразу підказує людині в консолі: «руками не чіпати».
12. **Не хардкодь** ID акаунтів, регіони, AMI: бери їх із data sources і змінних.
13. **Мінімум `depends_on`, `-target` і provisioners.** Їхня поява — сигнал, що зі структурою щось не так.

```hcl
provider "aws" {
  region = var.region

  default_tags {
    tags = {
      Environment = var.environment
      Project     = "shop"
      Owner       = "platform-team"
      ManagedBy   = "terraform"
    }
  }
}
```

### Модулі

14. Модуль = одна відповідальність, безпечні значення за замовчуванням, жодних блоків `provider`, outputs і документація.
15. Semver, CHANGELOG, `moved` при рефакторингу всередині модуля.
16. Сторонні модулі — лише з фіксованою версією і після перевірки коду.

### Безпека

17. Жодних секретів у Git. Секретами краще нехай керує сам сервіс (RDS managed password, Secrets Manager); ephemeral і write-only — там, де вони підтримуються.
18. OIDC замість статичних ключів; окремі ролі для plan і для apply.
19. `prevent_destroy` + захист на боці хмари (`deletion_protection`) для баз, бакетів зі state, KMS-ключів.
20. Сканери (`trivy`, `checkov`) і політики (`conftest`) у CI.

### Процес

21. **Apply — тільки з CI** і тільки зі збереженого та переглянутого плану.
22. **Читай plan повністю.** Шукай `destroy`, `must be replaced`, `forces replacement`.
23. **Drift detection** за розкладом.
24. **Доступ людей до prod-консолі — read-only.** Якщо міняти руками можуть усі, Terraform-код дуже швидко перестає бути правдою.
25. **Оновлюй Terraform і провайдери регулярно і невеликими кроками.** Стрибок через три major-версії — це вже окремий проєкт.
26. **Terraform — для інфраструктури.** Деплой застосунків — Helm/ArgoCD/CI, конфігурація всередині VM — Ansible/cloud-init/Packer.

---

## 16. Команди, які треба знати напам'ять {#commands}

```bash
# ── Базовий цикл ─────────────────────────────────────────────────────
terraform init                          # провайдери, модулі, backend
terraform init -upgrade                 # оновити провайдери/модулі в межах обмежень
terraform init -reconfigure             # змінився backend: не переносити state
terraform init -migrate-state           # змінився backend: перенести state у новий
terraform fmt -recursive                # відформатувати весь код
terraform validate                      # синтаксис і типи
terraform plan                          # що зміниться
terraform plan -out=tfplan              # зберегти план у файл
terraform apply tfplan                  # застосувати рівно цей план
terraform apply -auto-approve           # без підтвердження — тільки для пісочниць
terraform plan -destroy                 # подивитися, що знесе destroy
terraform destroy                       # знести все в цьому state

# ── Параметри plan/apply ─────────────────────────────────────────────
terraform plan -var-file=prod.tfvars -var="instance_type=t3.large"
terraform plan -target=module.vpc       # тільки частина графа — для аварійних ситуацій
terraform apply -replace="aws_instance.web"
terraform plan -refresh-only            # тільки drift
terraform plan -refresh=false           # не опитувати хмару (швидше, але наосліп)
terraform plan -parallelism=3           # менше одночасних запитів (rate limits API)
terraform plan -lock-timeout=10m        # чекати на lock, а не падати одразу
terraform plan -detailed-exitcode       # 0 — без змін, 1 — помилка, 2 — є зміни

# ── State ────────────────────────────────────────────────────────────
terraform state list
terraform state show 'module.vpc.aws_subnet.private["a"]'
terraform state mv SRC DST
terraform state rm ADDR
terraform state pull > backup.tfstate
terraform force-unlock LOCK_ID

# ── Інспекція ────────────────────────────────────────────────────────
terraform output -json
terraform show tfplan
terraform show -json tfplan | jq -r '.resource_changes[] | "\(.change.actions | join(",")) \(.address)"'
terraform graph | dot -Tsvg > graph.svg
terraform graph -format=mermaid         # граф у Mermaid (1.16+): рендериться просто в GitHub/GitLab
terraform console
terraform providers                     # дерево провайдерів по модулях
terraform version

# ── Імпорт ───────────────────────────────────────────────────────────
terraform plan -generate-config-out=generated.tf    # код для ресурсів з import-блоків
terraform query -generate-config-out=imported.tf    # знайти ресурси list-блоками і згенерувати import (1.14+)

# ── Workspaces ───────────────────────────────────────────────────────
terraform workspace list
terraform workspace new stage
terraform workspace select prod
terraform workspace show

# ── Налагодження ─────────────────────────────────────────────────────
TF_LOG=DEBUG terraform plan 2> debug.log                  # детальний лог іде в stderr
TF_LOG_PROVIDER=TRACE TF_LOG_PATH=tf.log terraform apply  # лише провайдер, одразу у файл
```

Змінні оточення, які варто знати:

| Змінна | Для чого |
|---|---|
| `TF_VAR_<name>` | Значення змінної `<name>` |
| `TF_LOG`, `TF_LOG_PATH`, `TF_LOG_CORE`, `TF_LOG_PROVIDER` | Логування: `TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR` |
| `TF_INPUT=false` | Не питати нічого інтерактивно |
| `TF_IN_AUTOMATION=true` | Вивід, пристосований для CI |
| `TF_CLI_ARGS_plan="-parallelism=5"` | Додаткові аргументи для конкретної команди |
| `TF_PLUGIN_CACHE_DIR` | Кеш провайдерів |
| `TF_DATA_DIR` | Інший шлях замість `.terraform` |
| `TF_WORKSPACE` | Обрати workspace без `workspace select` |

---

## 17. Траблшутінг: типові помилки {#troubleshooting}

### 17.1 `Error acquiring the state lock`

Хтось (або щось) тримає lock. Детально розібрано в розділі 7.6: подивись на `Who` і `Created`, зачекай або використай `-lock-timeout`, і тільки якщо процес точно мертвий — `terraform force-unlock <ID>`.

### 17.2 State зіпсований або видалений

Якщо в бакеті ввімкнений versioning, відновлюємо попередню версію:

```bash
aws s3api list-object-versions \
  --bucket acme-terraform-state \
  --prefix prod/network/terraform.tfstate \
  --query 'Versions[].{id:VersionId,date:LastModified,size:Size}'

aws s3api get-object \
  --bucket acme-terraform-state \
  --key prod/network/terraform.tfstate \
  --version-id <VERSION_ID> restored.tfstate

terraform state push restored.tfstate   # якщо в backend serial більший, Terraform вимагатиме -force
terraform plan                          # переконатися, що state відповідає реальності
```

Якщо versioning не було, залишається **заново імпортувати** всі ресурси блоками `import` (і більше ніколи не тримати state без versioning).

### 17.3 `Invalid for_each argument`

```
│ Error: Invalid for_each argument
│
│ The "for_each" set includes values derived from resource attributes that
│ cannot be determined until apply, and so Terraform cannot determine the full
│ set of keys that will identify the instances of this resource.
```

Причина: ключі `for_each` залежать від значень, які з'являться тільки після apply. Рішення — будувати ключі зі статичних даних, а невідомі значення класти у **значення** map:

```hcl
# ПОГАНО: ключі — ID підмереж, яких іще не існує
resource "aws_route_table_association" "private" {
  for_each       = toset(aws_subnet.private[*].id)
  subnet_id      = each.value
  route_table_id = aws_route_table.private.id
}

# ДОБРЕ: aws_subnet.private створено через for_each по var.subnets,
# тож ключі ("a", "b") відомі одразу, а ID лежать у значеннях
resource "aws_route_table_association" "private" {
  for_each       = aws_subnet.private
  subnet_id      = each.value.id
  route_table_id = aws_route_table.private.id
}
```

Аварійний обхідний шлях — спершу `terraform apply -target=aws_subnet.private`, потім повний apply. Але це милиця, а не рішення.

### 17.4 `Error: Cycle`

```
│ Error: Cycle: aws_security_group.app, aws_security_group.db
```

Класика — дві security groups з inline-правилами, які посилаються одна на одну. Рішення — винести правила в окремі ресурси:

```hcl
resource "aws_security_group" "app" {
  name   = "app"
  vpc_id = aws_vpc.main.id
}

resource "aws_security_group" "db" {
  name   = "db"
  vpc_id = aws_vpc.main.id
}

resource "aws_vpc_security_group_ingress_rule" "db_from_app" {
  security_group_id            = aws_security_group.db.id
  referenced_security_group_id = aws_security_group.app.id
  from_port                    = 5432
  to_port                      = 5432
  ip_protocol                  = "tcp"
}

resource "aws_vpc_security_group_egress_rule" "app_to_db" {
  security_group_id            = aws_security_group.app.id
  referenced_security_group_id = aws_security_group.db.id
  from_port                    = 5432
  to_port                      = 5432
  ip_protocol                  = "tcp"
}
```

### 17.5 Plan постійно показує зміни (perpetual diff)

Застосував, а наступний plan знову показує ті самі зміни. Типові причини:

- **API нормалізує значення.** Ти пишеш JSON-політику з іншими пробілами чи порядком ключів, AWS повертає її у своєму форматі, і Terraform бачить різницю. Рішення: `jsonencode()` або data source `aws_iam_policy_document` замість сирого JSON-рядка.
- **Ресурс змінює хтось інший**: autoscaler, оператор Kubernetes, інша система, що додає теги. Рішення: `ignore_changes` для цих атрибутів.
- **Двоє керують одним і тим самим**: наприклад, inline-правила в `aws_security_group` і окремі ресурси `aws_vpc_security_group_ingress_rule` для тієї самої групи. Вони постійно перезаписують одне одного. Обери щось одне.
- **`depends_on` на модулі чи data source** (розділ 6.3).
- **Баг провайдера.** Шукай issue в його GitHub і оновлюй провайдер.

### 17.6 `Provider produced inconsistent final plan` / `inconsistent result after apply`

Майже завжди це баг провайдера: у плані він пообіцяв одне, а API повернуло інше. Повторний apply часто проходить. Наступний крок — оновити провайдер і обов'язково пошукати issue: найімовірніше, ти не перший.

### 17.7 «Resource already exists»

`EntityAlreadyExists`, `BucketAlreadyOwnedByYou`, `InvalidGroup.Duplicate`: ресурс із таким ім'ям уже існує, але його немає в state. Або хтось створив його руками, або попередній apply створив ресурс і впав, не встигнувши записати його в state. Рішення — `import`.

### 17.8 Apply впав посередині

Terraform **не робить rollback**. Ресурси, створені до помилки, залишаються і вже записані в state. Ресурс, створення якого почалося, але не завершилося, позначається як **tainted**, і наступний apply його перестворить. Виправ причину, запусти `plan`/`apply` знову, і Terraform продовжить з того місця, де зупинився.

Якщо в кінці apply Terraform не зміг записати state у backend (наприклад, зникла мережа), він збереже state локально у файл `errored.tfstate`. Не видаляй його: залий у backend через `terraform state push errored.tfstate`, інакше Terraform «забуде» про щойно створені ресурси.

### 17.9 Помилки lock-файлу провайдерів

```
│ Error: Inconsistent dependency lock file
```

- Змінив обмеження версії в коді → `terraform init -upgrade`.
- У lock-файлі немає хешів для платформи CI → `terraform providers lock -platform=linux_amd64 -platform=darwin_arm64`.

### 17.10 `Backend configuration changed`

Змінив параметри backend (бакет, key), і Terraform вимагає явного рішення:

- `terraform init -migrate-state` — перенести наявний state у новий backend;
- `terraform init -reconfigure` — працювати з новим backend, нічого не переносячи (якщо правильний state там уже лежить).

### 17.11 Throttling і таймаути

- `ThrottlingException`, `Rate exceeded`, HTTP 429 — зменш `-parallelism` і налаштуй `max_retries` у провайдері.
- Довгі операції (RDS, EKS, CloudFront) падають за таймаутом — у багатьох ресурсів є блок `timeouts`:

```hcl
resource "aws_db_instance" "main" {
  # ...
  timeouts {
    create = "90m"
    update = "90m"
    delete = "60m"
  }
}
```

### 17.12 Plan дуже повільний

- Великий state → розбити його (розділ 7.8).
- Для локальних ітерацій → `-refresh=false` (але не в CI перед apply!).
- Кожен `init` завантажує провайдери заново → `TF_PLUGIN_CACHE_DIR`.
- Тисячі ресурсів у `for_each` або важкі data sources у циклах → переглянути структуру.

### 17.13 Алгоритм при будь-якій незрозумілій помилці

```bash
# 1. Прочитай помилку повністю: адреса ресурсу + код помилки API (AccessDenied, InvalidParameter...)

# 2. Які версії?
terraform version

# 3. Що Terraform думає про ресурс?
terraform state show 'aws_instance.web'

# 4. Детальний лог: видно сирі запити і відповіді API
TF_LOG=DEBUG TF_LOG_PATH=debug.log terraform plan
grep -i -A5 'error' debug.log | head -50

# 5. Під ким я взагалі працюю?
aws sts get-caller-identity

# 6. Що реально відбувається в хмарі: консоль, aws cli, CloudTrail
#    (для AccessDenied — хто і який саме API викликав)
```

> `aws sts get-caller-identity` вирішує напрочуд багато проблем: добра половина `AccessDenied` — це «не той профіль, не той акаунт, не та роль».

---

## 18. Каверзні питання на співбесіді — з відповідями {#interview}

### Як пояснити Terraform за 30 секунд

> Terraform — декларативний інструмент Infrastructure as Code. Я описую бажаний стан інфраструктури на HCL. Terraform через провайдери порівнює його зі state і з реальним станом ресурсів, будує граф залежностей, показує план змін, а після підтвердження виконує їх, паралельно там, де це можливо. State зберігається в remote backend з блокуванням, код поділений на модулі й окремі state для середовищ і шарів, а зміни проходять через PR, plan у CI і apply зі збереженого плану.

### Шпаргалка: що з'явилося в якій версії

На співбесідах інколи питають «з якої версії з'явилося X» або «як це робили до Y». Зубрити не треба, але еволюцію корисно розуміти:

| Версія | Що з'явилося |
|---|---|
| 1.1 | Блок `moved` |
| 1.2 | `precondition` / `postcondition`, `replace_triggered_by` |
| 1.3 | `optional()` зі значенням за замовчуванням в об'єктних типах |
| 1.4 | `terraform_data` (заміна `null_resource`) |
| 1.5 | Блоки `import` і `check`, `plan -generate-config-out` |
| 1.6 | `terraform test` |
| 1.7 | Блок `removed`, `mock_provider` у тестах, `for_each` в `import` |
| 1.8 | Функції від провайдерів, `moved` між різними типами ресурсів (якщо провайдер це підтримує) |
| 1.9 | `validation` змінної може посилатися на інші змінні й об'єкти |
| 1.10 | Ephemeral-ресурси, змінні та outputs; нативний lock у S3 (`use_lockfile`) |
| 1.11 | Write-only атрибути; `use_lockfile` стабільний, lock через DynamoDB — deprecated |
| 1.12 | `import` через `identity`; паралельні `run` у `terraform test` |
| 1.13 | Команда `terraform stacks` |
| 1.14 | `terraform query` і list-ресурси, блоки `action` |
| 1.15 | `const`-змінні в `source`/`version` модулів, атрибут `deprecated` для змінних і outputs, `validate` перевіряє backend |
| 1.16 | `import` усередині модулів, `lifecycle { destroy = false }`, `graph -format=mermaid`, `state show -json` |

### Базовий рівень

**1. Що таке Terraform і чим декларативний підхід відрізняється від імперативного?**
Terraform — IaC-інструмент: інфраструктура описується кодом, а Terraform приводить реальність у відповідність до опису. Імперативно ти пишеш кроки («створи 2 сервери»), і повторний запуск створить ще 2. Декларативно описуєш результат («має бути 2 сервери»), і повторний запуск нічого не змінить. Це ідемпотентність.

**2. Що роблять `init`, `plan`, `apply`, `destroy`?**
`init` завантажує провайдери й модулі, налаштовує backend, оновлює lock-файл. `plan` робить refresh, порівнює код зі state і показує зміни, нічого не змінюючи. `apply` виконує план (новий або збережений). `destroy` видаляє все, що є в state.

**3. Що таке state і навіщо він?**
JSON-файл, який зіставляє адреси ресурсів у коді з реальними об'єктами (ID) і зберігає їхні атрибути та залежності. Без нього Terraform не знає, які ресурси він створив, не може видалити те, що зникло з коду, і мусив би щоразу опитувати все підряд. Саме state відрізняє Terraform від Ansible.

**4. Що таке провайдер?**
Окремий плагін (бінарник), через який Terraform Core працює з конкретним API: AWS, Kubernetes, Cloudflare. Core спілкується з ним по gRPC. Провайдери пишуть HashiCorp, компанії-партнери і спільнота; завантажуються вони з Registry під час `init`.

**5. Чим `resource` відрізняється від `data`?**
`resource` — Terraform керує життєвим циклом: створює, змінює, видаляє. `data` — тільки читає те, що вже існує. Прибрав `resource` з коду — ресурс буде видалено; прибрав `data` — нічого не станеться.

**6. Variable vs local vs output?**
Variable — вхідний параметр, який можна задати ззовні. Local — обчислене значення всередині модуля, ззовні його не перевизначиш. Output — значення, яке модуль віддає назовні. Аналогія: параметри функції, локальні змінні і return.

**7. Який пріоритет у джерел змінних?**
Від нижчого до вищого: `default` → `TF_VAR_*` → `terraform.tfvars` → `terraform.tfvars.json` → `*.auto.tfvars` (за алфавітом) → `-var` / `-var-file` у CLI (у порядку вказання). Виграє останнє.

**8. Що комітити в Git, а що ні?**
Комітити: `*.tf`, `.terraform.lock.hcl`, `*.tfvars` без секретів. Не комітити: `.terraform/`, `*.tfstate*`, plan-файли, `crash.log`, override-файли, `.terraformrc`.

**9. Що означає `~> 6.0` і чим воно відрізняється від `~> 6.0.0`?**
`~>` дозволяє збільшувати лише останню вказану цифру. `~> 6.0` — будь-яка 6.x (≥ 6.0 і < 7.0). `~> 6.0.0` — лише patch-версії 6.0.x.

**10. Що означають `+`, `-`, `~`, `-/+`, `+/-`, `<=` у плані і що таке «known after apply»?**
Створення, видалення, оновлення на місці, заміна зі спершу видаленням, заміна зі спершу створенням (`create_before_destroy`), читання data source. «Known after apply» — значення, яке з'явиться лише після створення ресурсу (ID, IP). Тому план — прогноз, а не гарантія.

**11. Як Terraform визначає порядок створення ресурсів?**
Будує граф залежностей (DAG) з посилань між ресурсами: `subnet_id = aws_subnet.a.id` означає «спершу subnet». Незалежні ресурси створюються паралельно (за замовчуванням по 10), видаляються у зворотному порядку. Порядок блоків у файлах значення не має. Невидимі залежності задаються через `depends_on`.

**12. Terraform vs Ansible?**
Terraform — декларативне створення інфраструктури зі state: він знає, що створив, і видаляє зайве. Ansible — процедурне керування конфігурацією без state: налаштовує ОС і застосунки всередині серверів. Ansible створить бакет, але якщо прибрати задачу з плейбука, бакет залишиться. Зазвичай їх поєднують: Terraform створює VM, Ansible (або заздалегідь Packer) їх налаштовує.

### Середній рівень

**13. `count` vs `for_each`: у чому підступ `count`?**
`count` адресує екземпляри за індексом. Видалення елемента з середини списку зсуває індекси, і Terraform змінює або перестворює всі наступні ресурси. `for_each` адресує за стабільним ключем, тож видаляється рівно той елемент. `count` — для вимикача `0/1` і однакових копій, `for_each` — для колекцій.

**14. Коли `for_each` не працює?**
Коли ключі невідомі на етапі plan, наприклад це ID ресурсів, які ще не створені: помилка `Invalid for_each argument`. Ключі треба будувати зі статичних даних (імена, ключі вхідних map), а невідомі значення класти в значення map. Ще одне обмеження: ключі не можуть бути sensitive.

**15. Що таке remote backend і state locking? Як зараз робиться lock у S3?**
Remote backend зберігає state централізовано (S3, GCS, Azure Blob, HCP Terraform), а locking не дає двом процесам одночасно змінювати один state. У S3 з Terraform 1.10 є нативне блокування `use_lockfile = true` (lock-об'єкт у самому бакеті через conditional writes). З 1.11 воно стабільне, а блокування через DynamoDB — deprecated.

**16. Чому в блоці `backend` не можна використати змінні?**
Backend ініціалізується в `init` раніше, ніж обчислюються змінні: щоб їх обробити, потрібен state, а щоб знайти state, потрібен backend. Рішення — partial configuration (`terraform init -backend-config=prod.hcl` або `-backend-config="key=..."`) чи Terragrunt, що генерує backend. В OpenTofu з 1.8 змінні в backend дозволені.

**17. Що таке drift і що з ним робити?**
Розбіжність між кодом/state і реальністю: хтось змінив ресурс поза Terraform. Виявляється через `plan` (або `plan -refresh-only`), а систематично — нічним `plan -detailed-exitcode` (код 2 означає, що є зміни). Далі рішення: якщо ручна зміна правильна, перенести її в код; якщо ні, `apply` поверне все як у коді. У довгостроковій перспективі — забрати в людей права на запис у prod-консолі.

**18. Розкажи про `lifecycle`.**
`create_before_destroy` — при заміні спершу створити новий ресурс (zero-downtime; потрібні унікальні імена, наприклад `name_prefix`). `prevent_destroy` — помилка, якщо план видаляє ресурс. `ignore_changes` — не реагувати на зміни атрибутів ззовні (наприклад, `desired_capacity` від autoscaler). `replace_triggered_by` — перестворити ресурс, коли змінився інший. Плюс `precondition` і `postcondition`.

**19. Чи захистить `prevent_destroy` базу від видалення?**
Лише частково. Він зупинить план, який видаляє чи перестворює ресурс, і зупинить `terraform destroy`. Але якщо видалити з коду весь блок ресурсу, разом із ним зникне і `prevent_destroy`, і Terraform видалить базу. Від видалення через консоль чи CLI він теж ніяк не захищає. Тому потрібні ще `deletion_protection = true` на боці AWS, бекапи і policy as code в CI. (З 1.16 є ще `lifecycle { destroy = false }`: з ним навіть `terraform destroy` лише «забуває» ресурс, але й цей захист зникає разом із блоком ресурсу.)

**20. Як перейменувати ресурс, не перестворюючи його?**
Блоком `moved` з `from = aws_instance.web` і `to = aws_instance.app` (1.1+): зміна проходить через plan і review. Старий спосіб — `terraform state mv`, який одразу змінює state, без плану.

**21. Як завести під Terraform інфраструктуру, створену руками?**
Блоки `import` (1.5+) з адресою та ID ресурсу; `terraform plan -generate-config-out=generated.tf`, щоб згенерувати код; почистити код; довести план до `0 to add, 0 to change, 0 to destroy`; apply. З 1.7 `import` підтримує `for_each` для масового імпорту. Команда `terraform import` теж працює, але оминає review і не генерує код.

**22. Що таке модуль, як його версіонувати і чому в модулі не має бути блоку `provider`?**
Модуль — будь-який каталог із `.tf`-файлами; root — той, де запускаєш, child — ті, що викликаються. Версіонування — semver-теги в Git (`?ref=v1.2.3`) або реєстр (`version = "~> 1.2"`). Блок `provider` у модулі ламає для нього `count`/`for_each`/`depends_on` і не дає видалити його ресурси, якщо прибрати виклик модуля. Провайдери має передавати root-модуль.

**23. Що таке workspaces і чому їх не радять для dev/prod?**
CLI workspaces — кілька state для одного коду в одному backend. Для prod це погано: легко застосувати зміни не в тому workspace, однакові облікові дані і backend для всіх середовищ, відмінності реалізуються умовами по всьому коду. Workspaces підходять для однакових тимчасових середовищ (preview на PR). Для середовищ краще окремі каталоги, state і акаунти.

**24. Чи захищає `sensitive = true` секрет?**
Тільки від показу у виводі plan/apply. У state і в plan-файлі значення лежить відкритим текстом, а `terraform output NAME` і `terraform output -json` його покажуть. Справжній захист — шифрований backend з обмеженим доступом, ephemeral-ресурси і write-only атрибути (1.10/1.11+) або підхід, коли секрет узагалі не проходить через Terraform (RDS `manage_master_user_password`).

**25. Що таке `terraform_remote_state` і які в нього альтернативи?**
Data source, який читає outputs іншого state. Мінуси: потрібен доступ на читання до всього state (разом із секретами) і сильна зв'язність. Альтернативи: data sources за тегами або іменами, SSM Parameter Store чи Consul як явний контракт, `dependency` у Terragrunt, outputs між workspaces в HCP Terraform.

**26. Чому provisioners — «last resort»?**
`local-exec` і `remote-exec` виконуються лише під час створення (або видалення), не відображаються в плані, не ідемпотентні, їхній результат не зберігається в state, а помилка робить ресурс tainted. `remote-exec` ще й вимагає мережевого доступу і SSH з машини, де працює Terraform. Альтернативи: `user_data`/cloud-init, готові образи з Packer, Ansible після Terraform, нативні ресурси провайдера. А для дій поза CRUD (скинути кеш CloudFront, викликати Lambda) з 1.14 є блоки `action`, які описує сам провайдер.

**27. Що буде, якщо apply впаде посередині? Чи є rollback?**
Rollback немає. Створене до помилки залишається і записане в state, недостворений ресурс стає tainted. Виправляєш причину, запускаєш apply знову, і Terraform доводить інфраструктуру до бажаного стану. (На відміну від CloudFormation, який відкочує стек автоматично.)

**28. Коли можна використовувати `-target`?**
В аварійних ситуаціях: обійти проблему з `for_each` на невідомих ключах, точково щось виправити під час інциденту. Як регулярна практика це погано: план неповний, state розходиться з кодом, легко пропустити залежні зміни. Terraform і сам попереджає, що `-target` не для щоденного використання.

**29. Що таке `dynamic`-блоки і коли їх не варто використовувати?**
Генерація вкладених блоків (`ingress`, `setting`) у циклі через `for_each`. Корисно в модулях, коли кількість блоків задає користувач. Не варто, коли блоки можна написати явно: код стає нечитабельним. У багатьох випадках краще окремі ресурси (наприклад, `aws_vpc_security_group_ingress_rule`).

**30. Що зараз із `terraform refresh` і `terraform taint`?**
Обидві команди deprecated. Замість `refresh` — `terraform apply -refresh-only` (з планом і підтвердженням), замість `taint` — `terraform apply -replace=ADDR` (одразу видно в плані, що буде перестворено).

**31. Що таке `terraform_data` і навіщо він, якщо є `null_resource`?**
Вбудований (1.4+) ресурс, якому не потрібен жоден провайдер. Він зберігає значення (`input` → `output`) і перестворюється, коли змінюється `triggers_replace`. Замінює `null_resource` з провайдера `null`; зручний як тригер для `replace_triggered_by` або як носій для provisioners. З 1.9 наявні `null_resource` можна перенести на `terraform_data` блоком `moved`, нічого не перестворюючи.

**32. Як працювати з кількома регіонами чи акаунтами в одній конфігурації?**
Кілька блоків `provider` з `alias` (`provider = aws.us_east_1` у ресурсі), `assume_role` для інших акаунтів, передача в модулі через `providers = { ... }` і `configuration_aliases`. В AWS-провайдері v6 для іншого регіону того самого акаунта можна вказати `region` прямо в ресурсі.

### Senior-рівень

**33. Як організувати Terraform для багатьох команд і десятків AWS-акаунтів?**
AWS Organizations з акаунтом на кожне середовище або команду; state, розбитий за акаунтами, середовищами і шарами; спільні модулі в окремих репозиторіях із semver і приватним реєстром; оркестрація через Terragrunt, HCP Terraform або Spacelift; CI з OIDC і окремими ролями на кожен акаунт, ролі для plan і apply розділені; policy as code; CODEOWNERS на каталоги; drift detection; read-only доступ людей до prod-консолі. Головна думка: **мінімальний blast radius і чіткі межі відповідальності**.

**34. Як розбити величезний state на кілька, нічого не перестворюючи?**
Сучасний спосіб (1.7+): у новій конфігурації — код ресурсів плюс блоки `import` з їхніми ID; у старій — прибрати код ресурсів і додати блоки `removed` з `lifecycle { destroy = false }`. Спершу apply нової конфігурації (ресурси імпортуються), потім старої (ресурси забуваються без видалення). Обидва кроки проходять через plan, де видно `0 to destroy`. Перед початком — `state pull` у бекап і заморозка змін у цьому state. Старий спосіб — `terraform state mv -state=... -state-out=...` з локальними копіями і `state push`.

**35. Як Terraform влаштований усередині?**
Core парсить конфігурацію, будує граф ресурсів і для кожного вузла через gRPC викликає провайдер: `ReadResource` (refresh), `PlanResourceChange` (провайдер каже, що зміниться і чи потрібна заміна), `ApplyResourceChange` (виконання). Граф обходиться паралельно з обмеженням `-parallelism`. State оновлюється під час apply; якщо записати його в backend наприкінці не вдалося, Terraform зберігає локальний `errored.tfstate`.

**36. Що таке `serial` і `lineage`?**
`lineage` — унікальний ID, який створюється разом зі state і більше не змінюється. `serial` — лічильник, що збільшується з кожним записом. Terraform відмовиться записати state з іншим lineage («чужий» state) або з меншим serial (застарілий), якщо не вказати `-force` у `state push`. Це захист від перезапису state не тим файлом.

**37. Як безпечно працювати із секретами в Terraform?**
По можливості не пропускати їх через Terraform узагалі (managed passwords, генерація секретів самим сервісом). Де без цього не обійтися — ephemeral-ресурси для читання і write-only атрибути для передачі: вони не потрапляють ні в state, ні в plan. Облікові дані самого Terraform — короткоживучі, через OIDC або Vault. State — шифрований, з мінімальним доступом. Секретів у Git немає ніколи, а `sensitive` на змінних і outputs захищає від витоків у логи.

**38. Чому `terraform plan` на PR з форка — діра в безпеці?**
Plan не є безпечною read-only операцією: під час нього виконуються data sources (зокрема `external`, який запускає будь-яку програму, чи `http`, що може відправити дані назовні) і ініціалізуються провайдери, а модуль у PR може бути звідки завгодно. Шкідливий PR здатен украсти облікові дані CI ще до review. Тому plan із привілейованими обліковими даними — тільки для довірених гілок і авторів, а для зовнішніх PR — лише `fmt`/`validate`/статичний аналіз без секретів.

**39. Чим шкодить `depends_on` на модулі?**
Усе всередині модуля починає залежати від усього, від чого залежить модуль. Data sources у модулі відкладаються до apply, їхні значення стають `known after apply`, і план при кожному запуску показує зміни, а то й перестворення. Краще передавати конкретні значення (outputs) як вхідні змінні: тоді залежності точні.

**40. Як оновити major-версію провайдера на великій інфраструктурі?**
Прочитати upgrade guide, підняти версію в гілці, `init -upgrade`, прогнати plan в **усіх** state (скриптом або через CI), розібрати кожну зміну, виправити deprecated-аргументи, котити хвилями dev → stage → prod. Схему state провайдер мігрує сам, але значення за замовчуванням і поведінка можуть змінитися, і саме це ловимо в плані. Ніколи не змішувати оновлення провайдера з іншими змінами в одному PR.

**41. Terraform чи OpenTofu?**
Terraform — BSL 1.1, розвиває HashiCorp (IBM), найширша екосистема, HCP Terraform, Stacks, Sentinel. OpenTofu — MPL 2.0 під Linux Foundation, сумісний за мовою і state, ті самі провайдери, плюс власні фічі: шифрування state на клієнті, змінні в backend, `for_each` для провайдерів, прапорець `-exclude`. (Змінні в `source` модулів OpenTofu отримав раніше; у Terraform вони з'явилися в 1.15.) Вибір залежить від ліцензійних вимог компанії, потрібних фіч і платформи запуску. Міграцію між ними зазвичай роблять без особливих проблем, але чим далі проєкти розходяться, тим більше нюансів.

**42. Як тестувати Terraform-код?**
Піраміда: `fmt`/`validate`/`tflint` → сканери безпеки (`trivy`, `checkov`) → unit-тести модулів через `terraform test` на plan з `mock_provider` → політики на JSON-план (`conftest`, Sentinel) → інтеграційні тести з реальним apply у пісочниці (`terraform test` з `command = apply`, Terratest) → Infracost. І найважливіший тест — уважно прочитаний plan у PR.

**43. Чому apply у CI має йти зі збереженого plan-файлу?**
Щоб застосувалося саме те, що переглянули. Без файлу apply перераховує план, і якщо за цей час змінився код, state чи реальність, застосується щось інше. Зі збереженим планом Terraform сам відмовиться працювати, якщо state змінився (`Saved plan is stale`). Нюанси: plan-файл містить секрети, і застосовувати його треба тією самою версією Terraform і провайдерів.

**44. Як `create_before_destroy` взаємодіє із залежностями і з унікальними іменами?**
Він поширюється на залежності: якщо A з `create_before_destroy` залежить від B, B теж замінюватиметься в режимі «спершу створити». Ресурси з фіксованими унікальними іменами (security group, IAM role, LB) не зможуть існувати удвох, тому потрібен `name_prefix` або випадковий суфікс. І треба пам'ятати, що на час заміни існують обидві копії: подвійні ліміти, квоти і вартість.

**45. Чому `data "aws_ami"` з `most_recent = true` — міна сповільненої дії?**
З кожним новим релізом AMI змінюється ID, а зміна `ami` в `aws_instance` вимагає перестворення. Одного дня рядовий apply перестворить усі інстанси. Рішення: фіксувати AMI змінною й оновлювати її свідомо, `ignore_changes = [ami]` або ASG з launch template та instance refresh.

**46. Як зменшити blast radius Terraform?**
Маленькі state за середовищами і шарами, окремі акаунти для prod, окремі ролі з мінімальними правами для кожного state, `prevent_destroy` + `deletion_protection` на критичних ресурсах, policy as code, яка блокує видалення баз у CI, обов'язковий review плану, заборона ручних apply у prod.

**47. Чому не радять в одному state створювати EKS-кластер і одразу ресурси в ньому через провайдери `kubernetes`/`helm`?**
Провайдер `kubernetes` налаштовується атрибутами кластера (endpoint, сертифікат), а вони невідомі, поки кластер не створений, або змінюються, коли кластер перестворюється. Провайдери налаштовуються до того, як ці значення стануть відомими, звідси помилки з'єднання і непередбачувані плани. Видалення теж стає крихким: щоб знищити ресурси в кластері, кластер має ще існувати. Документація провайдера `kubernetes` прямо радить окремі state (окремі apply) для кластера і для того, що в нього деплоїться. А застосунки взагалі краще віддати ArgoCD чи Flux.

### Сценарні питання: «Що ти робитимеш, якщо...»

**48. Колега змінив security group руками в консолі.**
`terraform plan` покаже drift. З'ясувати, навіщо була зміна: якщо вона потрібна — внести її в код через PR (план стане чистим); якщо ні — `apply` поверне стан із коду. Потім усунути причину: read-only доступ до prod-консолі, drift detection за розкладом, правило «аварійні ручні зміни обов'язково переносяться в код».

**49. CI-джоб убили посеред apply, і state заблокований.**
Переконатися, що жоден процес справді не працює (CI, колеги). Якщо Terraform устиг зберегти `errored.tfstate`, залити його через `state push`. Зняти lock командою `terraform force-unlock <ID>`. Запустити `plan` і подивитися, що встигло застосуватися, а що ні; tainted-ресурси будуть перестворені. Довести apply до кінця. І налаштувати `concurrency` в CI так, щоб такий джоб не можна було перервати.

**50. Хтось видалив state-файл, або state зламаний.**
Не запускати apply: Terraform вирішить, що нічого немає, і почне створювати все заново (з конфліктами імен або дублікатами). Відновити попередню версію з versioning бакета, `state push`, перевірити `plan`. Якщо бекапу немає, заново імпортувати все через блоки `import` (з допомогою `-generate-config-out`). Висновки для постмортему: versioning, обмежений доступ до бакета, `prevent_destroy` на бакеті.

**51. Plan хоче перестворити продакшен-базу.**
Стоп, нічого не застосовувати. Знайти в плані атрибут із `# forces replacement` і зрозуміти причину: змінили незмінний атрибут (`identifier`, `engine`, `availability_zone`, `storage_encrypted`, `username`...), зробили рефакторинг без `moved`, оновлення провайдера змінило значення за замовчуванням, змінився snapshot чи AMI з data source. Виправити: повернути значення, додати `moved`, `ignore_changes` для атрибута, а якщо зміна справді потрібна — окремо спланувати міграцію даних (snapshot → нова база → переключення). Запобіжники на майбутнє: `prevent_destroy`, `deletion_protection`, політика в CI, яка блокує заміну баз.

**52. Треба підняти нове середовище (наприклад, stage), «як prod».**
Якщо код уже модульний, створити новий каталог `envs/stage/*` (або набір Terragrunt-конфігурацій) з тими самими модулями тієї самої версії, власними `tfvars`, окремим state і, в ідеалі, окремим акаунтом. Застосовувати шарами в порядку залежностей: network → data → compute → apps. Якщо код не модульний, це добрий момент винести спільне в модулі, використовуючи `moved` для prod.

**53. Plan виконується 15 хвилин. Як прискорити?**
Розбити state за шарами; для локальних ітерацій — `-refresh=false`; підняти `-parallelism`, якщо API дозволяє, і зменшити, якщо ловиш throttling; кеш провайдерів у CI, щоб не завантажувати їх при кожному `init`; прибрати важкі data sources у циклах. І поміряти: `TF_LOG=DEBUG` покаже, на що саме йде час.

**54. Apply падає з «already exists».**
Ресурс існує, але його немає в state: або його створили руками, або він загубився через невдалий apply. Додати блок `import`, переконатися, що план показує лише імпорт без змін, і застосувати.

**55. Ресурс треба прибрати з Terraform, але залишити в хмарі.**
Блок `removed` з `lifecycle { destroy = false }` (1.7+) замість коду ресурсу; зміна проходить через plan. Старий спосіб — `terraform state rm ADDR`.

**56. Ти пишеш модуль, яким користуються 20 команд. Як оновлювати його, нікому нічого не зламавши?**
Semver: нові необов'язкові змінні з безпечним default — minor-версія; видалення чи перейменування змінних і ресурсів — major. Блоки `moved` для перейменувань усередині модуля, щоб у споживачів нічого не перестворювалося. CHANGELOG з інструкцією з міграції, deprecation-період для старих змінних і outputs (атрибут `deprecated`, 1.15+), `terraform test` і приклади в `examples/`, які проганяються в CI. Споживачі фіксують версію і оновлюються, коли готові.

**57. Треба видалити 1 з 10 інстансів, створених через `count`. Як?**
Якщо просто прибрати елемент зі списку, зсунуться індекси і постраждають сусіди. Правильний шлях — спершу мігрувати на `for_each` зі стабільними ключами через `moved` (`[0]` → `["web-a"]` і так далі), переконатися, що план чистий, і лише потім видалити потрібний ключ. Швидкий аварійний варіант — `terraform destroy -target='aws_instance.web[3]'`, але без виправлення коду наступний apply створить інстанс знову, тож це милиця.

**58. Як зрозуміти, що Terraform-код у компанії «здоровий»?**
`plan` у всіх state показує `No changes` (немає ні drift, ні незастосованих змін); apply роблять тільки з CI; версії зафіксовані й не відстають на роки; state невеликі і логічно розбиті; модулі версіоновані; секретів немає в Git і, наскільки можливо, у state; нове середовище піднімається за години, а не за тижні.

---

## 19. Чек-лист: що треба знати про Terraform {#checklist}

- [ ] Пояснити, навіщо IaC і чим декларативний підхід відрізняється від імперативного
- [ ] Core vs провайдери, gRPC, Registry, рівні провайдерів (Official / Partner / Community)
- [ ] Цикл `init` → `plan` → `apply` → `destroy` і що відбувається на кожному кроці
- [ ] Три стани: код, state, реальність; refresh; drift
- [ ] Читати plan: `+`, `-`, `~`, `-/+`, `+/-`, `<=`, `forces replacement`, `known after apply`
- [ ] Граф залежностей, неявні залежності через посилання, `depends_on`, `-parallelism`
- [ ] HCL: типи, variables (validation, sensitive, optional), locals, outputs, data sources, вирази, `dynamic`, функції
- [ ] Пріоритет джерел змінних
- [ ] `count` vs `for_each` і пастка зі зсувом індексів
- [ ] `lifecycle`: `create_before_destroy`, `prevent_destroy` (і його межі), `ignore_changes`, `replace_triggered_by`
- [ ] `precondition`, `postcondition`, `check`
- [ ] State: що в ньому, навіщо він, секрети в state, `serial` і `lineage`
- [ ] Remote backend, locking, `use_lockfile` у S3 замість DynamoDB, `force-unlock`
- [ ] Partial backend configuration і чому в backend немає змінних
- [ ] Розбиття state за середовищами й шарами, blast radius, передача даних між state
- [ ] Версії: `required_version`, `~>`, lock-файл (і те, що модулі він не фіксує), `init -upgrade`, `providers lock`
- [ ] Модулі: структура, джерела, версіонування, жодного `provider` усередині, `configuration_aliases`
- [ ] Workspaces vs каталоги vs Terragrunt
- [ ] `moved`, `import` (+ `-generate-config-out`), `removed`, `-replace`, `-refresh-only`
- [ ] Секрети: межі `sensitive`, ephemeral, write-only, managed passwords, OIDC
- [ ] Небезпека plan на неперевірених PR, supply chain
- [ ] CI/CD: plan у PR, apply зі збереженого плану, `concurrency`, окремі ролі
- [ ] Тестування: `validate`, `tflint`, `trivy`/`checkov`, `terraform test` з `mock_provider`, Terratest, conftest, Infracost
- [ ] Траблшутінг: lock, зламаний state, `Invalid for_each argument`, `Cycle`, perpetual diff, `already exists`, падіння apply посередині, `errored.tfstate`, `TF_LOG`
- [ ] Нове в 1.14–1.16: `terraform query`, actions, `const`-змінні в `source` модулів, `deprecated`, `lifecycle { destroy = false }`
- [ ] Terraform vs OpenTofu vs CloudFormation vs Pulumi vs Ansible

---

Більше про самі AWS-сервіси, якими керує Terraform (IAM, VPC, S3, RDS), — у статті [AWS для DevOps](/blog/aws-guide/).
