+++
title = "Git зсередини: GitOps, Git Flow, Cherry-pick, Trunk-based"
date = 2022-06-14
draft = false
tags = ["git"]
categories = ["DevOps"]
description = "Повний розбір Git-стратегій: Git Flow, GitHub Flow, Trunk-Based Development, cherry-pick, reflog, detached HEAD"
+++
---

## Що таке GitOps

**GitOps** — це операційна модель, де Git є єдиним джерелом правди (single source of truth) для інфраструктури та застосунків. Будь-яка зміна в системі проходить через PR/MR у репозиторій, а не через ручні команди на сервері.

Ключові принципи:

- **Декларативність** — стан системи описаний у Git (Kubernetes manifests, Terraform, Helm charts)
- **Версійність** — кожен `git commit` = знімок стану інфраструктури
- **Автоматична синхронізація** — агент (ArgoCD, Flux) постійно порівнює бажаний стан (Git) з реальним (кластер) і вирівнює їх
- **Аудит** — `git log` — це повний журнал операцій

**Різниця GitOps від CI/CD:**

| | CI/CD (push-based) | GitOps (pull-based) |
|---|---|---|
| Хто деплоїть | Pipeline штовхає артефакт | Агент тягне зміни з Git |
| Доступ до кластера | У pipeline є credentials | Тільки агент всередині кластера |
| Rollback | Перезапустити старий pipeline | `git revert` або `git reset` |
| Drift detection | Немає | Агент постійно перевіряє |

---

## Git Flow

**Git Flow** — стратегія гілок Вінсента Дрієссена (2010). Підходить для проєктів із запланованими релізами.

### Структура гілок

```
main          ──●──────────────────────────────●──
                │                              │
release/1.0   ──●──bugfix──●────────────────●──│──
                            │                  │
develop       ──●───────────●──────────────────●──
                │           │
feature/A     ──●───────────●
```

**Постійні гілки:**
- `main` (або `master`) — тільки production-ready код, кожен коміт тегується версією
- `develop` — інтеграційна гілка, звідси виходять релізи

**Тимчасові гілки:**
- `feature/*` — відгалужується від `develop`, зливається назад у `develop`
- `release/*` — відгалужується від `develop`, зливається в `main` І `develop`
- `hotfix/*` — відгалужується від `main`, зливається в `main` І `develop`

### Команди

```bash
# Початок нової фічі
git checkout develop
git checkout -b feature/my-feature

# Завершення фічі
git checkout develop
git merge --no-ff feature/my-feature   # --no-ff зберігає історію merge commit
git branch -d feature/my-feature

# Початок релізу
git checkout develop
git checkout -b release/1.2.0

# Завершення релізу
git checkout main
git merge --no-ff release/1.2.0
git tag -a v1.2.0 -m "Release 1.2.0"
git checkout develop
git merge --no-ff release/1.2.0        # важливо: merge і в develop!
git branch -d release/1.2.0

# Hotfix
git checkout main
git checkout -b hotfix/critical-bug
# ... fix ...
git checkout main
git merge --no-ff hotfix/critical-bug
git tag -a v1.2.1 -m "Hotfix 1.2.1"
git checkout develop
git merge --no-ff hotfix/critical-bug
```

### Коли використовувати

✅ Продукт із фіксованими версіями (мобільні додатки, бібліотеки, desktop software)
✅ Довгий QA-цикл перед релізом
✅ Підтримка кількох версій одночасно

❌ Мікросервіси з continuous delivery
❌ Маленькі команди — надто багато overhead

---

## GitHub Flow

Простіша модель від GitHub. Одна довгострокова гілка `main`, решта — короткоживучі feature-гілки.

```
main     ──●─────────────────────────────●──
            │                            │
feature-A ──●──●──●──●── PR/Review ──────●
```

### Процес

```bash
# 1. Створити гілку від main
git checkout main
git pull
git checkout -b feature/add-login

# 2. Коміти
git add .
git commit -m "feat: add OAuth login"

# 3. Push і відкрити PR
git push -u origin feature/add-login

# 4. Після approve — merge у main (через UI або CLI)
git checkout main
git merge feature/add-login
git push

# 5. Deploy одразу після merge
```

### Ключові відмінності від Git Flow

| | Git Flow | GitHub Flow |
|---|---|---|
| Постійних гілок | 2 (main + develop) | 1 (main) |
| Цикл релізу | Запланований | Continuous delivery |
| Складність | Висока | Низька |
| Hotfix | Окрема гілка hotfix/* | Просто нова feature-гілка |

---

## Trunk-Based Development (TBD)

**TBD** — розробники інтегрують код у `main` (trunk) мінімум раз на день. Feature branches живуть максимум 1-2 дні.

### Чому це важливо

Довгоживучі гілки = merge hell. Чим довше гілка існує окремо, тим більший divergence від main, тим складніший merge.

TBD вирішує це радикально: немає довгоживучих гілок → немає merge hell.

### Feature Flags замість гілок

Код фічі може потрапити в main, але бути вимкненим через feature flag:

```python
if feature_flags.is_enabled("new_checkout_flow", user_id):
    return new_checkout()
else:
    return legacy_checkout()
```

```bash
# Типовий TBD workflow
git checkout main
git pull
# Змінюємо код (невеликий батч)
git add .
git commit -m "feat: add payment gateway stub behind flag"
git push

# CI запускається негайно, deploy якщо тести пройшли
```

### Порівняння стратегій

| | Git Flow | GitHub Flow | Trunk-Based |
|---|---|---|---|
| Кількість гілок | Багато | Мало | Мінімум |
| Час life-time гілки | Тижні | Дні | Години-дні |
| Merge конфлікти | Часто | Рідко | Дуже рідко |
| CI/CD | Складний | Простий | Обов'язковий |
| Підходить для | Versioned releases | Web apps | Великі команди, SaaS |
| Приклад | GitLab, Linux kernel | GitHub самого GitHub | Google, Facebook |

---

## Cherry-pick

`git cherry-pick` — застосовує зміни з конкретного коміту (або кількох) до поточної гілки. Git бере diff того коміту і застосовує його як новий коміт.

### Базовий синтаксис

```bash
# Взяти один коміт
git cherry-pick <commit-hash>

# Взяти кілька комітів
git cherry-pick abc123 def456 ghi789

# Діапазон комітів (виключаючи A, включаючи B)
git cherry-pick A..B

# Діапазон (включаючи A і B)
git cherry-pick A^..B

# Cherry-pick без автоматичного коміту (щоб переглянути зміни)
git cherry-pick -n <hash>   # або --no-commit

# Зберегти оригінальний timestamp і автора
git cherry-pick -x <hash>   # додає "cherry picked from commit..." у повідомлення
```

### Реальний сценарій: hotfix

```bash
# Знайшли критичний баг у main, виправили в feature-branch
# Хочемо застосувати тільки цей fix до release/2.0

git log feature/bugfix --oneline
# a1b2c3d fix: prevent null pointer in payment handler
# e4f5g6h wip: new UI components (НЕ хочемо цей)

git checkout release/2.0
git cherry-pick a1b2c3d
```

### Конфлікти при cherry-pick

```bash
git cherry-pick <hash>
# CONFLICT (content): Merge conflict in src/payment.py
# error: could not apply a1b2c3d

# Вирішуємо конфлікт вручну
vim src/payment.py
git add src/payment.py
git cherry-pick --continue

# Або скасувати
git cherry-pick --abort
```

### Коли НЕ використовувати cherry-pick

❌ Як замінник merge/rebase — якщо потрібно взяти всю гілку, використовуй merge

❌ Для переносу великої кількості комітів — краще rebase

❌ Якщо є залежності між комітами — cherry-pick бере коміт без його "батьків", може зламатися контекст

### Cherry-pick vs Rebase

```bash
# Cherry-pick: беремо конкретні коміти, вони стають НОВИМИ комітами
# (новий hash, навіть якщо diff той самий)

# Rebase: переносимо всю гілку поверх іншої бази
git rebase main feature/my-feature
# Еквівалентно cherry-pick всіх комітів feature/my-feature поверх main
```

---

## git reflog — чорна скринька Git

`reflog` (reference log) — це журнал всіх переміщень HEAD. Зберігається локально, не синхронізується через `push`.

```bash
git reflog
# або
git reflog show HEAD

# Приклад виводу:
# a1b2c3d HEAD@{0}: commit: feat: add login
# e4f5g6h HEAD@{1}: checkout: moving from develop to feature/login
# d7c8b9a HEAD@{2}: reset --hard: moving to d7c8b9a
# f0e1d2c HEAD@{3}: merge hotfix/fix: Fast-forward
# 9a8b7c6 HEAD@{4}: commit: wip: half-done changes
```

### Синтаксис посилань

```bash
HEAD@{0}   # поточна позиція HEAD
HEAD@{1}   # де HEAD був 1 операцію тому
HEAD@{2}   # 2 операції тому
HEAD@{1 hour ago}   # де HEAD був годину тому
HEAD@{yesterday}
HEAD@{2026-06-15 10:00:00}
```

### Сценарій 1: Відновити після "випадкового" reset --hard

```bash
# Ой, зробили reset --hard і втратили 3 коміти
git reset --hard HEAD~3

# Знаходимо загублені коміти через reflog
git reflog
# a1b2c3d HEAD@{0}: reset: moving to HEAD~3
# lost1234 HEAD@{1}: commit: feat: payment integration   ← ось він
# lost5678 HEAD@{2}: commit: feat: checkout page
# lost9abc HEAD@{3}: commit: feat: cart logic

# Відновлюємо
git reset --hard lost1234   # або HEAD@{1}
```

### Сценарій 2: Знайти видалену гілку

```bash
# Видалили гілку feature/important
git branch -D feature/important

git reflog
# знаходимо останній коміт тієї гілки
# deadbeef HEAD@{5}: commit: last commit on feature/important

git checkout -b feature/important deadbeef
```

### Сценарій 3: Відновити після поганого rebase

```bash
git rebase main   # щось пішло не так
git reflog
# ORIG_HEAD зберігає стан до небезпечних операцій
git reset --hard ORIG_HEAD
```

### Налаштування reflog

```bash
# Reflog зберігається 90 днів (за замовчуванням)
git config gc.reflogExpire "180 days"   # збільшити до 180
git config gc.reflogExpireUnreachable "30 days"

# Перегляд reflog конкретної гілки
git reflog show main
git reflog show origin/main
```

---

## Detached HEAD — стан "відірваної голови"

Один із найпоширеніших джерел паніки у розробників.

### Що таке HEAD

`HEAD` — це вказівник на поточний коміт. Зазвичай HEAD вказує на гілку (через symbolic reference), а та гілка вказує на коміт:

```
HEAD → main → commit a1b2c3d
```

### Що таке Detached HEAD

HEAD вказує безпосередньо на коміт, а не на гілку:

```
HEAD → commit a1b2c3d   (минаючи будь-яку гілку)
```

```bash
$ git checkout a1b2c3d
# або
$ git checkout v1.0.0   # тег не є гілкою
# або
$ git checkout origin/main   # remote ref

# Git скаже:
# You are in 'detached HEAD' state. You can look around, make experimental
# changes and commit them, and you can discard any commits you make in this
# state without impacting any branches by switching back to a branch.
```

### Чому це небезпечно

Якщо зробити `commit` у detached HEAD і потім переключитися на гілку — новий коміт стає "unreachable" і буде видалений garbage collector через 30 днів.

```bash
# Небезпечний сценарій
git checkout a1b2c3d   # detached HEAD
# ... редагуємо файли ...
git commit -m "important fix"   # новий коміт orphan1234

git checkout main   # переходимо на гілку

# orphan1234 тепер unreachable! 
# Git попередить: "warning: detached HEAD"
```

### Як виправити: зберегти роботу з detached HEAD

```bash
# Варіант 1: створити нову гілку з поточної позиції
git checkout -b my-saved-work

# Варіант 2: якщо вже переключились і втратили коміт
git reflog
# orphan1234 HEAD@{2}: commit: important fix
git checkout -b recovered-work orphan1234

# Варіант 3: cherry-pick в існуючу гілку
git cherry-pick orphan1234
```

### Законні використання Detached HEAD

```bash
# Переглянути старий стан коду (read-only дослідження)
git checkout v2.1.0
# ... дивимось код, не змінюємо ...
git checkout main   # повертаємось

# Bisect (автоматичне) — git сам переходить у detached HEAD
git bisect start
git bisect bad HEAD
git bisect good v1.0.0
# git тестує коміти у detached HEAD
```

---

## Каверзні команди — що, де, чому

### git reset vs git revert vs git restore

```bash
# git reset — переміщує HEAD (і гілку) назад
git reset --soft HEAD~1    # скасовує коміт, зміни лишаються у staging
git reset --mixed HEAD~1   # (default) скасовує коміт, зміни лишаються у working dir
git reset --hard HEAD~1    # скасовує коміт, зміни ВИДАЛЯЮТЬСЯ (небезпечно!)

# git revert — створює НОВИЙ коміт, що скасовує зміни (safe для public history)
git revert HEAD            # скасовує останній коміт
git revert a1b2c3d         # скасовує конкретний коміт
git revert --no-commit HEAD~3..HEAD  # підготувати revert кількох комітів без автокоміту

# git restore — відновлює файли (не чіпає коміти)
git restore file.txt           # відновити з staging/HEAD
git restore --staged file.txt  # прибрати з staging (unstage)
git restore --source HEAD~2 file.txt  # відновити файл з 2 комітів тому
```

**Правило:** `reset` — для локальної history (ще не запушили). `revert` — для публічної history (вже в remote).

### git stash

```bash
git stash                      # сховати незбережені зміни
git stash push -m "wip: auth" # з іменем
git stash list                 # переглянути всі stash
git stash apply stash@{0}      # застосувати (stash залишається)
git stash pop                  # застосувати і видалити
git stash drop stash@{1}       # видалити конкретний
git stash clear                # видалити всі

# Stash тільки staged змін
git stash push --staged

# Stash включаючи untracked файли
git stash push -u
```

### git rebase — інтерактивний

```bash
# Переписати останні 3 коміти
git rebase -i HEAD~3

# У відкритому редакторі:
# pick a1b2c3d feat: login
# pick b2c3d4e fix: typo
# pick c3d4e5f wip: half done

# Команди:
# pick   — залишити як є
# reword — змінити повідомлення коміту
# edit   — зупинитись для редагування
# squash — з'єднати з попереднім (зберегти обидва повідомлення)
# fixup  — з'єднати з попереднім (викинути це повідомлення)
# drop   — видалити коміт
# exec   — виконати shell команду

# Після збереження git виконає rebase
```

### git bisect — бінарний пошук бага

```bash
git bisect start
git bisect bad                  # поточний стан — баг є
git bisect good v2.0.0          # тут ще не було

# Git переходить у detached HEAD на середній коміт
# Тестуємо:
git bisect good   # або
git bisect bad

# Повторюємо до знаходження першого поганого коміту
# Git виведе: "a1b2c3d is the first bad commit"

git bisect reset   # вийти з bisect mode

# Автоматичний bisect зі скриптом
git bisect run ./test.sh   # 0 = good, 1 = bad
```

### git worktree — кілька робочих директорій

```bash
# Перевіряти PR без stash/switch
git worktree add ../hotfix-review hotfix/critical-bug
cd ../hotfix-review   # окрема директорія, та ж репо
# ... перевіряємо ...
git worktree remove ../hotfix-review
```

---

### "Яка різниця між merge і rebase?"

**Merge** зберігає повну історію і створює merge commit:
```
A - B - C - D (main)
     \       \
      E - F - G (merge commit)
```

**Rebase** переписує коміти поверх нової бази, лінійна історія:
```
A - B - C - E' - F' (main, після rebase)
```

`E'` і `F'` — нові коміти (інший hash), навіть якщо diff той самий.

**Золоте правило rebase:** ніколи не rebase публічні гілки (ті, що вже запушені і хтось ще використовує).

---

### "Що таке fast-forward merge і коли він неможливий?"

Fast-forward відбувається, коли гілка є прямим продовженням main — немає розходжень, Git просто пересуває вказівник:

```bash
main:    A - B
feature:         C - D

# Fast-forward: main → A - B - C - D
git merge feature   # автоматично FF якщо можливо
git merge --ff-only feature   # примусово тільки FF, інакше помилка
git merge --no-ff feature     # завжди merge commit
```

FF неможливий, якщо main мав нові коміти після відгалуження feature.

---

### "Що відбувається при git push --force?"

`push --force` перезаписує remote гілку поточним локальним станом. Якщо хтось інший запушив після тебе — його коміти зникнуть.

```bash
# Безпечна альтернатива:
git push --force-with-lease   # відмовить якщо remote змінився

# force-with-lease під капотом:
# перевіряє, чи remote ref збігається з тим, що ти бачив (fetch)
# якщо хтось запушив між твоїм fetch і push — помилка
```

---

### "Як знайти коміт, який зламав функціональність?"

Три підходи:
1. `git bisect` — бінарний пошук O(log n)
2. `git log -S "function_name"` — знайти коміти, що змінили рядок
3. `git blame file.py` — хто і коли змінив кожен рядок

```bash
# git log -S (pickaxe search)
git log -S "calculate_tax" --oneline

# git log -G (regex пошук)
git log -G "def.*payment" --oneline -p

# git blame
git blame -L 40,60 src/payment.py   # рядки 40-60
```

---

### "Що таке git tag і яка різниця між lightweight та annotated?"

```bash
# Lightweight tag — просто вказівник на коміт (як гілка, але не рухається)
git tag v1.0.0

# Annotated tag — повноцінний об'єкт у Git з автором, датою, повідомленням
git tag -a v1.0.0 -m "Release 1.0.0"

# Annotated теги рекомендовані для релізів:
# - мають власний hash
# - зберігаються як окремі об'єкти
# - можуть бути підписані GPG (git tag -s)

# Push тегів (за замовчуванням push не відправляє теги)
git push origin v1.0.0
git push origin --tags   # всі теги
```

---

### "Що відбувається з комітами після git branch -d?"

Коміти не видаляються одразу. Вони стають "unreachable" — на них не вказує жодна гілка/тег. Git Garbage Collector (`git gc`) видалить їх через ~30 днів.

До того часу їх можна знайти через `git reflog` або `git fsck --unreachable`.

---

### "Яка різниця між origin/main і main?"

```bash
git branch -a
# main               ← локальна гілка
# remotes/origin/main ← remote-tracking гілка (локальний кеш стану remote)

# origin/main оновлюється тільки при fetch/pull
git fetch   # оновлює origin/main без merge у локальний main
git pull    # = git fetch + git merge origin/main
```

---

### "Що таке ORIG_HEAD, FETCH_HEAD, MERGE_HEAD?"

```bash
ORIG_HEAD  # стан HEAD до небезпечної операції (merge, rebase, reset)
           # git reset --hard ORIG_HEAD → скасувати останній merge/rebase

FETCH_HEAD # що fetch принесло (останній fetch)
           # git merge FETCH_HEAD → зробити те, що робить pull

MERGE_HEAD # в процесі merge — вказує на коміт, який зливається
           # існує тільки під час конфлікту

CHERRY_PICK_HEAD # аналогічно для cherry-pick в конфліктному стані
REBASE_HEAD      # під час rebase
```

---

### "Як зробити Git коміт порожнім?"

```bash
git commit --allow-empty -m "trigger CI"
# Корисно щоб запустити CI pipeline без реальних змін
```

---

### "Що таке git submodule і коли використовувати?"

```bash
# Додати зовнішній репозиторій як підмодуль
git submodule add https://github.com/vendor/lib vendor/lib
git submodule update --init --recursive   # ініціалізувати після clone

# Проблеми:
# - submodule вказує на конкретний коміт, не на гілку
# - легко забути оновити після змін у submodule
# - clone без --recurse-submodules дає порожні директорії

# Альтернативи: git subtree, package managers (npm, pip, go modules)
```

---

### "Як перевірити, які файли зміниться при merge?"

```bash
# Переглянути diff перед merge
git diff main..feature/my-feature

# Тільки назви файлів
git diff --name-only main..feature/my-feature

# Симулювати merge без виконання
git merge --no-commit --no-ff feature/my-feature
git diff --cached   # переглянути staged зміни
git merge --abort   # скасувати симуляцію
```

---

## Шпаргалка: git log — фільтрація та форматування

```bash
# Базові фільтри
git log --oneline --graph --all           # дерево гілок
git log --author="Andrii" --oneline       # коміти конкретного автора
git log --since="2 weeks ago" --until="yesterday"
git log --grep="hotfix" --oneline         # пошук у повідомленнях
git log -- src/payment.py                 # коміти що чіпали файл

# Форматування
git log --pretty=format:"%h %an %ar %s"  # hash, author, relative date, subject
git log --pretty=format:"%C(yellow)%h%C(reset) %C(green)%ar%C(reset) %s"

# Статистика
git log --stat                            # змінені файли і рядки
git log --shortstat                       # тільки підсумок
git log -p                                # повний diff кожного коміту
git log -p -S "function_name"            # diff де зустрічається рядок

# Знайти коміт за вмістом
git log --all --full-history -- "**/payment*"  # видалені файли теж
```

---

## Підсумок: вибір стратегії

```
Ти у великій команді + SaaS + CI/CD є?
  └─► Trunk-Based Development

Продукт з версіями + довгий QA + команда 5-20?
  └─► Git Flow

Невелика команда + веб + continuous deploy?
  └─► GitHub Flow

GitOps + Kubernetes?
  └─► GitHub Flow або TBD + ArgoCD/Flux
```

Немає "правильної" стратегії — є контекст. Головне: коротколивучі гілки, часті інтеграції, автоматичне тестування. Решта — деталі.

---
+++
