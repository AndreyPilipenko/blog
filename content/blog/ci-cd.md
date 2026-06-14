---
title: "CI/CD: повний гайд + усі каверзні питання для співбесіди"
date: 2026-06-14
draft: false
tags: ["ci/cd", "devops", "github-actions", "jenkins", "interview"]
categories: ["DevOps"]
description: "Від базових концепцій до підступних питань на Senior-рівні: pipeline, artifacts, secrets, rollback, GitOps — усе в одній статті."
---

> Стаття розрахована на тих, хто готується до DevOps-співбесіди або хоче систематизувати знання по CI/CD. Охоплює теорію, практику, GitHub Actions і Jenkins — із реальними прикладами конфігів і поясненням «чому саме так».

---

## Зміст

1. [Що таке CI/CD і навіщо це взагалі](#що-таке-cicd)
2. [Ключові концепції і термінологія](#ключові-концепції)
3. [GitHub Actions — детально](#github-actions)
4. [Jenkins — детально](#jenkins)
5. [Artifacts, кешування, оптимізація](#artifacts-і-кешування)
6. [Secrets і безпека в пайплайні](#secrets-і-безпека)
7. [Стратегії деплою](#стратегії-деплою)
8. [Rollback і disaster recovery](#rollback)
9. [Моніторинг і observability пайплайну](#моніторинг-пайплайну)
10. [GitOps і CD у Kubernetes](#gitops)
11. [Каверзні питання на співбесіді](#каверзні-питання)

---

## Що таке CI/CD {#що-таке-cicd}

**Continuous Integration (CI)** — практика, при якій кожен розробник мінімум раз на день інтегрує свій код у спільну гілку. Кожна інтеграція автоматично верифікується: запускаються build, тести, линтери. Мета — виявляти конфлікти і баги якомога раніше, поки вони ще дешеві у виправленні.

**Continuous Delivery (CD)** — розширення CI: після успішного пайплайну артефакт готовий до деплою в будь-який момент. Фактичний push на прод — ручна дія (клік кнопки або апрув).

**Continuous Deployment** — ще один крок: якщо всі перевірки пройдені, код автоматично летить на прод без жодного ручного кроку. Вимагає дуже зрілої культури тестування.

```
Developer push
      │
      ▼
  ┌─────────┐    fail     ┌──────────────┐
  │  CI Run │ ──────────► │ Notify team  │
  └────┬────┘             └──────────────┘
       │ pass
       ▼
  ┌─────────────┐
  │  Artifact   │  ◄── docker image / binary / helm chart
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐   manual gate    ┌──────────┐
  │  Staging    │ ──────────────► │   Prod   │   ← Continuous Delivery
  └─────────────┘                 └──────────┘

  або автоматично ──────────────────────────►        ← Continuous Deployment
```

### Чому CI/CD — це не просто «автозапуск тестів»

- Скорочує **lead time** (час від коміту до прод) з тижнів до годин
- Знижує **batch size**: маленькі зміни легше ревʼюити і відкочувати
- Дає **fast feedback loop**: розробник дізнається про зламаний тест за хвилини, а не на code review через день
- Зменшує **deployment fear**: якщо деплоїти часто, кожен деплой — дрібна і зрозуміла зміна

---

## Ключові концепції {#ключові-концепції}

### Pipeline

Послідовність автоматизованих етапів (stages/jobs), через які проходить код від коміту до деплою. Типова структура:

```
lint → test → build → security-scan → publish → deploy-staging → smoke-test → deploy-prod
```

### Job vs Step vs Stage

| Термін | GitHub Actions | Jenkins |
|--------|----------------|---------|
| Найбільша одиниця | Workflow | Pipeline |
| Середня | Job | Stage |
| Найменша | Step | Step |

У **GitHub Actions** jobs за замовчуванням виконуються паралельно. Залежності між ними задаються через `needs`.  
У **Jenkins Declarative Pipeline** stages виконуються послідовно, але можна використовувати `parallel {}` блок.

### Runner / Agent

Машина або контейнер, де фізично виконується код пайплайну.

- **GitHub Actions**: GitHub-hosted runners (ubuntu-latest, windows-latest, macos-latest) або self-hosted.
- **Jenkins**: Master node (контролер) + Agents (worker-и). Агентами можуть бути фізичні машини, Docker-контейнери або Kubernetes Pods.

### Artifact

Результат білду — файл або набір файлів, який «переживає» завершення job і може бути використаний наступним етапом або збережений. Наприклад: `app.jar`, `dist/`, Docker image, helm chart.

### Trigger (тригер)

Подія, яка запускає пайплайн: push у гілку, PR, tag, розклад (cron), ручний запуск, webhook від зовнішньої системи.

---

## GitHub Actions {#github-actions}

### Анатомія workflow-файлу

```yaml
# .github/workflows/ci.yml          # шлях до файлу — GitHub шукає workflows саме тут

name: CI Pipeline                   # назва, відображається в UI вкладки Actions

# ─── ТРИГЕРИ ──────────────────────────────────────────────────────────────────
on:                                 # блок "коли запускати цей workflow"

  push:                             # тригер: хтось зробив git push
    branches: [main, develop]       # але тільки якщо push йде в ці гілки
                                    # push у feature/xyz — НЕ запустить workflow

  pull_request:                     # тригер: відкрили / оновили PR
    branches: [main]                # тільки якщо PR цілиться в main
                                    # це запускає CI на кожен новий коміт у PR

  workflow_dispatch:                # тригер: ручний запуск через UI або API
                                    # без параметрів — просто кнопка "Run workflow"

# ─── ГЛОБАЛЬНІ ЗМІННІ ─────────────────────────────────────────────────────────
env:                                # змінні доступні у ВСІХ jobs цього workflow
  REGISTRY: ghcr.io                 # адреса GitHub Container Registry
                                    # ghcr.io — вбудований в GitHub, не треба
                                    # платити за DockerHub або піднімати свій

  IMAGE_NAME: ${{ github.repository }}  # контекст github.repository повертає
                                        # "owner/repo-name", наприклад "andrii/myapp"
                                        # буде використано як частина тегу image

# ─── JOBS ─────────────────────────────────────────────────────────────────────
jobs:                               # список jobs; за замовчуванням виконуються
                                    # ПАРАЛЕЛЬНО, якщо не вказано needs

# ══════════════════════════════════════════════════════════════════════════════
# JOB 1: LINT
# ══════════════════════════════════════════════════════════════════════════════
  lint:                             # ім'я job (довільне, але унікальне в межах workflow)
    runs-on: ubuntu-latest          # тип runner'а; ubuntu-latest = свіжа Ubuntu VM
                                    # GitHub підіймає чисту VM під кожен run
                                    # альтернативи: windows-latest, macos-latest,
                                    # або self-hosted для власних машин

    steps:                          # кроки виконуються ПОСЛІДОВНО зверху вниз

      - uses: actions/checkout@v4   # крок 1: клонує репо в робочу директорію VM
                                    # без цього кроку — файлів коду немає взагалі
                                    # @v4 — закріплена мажорна версія action
                                    # НІКОЛИ не пишіть @latest — сьогодні може
                                    # зламати пайплайн без жодних змін з вашого боку

      - uses: actions/setup-python@v5   # крок 2: встановлює Python на runner
        with:                           # блок параметрів для цього action
          python-version: '3.12'        # яку версію Python встановити
                                        # без цього кроку — береться системний Python
                                        # VM (може бути 3.10 або будь-який інший)

      - run: pip install ruff && ruff check .
        # крок 3: shell команда (bash за замовчуванням)
        # pip install ruff      — встановлює ruff (швидкий Python лінтер на Rust)
        # && ruff check .       — запускає лінтер по всіх .py файлах
        # якщо ruff знайде помилки — повертає exit code != 0
        # ненульовий exit code = крок вважається failed = весь job failed

# ══════════════════════════════════════════════════════════════════════════════
# JOB 2: TEST
# ══════════════════════════════════════════════════════════════════════════════
  test:
    runs-on: ubuntu-latest

    needs: lint                     # ЗАЛЕЖНІСТЬ: цей job стартує тільки після того,
                                    # як job "lint" завершився УСПІШНО
                                    # якщо lint впав — test навіть не стартує
                                    # це заощаджує хвилини runner-часу

    strategy:                       # блок стратегії запуску job
      matrix:                       # matrix = запустити job кілька разів
                                    # з різними комбінаціями змінних
        python-version: ['3.11', '3.12']
        # результат: GitHub створить 2 паралельних job:
        #   test (3.11)
        #   test (3.12)
        # якщо додати ще одну вісь — буде декартовий добуток:
        #   os: [ubuntu, windows] + python: [3.11, 3.12] = 4 jobs

    steps:
      - uses: actions/checkout@v4   # знову клонуємо — кожен matrix job це окрема VM
                                    # стан між jobs НЕ зберігається автоматично

      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          # ${{ matrix.python-version }} — підставляє поточне значення з matrix
          # для першого job = '3.11', для другого = '3.12'
          # так один набір steps працює для всіх версій

      - name: Cache pip              # name — людська назва кроку в UI (опціонально,
                                    # але рятує коли треба знайти де впало)
        uses: actions/cache@v4      # action для кешування директорій між runs
        with:
          path: ~/.cache/pip        # що кешувати: директорія pip cache
                                    # pip завантажені пакети зберігає саме тут

          key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
          # ключ кешу — рядок який ідентифікує цей конкретний кеш
          # runner.os       → "Linux" (щоб не мікшувати кеші різних ОС)
          # hashFiles(...)  → SHA256 хеш файлу requirements.txt
          # якщо requirements.txt не змінився — хеш той самий → кеш HIT
          # якщо додали нову залежність → хеш інший → кеш MISS → pip качає заново
          # без кешу: pip install може займати 2-3 хв на кожен run

      - run: pip install -r requirements.txt && pytest --cov
        # pip install -r requirements.txt  — встановлює залежності з файлу
        #   якщо кеш HIT — pip не качає з інтернету, бере локально (секунди)
        #   якщо кеш MISS — качає з PyPI (хвилини)
        # pytest --cov                     — запускає тести з coverage
        #   --cov генерує звіт покриття коду
        #   падіння будь-якого тесту = exit code 1 = job failed

# ══════════════════════════════════════════════════════════════════════════════
# JOB 3: BUILD AND PUSH
# ══════════════════════════════════════════════════════════════════════════════
  build-and-push:
    runs-on: ubuntu-latest

    needs: test                     # стартує тільки після ВСІХ matrix jobs "test"
                                    # тобто і 3.11, і 3.12 мають пройти успішно

    permissions:                    # явне обмеження прав GITHUB_TOKEN для цього job
                                    # за замовчуванням токен може все — це небезпечно
      contents: read                # дозвіл читати код репозиторію (для checkout)
      packages: write               # дозвіл писати в GitHub Packages (ghcr.io)
                                    # без цього рядка docker push поверне 403

    steps:
      - uses: actions/checkout@v4   # клонуємо код — потрібен Dockerfile

      - name: Log in to GHCR        # логінимось у container registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}    # ghcr.io (з глобального env вгорі)
          username: ${{ github.actor }}    # github.actor = логін того, хто зробив push
                                           # наприклад "andrii"
          password: ${{ secrets.GITHUB_TOKEN }}
          # GITHUB_TOKEN — спеціальний секрет, який GitHub АВТОМАТИЧНО генерує
          # для кожного workflow run і видаляє після його завершення
          # не треба створювати вручну — він завжди є
          # scope обмежений правами з блоку permissions вище
          # тому packages: write і дає нам право пушити в ghcr.io

      - name: Build and push
        uses: docker/build-push-action@v5  # action для build + push в одному кроці
        with:
          push: ${{ github.ref == 'refs/heads/main' }}
          # умовний push: будуємо image завжди, але пушимо ТІЛЬКИ якщо
          # це push у гілку main (refs/heads/main)
          # якщо це PR або push у develop — push: false
          # навіщо: в PR хочемо перевірити що image будується без помилок,
          # але не забруднювати registry тимчасовими образами

          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          # фінальний тег image, наприклад:
          # ghcr.io/andrii/myapp:a3f8c91d...
          #
          # env.REGISTRY    → ghcr.io
          # env.IMAGE_NAME  → andrii/myapp
          # github.sha      → повний SHA коміту (40 символів)
          #
          # чому SHA, а не latest?
          # SHA незмінний — завжди можна знайти який коміт в якому image
          # latest постійно перезаписується — важко відкотити і дебажити
```

### Важливі особливості GitHub Actions

**Контексти** — обʼєкти з метаданими: `github`, `env`, `secrets`, `runner`, `job`, `steps`. Доступні через `${{ <context>.<property> }}`.

**GITHUB_TOKEN** — автоматично генерується для кожного workflow run. Має права в межах репозиторію. Для cross-repo операцій потрібен PAT або GitHub App token.

**Environments** — іменовані середовища (staging, production) з окремими секретами і **protection rules** (required reviewers, deployment branches). Найважливіше для CD:

```yaml
deploy:
  environment:
    name: production
    url: https://myapp.com    # відображається в UI
  steps:
    - run: ./deploy.sh
```

**Reusable workflows** — виклик одного workflow з іншого:

```yaml
jobs:
  call-deploy:
    uses: org/shared-workflows/.github/workflows/deploy.yml@main
    with:
      environment: production
    secrets: inherit
```

**Composite actions** — упаковка кількох steps в одну дію для перевикористання:

```yaml
# .github/actions/setup-app/action.yml
name: Setup App
runs:
  using: composite
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: '20'
    - run: npm ci
      shell: bash
```

### Паралельність і concurrency

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true   # скасовує попередній run при новому push
```

Без цього при швидких послідовних pushах у PR можуть одночасно бігти 5 workflows і всі деплоїтися в staging.

---

## Jenkins {#jenkins}

### Declarative vs Scripted Pipeline

**Declarative** (рекомендований):

```groovy
// Jenkinsfile
pipeline {
    agent {
        docker {
            image 'node:20-alpine'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    environment {
        DOCKER_REGISTRY = 'registry.example.com'
        APP_NAME        = 'my-app'
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Lint & Test') {
            parallel {
                stage('Lint') {
                    steps {
                        sh 'npm run lint'
                    }
                }
                stage('Unit Tests') {
                    steps {
                        sh 'npm test -- --coverage'
                    }
                    post {
                        always {
                            junit 'test-results/**/*.xml'
                            publishCoverage adapters: [coberturaAdapter('coverage/cobertura-coverage.xml')]
                        }
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def image = docker.build("${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER}")
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'registry-credentials') {
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                sh './scripts/deploy.sh staging ${BUILD_NUMBER}'
            }
        }

        stage('Integration Tests') {
            steps {
                sh 'npm run test:e2e -- --base-url https://staging.example.com'
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            input {
                message 'Deploy to production?'
                ok 'Deploy'
                submitter 'admin,devops-team'
            }
            steps {
                sh './scripts/deploy.sh production ${BUILD_NUMBER}'
            }
        }
    }

    post {
        success {
            slackSend channel: '#deployments', color: 'good',
                      message: "✅ ${JOB_NAME} #${BUILD_NUMBER} succeeded"
        }
        failure {
            slackSend channel: '#deployments', color: 'danger',
                      message: "❌ ${JOB_NAME} #${BUILD_NUMBER} failed: ${BUILD_URL}"
        }
        always {
            cleanWs()   // очищаємо workspace після кожного запуску
        }
    }
}
```

**Scripted Pipeline** — повний Groovy, більша гнучкість, але менш читабельний і важчий в підтримці. Використовується коли Declarative не дає потрібного контролю.

### Jenkins Master-Agent архітектура

```
          ┌────────────────────┐
          │   Jenkins Master   │
          │  (Controller)      │
          │  - UI / API        │
          │  - Job scheduling  │
          │  - Plugin mgmt     │
          └────────┬───────────┘
                   │  JNLP або SSH
        ┌──────────┼──────────┐
        ▼          ▼          ▼
   ┌─────────┐ ┌─────────┐ ┌──────────────┐
   │ Agent 1 │ │ Agent 2 │ │  K8s Agent   │
   │ (Linux) │ │ (macOS) │ │  (Pod-based) │
   └─────────┘ └─────────┘ └──────────────┘
```

**Kubernetes plugin** для Jenkins — агенти запускаються як Pods у K8s і автоматично видаляються після завершення job. Золотий стандарт для масштабованих інсталяцій.

### Shared Libraries

Виносимо повторюваний Groovy-код у бібліотеку, яку шерять всі Jenkinsfiles:

```
shared-library/
├── vars/
│   └── deployToKubernetes.groovy   # global variable — викликається як функція
└── src/
    └── com/company/
        └── Docker.groovy            # клас для перевикористання
```

```groovy
// vars/deployToKubernetes.groovy
def call(Map config) {
    sh """
        helm upgrade --install ${config.releaseName} ${config.chart} \
          --namespace ${config.namespace} \
          --set image.tag=${config.imageTag}
    """
}

// Jenkinsfile
@Library('company-shared-library') _

pipeline {
    stages {
        stage('Deploy') {
            steps {
                deployToKubernetes(
                    releaseName: 'my-app',
                    chart: './helm/my-app',
                    namespace: 'production',
                    imageTag: env.BUILD_NUMBER
                )
            }
        }
    }
}
```

---

## Artifacts і кешування {#artifacts-і-кешування}

### Різниця між cache і artifact

| | Cache | Artifact |
|---|---|---|
| **Мета** | Прискорити білд (npm, pip, gradle) | Передати результат між jobs/stages |
| **Час життя** | Тижні, глобально між runs | Кілька днів, прив'язаний до run |
| **Приклад** | `node_modules/`, `.m2/` | `dist/`, `app.jar`, docker image |

```yaml
# GitHub Actions: правильний ключ кешу
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-   # fallback: використовуємо застарілий кеш
```

**Підводний камінь**: якщо ключ містить хеш файлу залежностей — при оновленні `package-lock.json` новий кеш будується з нуля. `restore-keys` дозволяє взяти найближчий старий кеш і зекономити час.

### Передача artifacts між jobs

```yaml
jobs:
  build:
    steps:
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: dist-files
          path: dist/
          retention-days: 7

  deploy:
    needs: build
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist-files
          path: dist/
      - run: ./deploy.sh
```

---

## Secrets і безпека {#secrets-і-безпека}

### Де зберігати секрети

- **GitHub Secrets** — прості key-value, шифруються через libsodium. Доступні через `${{ secrets.MY_SECRET }}`. Ніколи не логуються в stdout.
- **GitHub Environment Secrets** — секрети прив'язані до environment (production, staging), доступні тільки при деплої в цей environment.
- **HashiCorp Vault + OIDC** — enterprise рішення, секрети не зберігаються в GitHub взагалі.
- **AWS Secrets Manager / GCP Secret Manager** — хмарні сховища, інтегруються через IAM.

### OIDC замість довгоживучих ключів

Замість того, щоб класти AWS credentials в GitHub Secrets, краще використовувати OIDC:

```yaml
permissions:
  id-token: write   # дозвіл на отримання OIDC token

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789:role/GitHubActionsRole
      aws-region: us-east-1
      # Немає жодних ключів! GitHub отримує тимчасові credentials через OIDC
```

AWS перевіряє JWT-токен від GitHub і видає тимчасові credentials. Ніяких довгоживучих Access Key / Secret Key, які можуть витекти.

### Захист від secret leakage

```yaml
# Погано — значення потрапить у лог
- run: echo "Token is ${{ secrets.API_TOKEN }}"

# Добре — маскується як ***
- run: |
    echo "::add-mask::$MY_TOKEN"
    ./deploy.sh
  env:
    MY_TOKEN: ${{ secrets.API_TOKEN }}
```

GitHub автоматично маскує значення секретів у логах, але якщо ви base64-encode секрет і виводите його — маскування не спрацює.

### Least privilege у pipeline

- Кожен job повинен мати тільки ті permissions, які йому потрібні
- GITHUB_TOKEN за замовчуванням тепер `read-only` (можна змінити в налаштуваннях репо)
- Self-hosted runners не запускають код з fork PR без explicit approval

---

## Стратегії деплою {#стратегії-деплою}

### Blue-Green Deployment

Два ідентичних середовища. Нова версія деплоїться в «зелене», трафік переключається миттєво. Rollback — переключити трафік назад.

```
              ┌─────────────┐
 Users ──────►│ Load Balancer│
              └──────┬──────┘
                     │
          ┌──────────┴──────────┐
          │                     │
    ┌─────▼──────┐       ┌──────▼─────┐
    │  Blue (v1) │       │ Green (v2) │  ← активний
    │  (standby) │       │  NEW CODE  │
    └────────────┘       └────────────┘
```

**Плюси**: нульовий downtime, миттєвий rollback.  
**Мінуси**: вдвічі більше ресурсів, складніше з database migrations.

### Canary Deployment

Нова версія отримує малий відсоток трафіку (1-5%), поступово збільшується при відсутності помилок.

```yaml
# Приклад з Argo Rollouts
spec:
  strategy:
    canary:
      steps:
        - setWeight: 5      # 5% трафіку на нову версію
        - pause: {duration: 10m}
        - setWeight: 20
        - pause: {duration: 10m}
        - setWeight: 100
```

### Rolling Update

Поступова заміна старих інстансів новими. Kubernetes rolling update за замовчуванням:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # max нових pods понад desired
      maxUnavailable: 0   # жоден pod не може бути недоступний
```

### Feature Flags

Код деплоїться в прод, але нова функціональність вимкнена за флагом. Дозволяє деплоїти і релізити незалежно. Інструменти: LaunchDarkly, Unleash, Flipt.

---

## Rollback {#rollback}

### Чому rollback важливіший за деплой

«Деплоїти швидко — добре. Відкочувати ще швидше — обовʼязково.»

### Стратегії rollback

**1. Попередній image tag**
```bash
# Helm
helm rollback my-app 2   # відкат до revision 2
helm history my-app      # переглянути всі ревізії

# Kubernetes
kubectl rollout undo deployment/my-app
kubectl rollout undo deployment/my-app --to-revision=3
```

**2. Revert commit + re-deploy**
```bash
git revert HEAD~1 --no-edit
git push origin main   # тригерить CI/CD
```

**3. Feature flag вимкнення** — найшвидший метод, якщо використовуються feature flags.

### Database migrations і rollback

Найскладніша частина. Правило: кожна migration повинна бути **backward compatible** зі старою версією коду мінімум на один деплой-цикл.

**Підхід Expand-Contract (3 фази):**
1. **Expand**: додати нову колонку (стара версія продовжує писати в стару)
2. **Migrate**: обидві версії пишуть в обидві колонки
3. **Contract**: видалити стару колонку (тільки після повного деплою нової версії)

---

## Моніторинг пайплайну {#моніторинг-пайплайну}

### Метрики, які треба відстежувати

**DORA Metrics** — стандарт для оцінки ефективності DevOps:

| Метрика | Що вимірює | Elite performers |
|---------|-----------|-----------------|
| **Deployment Frequency** | Як часто деплоїте в прод | Кілька разів на день |
| **Lead Time for Changes** | Від коміту до прод | < 1 години |
| **Change Failure Rate** | % деплоїв, що спричинили інцидент | < 5% |
| **Time to Restore** | Час відновлення після інциденту | < 1 години |

### Сповіщення

```yaml
# GitHub Actions: сповіщення в Slack при падінні на main
- name: Notify Slack on failure
  if: failure() && github.ref == 'refs/heads/main'
  uses: slackapi/slack-github-action@v1
  with:
    channel-id: 'C012AB3CD'
    payload: |
      {
        "text": "❌ Pipeline failed on main",
        "blocks": [{
          "type": "section",
          "text": {
            "type": "mrkdwn",
            "text": "*Job:* ${{ github.job }}\n*Run:* ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
          }
        }]
      }
  env:
    SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

---

## GitOps і CD у Kubernetes {#gitops}

### GitOps-принцип

Git — єдине джерело правди про стан інфраструктури. Замість `kubectl apply` у пайплайні, оператор (ArgoCD, Flux) постійно звіряє стан кластеру з Git і усуває drift.

```
Developer ──► git push ──► Git Repo (desired state)
                                │
                                │  watches
                           ┌────▼─────┐
                           │  ArgoCD  │  ◄── reconciles constantly
                           └────┬─────┘
                                │ applies
                           ┌────▼─────┐
                           │    K8s   │  (actual state)
                           └──────────┘
```

### ArgoCD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/company/k8s-manifests
    targetRevision: HEAD
    path: apps/my-app/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true      # видаляти ресурси, яких немає в Git
      selfHeal: true   # відновлювати при ручних змінах у кластері
```

**Image Updater** — ArgoCD Image Updater стежить за новими тегами в registry і автоматично оновлює маніфести в Git при появі нового image.

---

## Каверзні питання на співбесіді {#каверзні-питання}

### Базовий рівень

**Q: Чим відрізняється Continuous Delivery від Continuous Deployment?**

A: Обидва забезпечують готовність артефакту до деплою після CI. Різниця — в останньому кроці: Delivery вимагає ручного тригера (апрув, клік кнопки), Deployment відбувається повністю автоматично. Continuous Deployment підходить командам з дуже зрілим покриттям тестами і observability.

---

**Q: Що таке Jenkinsfile і де він має зберігатися?**

A: Jenkinsfile — текстовий файл у корені репозиторію, що описує пайплайн у вигляді коду (Pipeline as Code). Він має зберігатися в самому репо разом з кодом, щоб версіонуватися разом з ним. Це дозволяє: переглядати зміни пайплайну через PR, тестувати зміни пайплайну в feature-гілці без впливу на main, відкочувати пайплайн разом з кодом.

---

**Q: Як запустити job тільки при пуші в `main`, але не в PR?**

```yaml
# GitHub Actions
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  deploy:
    if: github.event_name == 'push'   # не запускати в PR
    steps:
      - run: ./deploy.sh
```

---

### Середній рівень

**Q: Пайплайн завис через зависший тест. Як захиститися?**

A: Встановити timeout на рівні job/step і на рівні всього pipeline:

```yaml
# GitHub Actions
jobs:
  test:
    timeout-minutes: 15   # весь job не більше 15 хвилин
    steps:
      - run: pytest
        timeout-minutes: 10   # конкретний step

# Jenkins Declarative
options {
    timeout(time: 30, unit: 'MINUTES')
}
```

---

**Q: Як уникнути race condition при паралельних деплоях?**

A: Кілька підходів:
1. `concurrency` у GitHub Actions (cancel-in-progress або queue)
2. `disableConcurrentBuilds()` у Jenkins
3. Distributed lock (наприклад через Redis або DynamoDB) — якщо деплоять кілька систем
4. Захист через environment protection rules у GitHub

---

**Q: Як безпечно передати database URL у пайплайн?**

A: Ніколи не хардкодити в Jenkinsfile/workflow. Варіанти від гіршого до кращого:
1. **Environment variables у CI системі** (GitHub Secrets, Jenkins credentials) — прийнятно
2. **Vault + OIDC** — додаток отримує секрет у runtime, без зберігання в CI
3. **IRSA / Workload Identity** — pod отримує credentials через IAM, жодних секретів взагалі

---

**Q: Що таке matrix strategy і коли її використовувати?**

A: Matrix дозволяє запускати один і той самий job з різними комбінаціями параметрів паралельно. Типові випадки: тестування на кількох версіях мови, кількох ОС, кількох версіях залежностей.

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    python-version: ['3.11', '3.12']
  fail-fast: false   # не зупиняти інші runs при падінні одного
```

Це дає 4 паралельних jobs (2×2). `fail-fast: false` корисний коли хочемо побачити всі падіння, а не зупинятися на першому.

---

### Senior-рівень

**Q: Як організувати CI/CD для монорепо з 50 сервісами?**

A: Проблема — запускати весь пайплайн для кожного сервісу при будь-якому коміті занадто дорого. Рішення:

1. **Path filtering** — запускати job тільки якщо змінились відповідні файли:
```yaml
on:
  push:
    paths:
      - 'services/user-service/**'
      - 'shared-libs/**'
```

2. **Affected detection** — інструменти як Nx, Turborepo, Bazel розуміють граф залежностей і визначають, які сервіси «зачеплені» конкретним комітом.

3. **Dynamic matrix** — генерувати список змінених сервісів у першому job і передавати його як matrix до наступного:
```yaml
jobs:
  detect-changes:
    outputs:
      services: ${{ steps.changed.outputs.services }}
    steps:
      - id: changed
        run: |
          SERVICES=$(./scripts/detect-changed-services.sh)
          echo "services=$SERVICES" >> $GITHUB_OUTPUT

  build:
    needs: detect-changes
    strategy:
      matrix:
        service: ${{ fromJson(needs.detect-changes.outputs.services) }}
```

---

**Q: Пайплайн регулярно падає через flaky tests. Що робити?**

A: Флакі тести — окремий клас проблем. Підходи:

1. **Quarantine** — позначити флакі тести окремим тегом, запускати в окремому job з `continue-on-error: true`, не блокувати merge
2. **Retry** — перезапускати тільки флакі тести (pytest-rerunfailures, jest --testRetrys)
3. **Tracking** — вести метрики по флакі (тест A падає в 15% runs) і фіксити найгірших
4. **Root cause** — флакість зазвичай від: race conditions, залежності від часу, спільного стану між тестами, зовнішніх HTTP-запитів (мокати їх)

```yaml
# GitHub Actions: retry всього job при флакі
- uses: nick-fields/retry@v3
  with:
    timeout_minutes: 10
    max_attempts: 3
    command: npm test
```

---

**Q: Як перевіряти, що Docker image не містить критичних вразливостей?**

A: Інтегрувати сканування у пайплайн:

```yaml
- name: Scan image with Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    format: sarif
    output: trivy-results.sarif
    severity: 'CRITICAL,HIGH'
    exit-code: '1'   # зламати пайплайн при знахідці

- name: Upload to GitHub Security tab
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: trivy-results.sarif
```

Також: Snyk, Grype, Docker Scout. Найкраще сканувати base image окремо + final image, щоб розуміти звідки прийшла вразливість.

---

**Q: Чим небезпечні self-hosted runners і як їх захистити?**

A: Основні ризики:

- **Poisoned pipeline attack** — зловмисник через PR запускає код на вашому раннері з доступом до внутрішньої мережі
- **Secret exfiltration** — якщо secrets доступні в environment, шкідливий код може їх витягти
- **Persistence** — атакуючий залишає backdoor між runs

Захист:
1. Ніколи не запускати PR з форків на self-hosted раннерах без explicit approval
2. Ephemeral runners — кожен job запускається у свіжому контейнері (GitHub Actions Runner Controller в K8s)
3. Мінімальні права runner-а в мережі
4. Окремий runner pool для sensitive jobs (production deploy)

---

**Q: Як будувати пайплайн якщо один з кроків потребує 20 хвилин?**

A: Паралелізм і кешування:

1. **Паралельні jobs** — lint, unit tests, security scan паралельно, не послідовно
2. **Test splitting** — розбити тест-сьют на N частин і запускати паралельно (pytest-split, jest --shard)
3. **BuildKit cache** — Docker layer caching між runs:
```yaml
- uses: docker/build-push-action@v5
  with:
    cache-from: type=gha           # GitHub Actions cache
    cache-to: type=gha,mode=max
```
4. **Incremental builds** — Bazel, Gradle build cache, Nx computation cache
5. **Оцінити необхідність** — 20 хвилин E2E тестів на кожний PR? Можливо, запускати тільки на main

---

**Q: Як зробити zero-downtime deployment для stateful сервісу з PostgreSQL?**

A: Це складна задача без срібної кулі:

1. **Migration першою, код потім** — запускати migration окремим job до деплою нового коду. Migration повинна бути backward compatible.
2. **Expand-Contract pattern** — ніколи не дропати/перейменовувати колонки в одній migration разом з кодом. Три фази.
3. **Connection draining** — preStop hook у Kubernetes дає час завершити поточні запити:
```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 15"]
```
4. **PodDisruptionBudget** — гарантує, що мінімум N pods доступні під час rolling update:
```yaml
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-app
```

---

**Q: Чим відрізняється `git merge` від `git rebase` в контексті CI/CD?**

A: В CI/CD більш важлива стратегія гілкування і яка стратегія merge використовується:

- **Merge commit** — зберігає всю історію, чіткий `Merge branch 'feature'` коміт. Легше розслідувати коли щось зламалось.
- **Squash merge** — всі коміти з PR → один коміт. Чистіша лінійна історія на main, але втрачається деталізація.
- **Rebase** — лінійна історія без merge commits. Проблема: rebase переписує SHA, тому ніколи не rebase публічних гілок.

Для CI важливо: якщо ви squash — тег артефакту має бути SHA кінцевого коміту на main, не SHA feature branch.

---

**Q: Jenkins pipeline або GitHub Actions — що вибрати і чому?**

A: Не «або», а «залежно від контексту»:

| Критерій | GitHub Actions | Jenkins |
|----------|---------------|---------|
| Хостинг | SaaS (або self-hosted) | Self-hosted |
| Інтеграція з GitHub | Нативна | Через плагін |
| Кастомізація | Обмежена workflow DSL | Повний Groovy/Java API |
| Enterprise features | GitHub Enterprise | Дуже гнучко |
| On-prem вимоги | Self-hosted runners | Так |
| Вартість | Per-minute billing | Інфраструктура |
| Екосистема | Marketplace з тисячами actions | 1800+ плагінів |

**Відповідь на співбесіді**: «Для нових проектів на GitHub — Actions через нативну інтеграцію і zero ops. Для legacy enterprise з on-prem, складними approval workflows і потребою в Shared Libraries — Jenkins. На великому масштабі обидва можуть поєднуватися.»

---

### Про підводні камені

**Q: Назвіть 3 найчастіші помилки в CI/CD пайплайнах**

A:

1. **Секрети в коді або логах** — `echo $API_KEY` у step, хардкод credentials у Dockerfile ARG, які потрапляють у image history.

2. **Відсутність ідемпотентності деплою** — якщо деплой запустили двічі (через retry або паралельний run), другий запуск не повинен ламати перший. Helm upgrade ідемпотентний, `kubectl apply` ідемпотентний, `kubectl create` — ні.

3. **Ігнорування flaky tests замість фікса** — додавання `--retries 3` ховає проблему. Флакі тест — це симптом реальної проблеми: race condition, нечищений стан між тестами, неправильний mock.

---

## Підсумок

CI/CD — це не інструмент, а **культура і набір практик**. Найкращий пайплайн марний без:

- Тестів, які мають сенс і не флакають
- Команди, яка дійсно читає сповіщення про падіння
- Процесу rollback, який відпрацьовується заздалегідь, а не в момент інциденту
- Моніторингу після деплою (deployment markers у Grafana, error rate в Sentry)

На співбесіді важливо показати не тільки знання YAML-синтаксису, а й **розуміння трейдофів**: коли Jenkins краще за Actions, чому canary краще за big bang, як database migrations вписуються в zero-downtime деплой.

---

