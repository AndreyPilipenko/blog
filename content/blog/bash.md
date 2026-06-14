---
title: "eBash"
date: 2026-06-14
draft: false
tags: ["bash", "linux"]
description: "Вичерпний гід для підготовки до співбесіди по Bash: синтаксис, пастки, відлагодження, коди виходу, каверзні питання з поясненнями та реальними прикладами."
---

---

## Зміст

1. [Основи, які обов'язково знати](#основи)
2. [Змінні та підстановки](#змінні)
3. [Умови та тести](#умови)
4. [Цикли](#цикли)
5. [Функції](#функції)
6. [Обробка помилок та коди виходу](#коди-виходу)
7. [Перенаправлення та pipe](#перенаправлення)
8. [Масиви та асоціативні масиви](#масиви)
9. [Рядкові операції](#рядки)
10. [Процеси та сигнали](#процеси)
11. [Відлагодження](#відлагодження)
12. [Каверзні питання](#каверзні-питання)
13. [Реальні задачі зі співбесід](#задачі)

---

## Основи {#основи}

### Shebang та режими запуску

```bash
#!/bin/bash        # Явний bash
#!/usr/bin/env bash # Кращий варіант — шукає bash в $PATH (portable)
#!/bin/sh          # POSIX sh — без bashisms!
```

**Різниця між `bash script.sh`, `./script.sh` і `source script.sh`:**

```bash
bash script.sh    # Запускає в дочірньому процесі, змінні не впливають на поточний shell
./script.sh       # Те саме, але потрібен shebang + chmod +x
source script.sh  # Виконує В ПОТОЧНОМУ shell! Змінні, cd — все впливає
. script.sh       # Те саме, що source (POSIX-сумісний синтаксис)
```

> **Питання:** Чому `cd` у скрипті не змінює директорію в терміналі?
> **Відповідь:** Скрипт виконується в subshell. Після завершення subshell зникає разом зі своїм `$PWD`. Використовуйте `source` або функцію.

### Спеціальні змінні оточення

```bash
$0    # Ім'я скрипта
$1–$9 # Позиційні параметри
$@    # Всі аргументи як окремі слова (зберігає пробіли!)
$*    # Всі аргументи як один рядок (не використовуйте)
$#    # Кількість аргументів
$?    # Код виходу останньої команди
$$    # PID поточного процесу
$!    # PID останнього фонового процесу
$_    # Останній аргумент попередньої команди
$-    # Поточні флаги shell (himBHs...)
$IFS  # Internal Field Separator (за замовчуванням: пробіл, таб, новий рядок)
```

---

## Змінні та підстановки {#змінні}

### Присвоєння та scope

```bash
# ВАЖЛИВО: без пробілів навколо =
VAR="hello"     # OK
VAR = "hello"   # ПОМИЛКА — bash спробує виконати команду 'VAR'

# local — тільки всередині функції
my_func() {
    local var="local"   # не виходить за межі функції
    global_var="global" # доступна ззовні!
}

# export — передає в дочірні процеси
export PATH="$PATH:/opt/bin"
export -f my_func  # Експортувати функцію!

# readonly — незмінна
readonly CONFIG_FILE="/etc/app/config"
CONFIG_FILE="other"  # bash: CONFIG_FILE: readonly variable
```

### Parameter Expansion — найважливіше

```bash
VAR="Hello World"

# Базові
${VAR}            # "Hello World" (явні межі)
${#VAR}           # 11 (довжина рядка)

# Значення за замовчуванням
${VAR:-default}   # якщо VAR порожня або не встановлена → "default" (VAR не змінюється)
${VAR:=default}   # якщо VAR порожня → встановлює VAR="default"
${VAR:+other}     # якщо VAR встановлена → "other" (інвертоване)
${VAR:?error msg} # якщо VAR порожня → виводить помилку і виходить

# Підрядки
${VAR:0:5}        # "Hello" (від позиції 0, довжина 5)
${VAR:6}          # "World" (від позиції 6 до кінця)
${VAR: -5}        # "World" (УВАГА: пробіл перед -5 обов'язковий!)

# Видалення підрядків (globbing)
FILE="archive.tar.gz"
${FILE#*.}        # "tar.gz"  (видалити найкоротший збіг з початку)
${FILE##*.}       # "gz"      (видалити найдовший збіг з початку)
${FILE%.*}        # "archive.tar" (видалити найкоротший збіг з кінця)
${FILE%%.*}       # "archive" (видалити найдовший збіг з кінця)

# Заміна
${VAR/World/Bash} # "Hello Bash" (перше входження)
${VAR//l/L}       # "HeLLo WorLd" (всі входження)
${VAR/#Hello/Hi}  # "Hi World" (тільки на початку)
${VAR/%World/!}   # "Hello !" (тільки в кінці)

# Зміна регістру (bash 4+)
${VAR,,}          # "hello world" (все малими)
${VAR^^}          # "HELLO WORLD" (все великими)
${VAR,}           # "hello World" (перша буква малою)
${VAR^}           # "Hello World" (перша буква великою)
```

### Підстановка команд

```bash
# Сучасний синтаксис (рекомендований, підтримує вкладення)
NOW=$(date +%Y-%m-%d)
FILES=$(find /var/log -name "*.log" -newer $(date -d "7 days ago" +%Y-%m-%d))

# Старий синтаксис з backticks (уникайте)
NOW=`date +%Y-%m-%d`  # Важко читати, погано вкладається

# Арифметика
COUNT=$((10 + 5 * 2))   # 20
echo $((2**10))          # 1024
echo $((16#FF))          # 255 (hex → decimal)
echo $((8#77))           # 63  (octal → decimal)
```

---

## Умови та тести {#умови}

### `[`, `[[` і `(( ))` — різниця принципова

```bash
# [ ] — POSIX, виклик команди /usr/bin/[ або shell builtin
# Потребує лапок навколо змінних!
[ "$VAR" = "value" ]   # OK
[ $VAR = "value" ]     # НЕБЕЗПЕЧНО: якщо VAR містить пробіли — помилка

# [[ ]] — bash keyword (не команда), безпечніше
# Не потребує лапок для більшості випадків
# Підтримує && || без екранування
# Підтримує regex =~
[[ $VAR = "value" ]]   # OK, навіть без лапок
[[ $VAR == val* ]]     # Glob-matching (не regex!)
[[ $VAR =~ ^[0-9]+$ ]] # Regex! Без лапок навколо pattern

# (( )) — арифметичні умови
(( 5 > 3 ))           # true (exit 0)
(( count++ ))         # increment і перевірка (0 = false в арифметиці!)
```

### Флаги тестів

```bash
# Файлові тести
[ -e FILE ]   # exists (будь-який тип)
[ -f FILE ]   # regular file
[ -d FILE ]   # directory
[ -L FILE ]   # symbolic link
[ -r FILE ]   # readable
[ -w FILE ]   # writable
[ -x FILE ]   # executable
[ -s FILE ]   # size > 0 (не порожній)
[ -S FILE ]   # socket
[ -p FILE ]   # named pipe
[ FILE1 -nt FILE2 ]  # newer than
[ FILE1 -ot FILE2 ]  # older than
[ FILE1 -ef FILE2 ]  # same inode (hardlink)

# Рядкові тести
[ -z "$STR" ]   # zero length (порожній)
[ -n "$STR" ]   # non-zero length (не порожній)
[ "$A" = "$B" ] # рівні (POSIX)
[ "$A" != "$B"] # не рівні

# Числові тести (тільки цілі числа!)
[ "$A" -eq "$B" ]  # equal
[ "$A" -ne "$B" ]  # not equal
[ "$A" -lt "$B" ]  # less than
[ "$A" -le "$B" ]  # less or equal
[ "$A" -gt "$B" ]  # greater than
[ "$A" -ge "$B" ]  # greater or equal
```

### case statement

```bash
case "$1" in
    start|begin)
        echo "Starting..."
        ;;
    stop|halt)
        echo "Stopping..."
        ;;
    [0-9]*)             # glob pattern!
        echo "Got a number: $1"
        ;;
    *)
        echo "Unknown: $1"
        exit 1
        ;;
esac

# bash 4.0+: ;& (fall-through), ;;& (continue matching)
case "$char" in
    [A-Z])
        echo "Upper"
        ;;&   # продовжити перевірку наступних патернів!
    [A-Za-z])
        echo "Letter"
        ;;
esac
```

---

## Цикли {#цикли}

### for loop

```bash
# C-style (арифметика)
for ((i=0; i<10; i++)); do
    echo "$i"
done

# Ітерація по словах
for item in one two three; do
    echo "$item"
done

# Ітерація по файлах (правильний спосіб)
for file in /var/log/*.log; do
    [[ -f "$file" ]] || continue  # захист від порожнього glob
    echo "Processing: $file"
done

# Ітерація по масиву
arr=("a" "b" "c")
for item in "${arr[@]}"; do  # Лапки обов'язкові!
    echo "$item"
done

# По рядках файлу (ПРАВИЛЬНО)
while IFS= read -r line; do
    echo "Line: $line"
done < /etc/passwd

# По виводу команди (обережно з IFS!)
while IFS= read -r line; do
    echo "$line"
done < <(find /etc -name "*.conf" 2>/dev/null)
# < <(...) — process substitution, не subshell як | while
```

### while та until

```bash
# while: виконується поки умова true
count=0
while (( count < 5 )); do
    echo "$count"
    (( count++ ))
done

# until: виконується поки умова false
until ping -c1 -W1 8.8.8.8 &>/dev/null; do
    echo "Waiting for network..."
    sleep 5
done

# Нескінченний цикл з break
while true; do
    read -r input
    [[ "$input" == "quit" ]] && break
    echo "You said: $input"
done

# continue — пропустити ітерацію
for i in {1..10}; do
    (( i % 2 == 0 )) && continue  # пропустити парні
    echo "Odd: $i"
done
```

> **Каверза:** `while read` всередині pipe виконується в subshell — змінні не видно ззовні!
> ```bash
> # НЕПРАВИЛЬНО:
> cat file.txt | while read line; do count=$((count+1)); done
> echo $count  # 0! subshell
>
> # ПРАВИЛЬНО:
> while read line; do count=$((count+1)); done < file.txt
> echo $count  # правильне значення
> ```

---

## Функції {#функції}

```bash
# Два еквівалентних синтаксиси
my_func() { ... }      # POSIX
function my_func { ... } # Bash-specific

# Параметри функції — як скрипта: $1, $2, $@
# Але $0 — це ім'я скрипта, не функції!
greet() {
    local name="${1:-World}"  # default value
    echo "Hello, $name!"
    return 0  # return — тільки для числового exit code (0-255)!
              # Не для повернення рядків!
}

# Повернення значень з функції
get_hostname() {
    # Варіант 1: через stdout (найпоширеніший)
    hostname -f
}
FQDN=$(get_hostname)

get_user_count() {
    # Варіант 2: через глобальну змінну (побічний ефект)
    USER_COUNT=$(wc -l < /etc/passwd)
}

# Варіант 3: через nameref (bash 4.3+)
get_value() {
    local -n result_ref=$1  # nameref!
    result_ref="the value"
}
get_value my_var
echo "$my_var"  # "the value"

# Рекурсія (обережно: bash повільний)
factorial() {
    local n=$1
    (( n <= 1 )) && echo 1 && return
    echo $(( n * $(factorial $((n-1))) ))
}
```

---

## Обробка помилок та коди виходу {#коди-виходу}

### Стандартні коди виходу

| Код | Значення |
|-----|----------|
| 0 | Успіх |
| 1 | Загальна помилка |
| 2 | Неправильне використання shell builtin |
| 126 | Команда не виконувана (немає прав) |
| 127 | Команда не знайдена |
| 128 | Невалідний аргумент для exit |
| 128+N | Завершення через сигнал N (напр. 130 = Ctrl+C / SIGINT) |
| 130 | Завершено через SIGINT (128+2) |
| 137 | Завершено через SIGKILL (128+9) — OOM killer |
| 139 | Segmentation fault (128+11) |
| 255 | Exit code out of range |

```bash
# Перевірка виходу
command
echo $?    # код виходу

# Умовне виконання
command && echo "OK" || echo "FAIL"

# Правильна перевірка (не з &&||, бо маємо три стани)
if command; then
    echo "OK"
else
    echo "FAIL: $?"
fi
```

### `set -e`, `set -u`, `set -o pipefail` — обов'язково знати

```bash
#!/bin/bash
set -euo pipefail
# set -e: виход при будь-якій помилці (exit code != 0)
# set -u: помилка при використанні невстановленої змінної
# set -o pipefail: pipe завершується з кодом першої команди що впала

# Також корисно для дебагу:
set -x  # трасування всіх команд
set -v  # вивід кожного рядка перед виконанням

IFS=$'\n\t'  # безпечний IFS (прибирає пробіл)
```

**Підводні камені `set -e`:**

```bash
set -e

# ПРОБЛЕМА: false в умові не перерве скрипт
if false; then echo "never"; fi   # OK, бо це умовний вираз
false || true                      # OK
false && true                      # OK (false — це умова)

# ПРОБЛЕМА: функції теж підпадають під set -e
might_fail() { return 1; }
might_fail   # → EXIT! якщо не обгорнути

# Правильно:
might_fail || true        # ігнорувати помилку
might_fail || handle_err  # або обробити
if might_fail; then ...; fi  # або перевірити
```

### trap — перехоплення сигналів

```bash
#!/bin/bash

# Cleanup при виході (будь-якому!)
cleanup() {
    echo "Cleaning up..."
    rm -f /tmp/myapp_lock
    # Не викликай exit тут!
}
trap cleanup EXIT  # виконається при будь-якому exit

# Обробка сигналів
handle_sigterm() {
    echo "Got SIGTERM, graceful shutdown..."
    # зупинити дочірні процеси
    kill $(jobs -p) 2>/dev/null
    exit 0
}
trap handle_sigterm SIGTERM SIGINT

# Ігнорувати сигнал
trap '' SIGHUP  # ігнорувати SIGHUP (як nohup)

# Скинути обробник до дефолтного
trap - SIGTERM

# Дебаг: вивід команди перед виконанням
trap 'echo "Line $LINENO: $BASH_COMMAND"' DEBUG

# Перехопити ERR (помилки)
trap 'echo "Error at line $LINENO, exit code: $?"' ERR

# Типовий production-ready шаблон:
set -euo pipefail
TMPDIR=$(mktemp -d)
trap 'rm -rf "$TMPDIR"' EXIT

# ... основна логіка використовує $TMPDIR ...
```

---

## Перенаправлення та pipe {#перенаправлення}

### File Descriptors

```bash
# Стандартні FD:
# 0 = stdin
# 1 = stdout
# 2 = stderr

# Базові перенаправлення
command > file     # stdout → file (перезапис)
command >> file    # stdout → file (дописати)
command 2> file    # stderr → file
command 2>&1       # stderr → куди йде stdout
command &> file    # і stdout, і stderr → file (bash 4+)
command 1>/dev/null 2>&1  # заглушити все

# ВАЖЛИВО: порядок має значення!
command 2>&1 > file   # НЕПРАВИЛЬНО: stderr йде в старий stdout (термінал)
command > file 2>&1   # ПРАВИЛЬНО: спочатку redirect stdout, потім stderr до нього
```

### Process Substitution

```bash
# < (...) — читати вивід команди як файл
diff <(sort file1.txt) <(sort file2.txt)

# > (...) — записати в процес як у файл
tee >(gzip > output.gz) >(wc -l) > /dev/null

# Чому це краще ніж pipe?
# Pipe створює subshell — змінні не видно після
while read line; do count=$((count+1)); done < <(command)
echo $count  # Працює! process substitution не subshell для батька
```

### Here Documents та Here Strings

```bash
# Here document
cat <<EOF
This is line 1
Variables: $HOME
EOF

# Без розкриття змінних (лапки навколо маркера)
cat <<'EOF'
No expansion: $HOME stays literal
EOF

# З відступом (<<-, прибирає провідні таби, не пробіли!)
cat <<-EOF
	Line with tab indent
	Another line
EOF

# Here string — подати рядок на stdin
grep "pattern" <<< "$variable"
base64 <<< "encode me"
read -r first rest <<< "one two three"
echo "$first"  # "one"
```

### Named Pipes (FIFO)

```bash
mkfifo /tmp/mypipe
command1 > /tmp/mypipe &
command2 < /tmp/mypipe
rm /tmp/mypipe
```

---

## Масиви та асоціативні масиви {#масиви}

```bash
# Indexed arrays
arr=(one two "three four" five)
echo "${arr[0]}"        # "one"
echo "${arr[@]}"        # всі елементи (окремо)
echo "${arr[*]}"        # всі елементи (один рядок з IFS)
echo "${#arr[@]}"       # 4 (кількість елементів)
echo "${!arr[@]}"       # 0 1 2 3 (індекси)
echo "${arr[@]:1:2}"    # "two" "three four" (slice)

# Додавання
arr+=(six seven)
arr[10]="ten"           # sparse array — пропуск індексів OK

# Видалення
unset arr[1]            # видалити елемент (але індекс залишиться "дірою"!)
unset arr               # видалити весь масив

# Iterate (правильно)
for item in "${arr[@]}"; do  # ЗАВЖДИ лапки!
    echo "$item"
done

# Асоціативні масиви (bash 4+)
declare -A map
map[key]="value"
map=([foo]="bar" [baz]="qux")

echo "${map[foo]}"      # "bar"
echo "${!map[@]}"       # всі ключі (порядок непередбачуваний!)
echo "${map[@]}"        # всі значення

# Перевірити наявність ключа
[[ -v map[key] ]] && echo "exists"

# Типовий use case: підрахунок
declare -A count
while read -r word; do
    (( count[$word]++ )) || true
done < words.txt
for word in "${!count[@]}"; do
    echo "${count[$word]} $word"
done | sort -rn
```

---

## Рядкові операції {#рядки}

```bash
# Довжина
str="Hello, World!"
echo ${#str}           # 13

# Підрядок (substring)
echo "${str:7:5}"      # "World"
echo "${str: -6}"      # "orld!" (від кінця, пробіл важливий!)

# Пошук і заміна
echo "${str/World/Bash}"   # "Hello, Bash!"
echo "${str//l/L}"         # "HeLLo, WorLd!"

# Видалення (prefix/suffix)
path="/usr/local/bin/bash"
echo "${path##*/}"         # "bash" (basename)
echo "${path%/*}"          # "/usr/local/bin" (dirname)

# Порівняння
[[ "abc" < "abd" ]]   # лексикографічне порівняння (тільки [[]])
[[ "abc" == ab* ]]    # glob matching

# Regex (тільки [[]])
[[ "2024-01-15" =~ ^([0-9]{4})-([0-9]{2})-([0-9]{2})$ ]]
echo "${BASH_REMATCH[0]}"  # вся відповідність: "2024-01-15"
echo "${BASH_REMATCH[1]}"  # перша група: "2024"
echo "${BASH_REMATCH[2]}"  # друга група: "01"

# Split рядка
IFS=',' read -ra parts <<< "one,two,three"
echo "${parts[1]}"  # "two"

# Trim пробілів (немає вбудованого!)
trim() {
    local var="$1"
    var="${var#"${var%%[![:space:]]*}"}"   # ltrim
    var="${var%"${var##*[![:space:]]}"}"   # rtrim
    echo "$var"
}

# printf для форматування
printf "%-20s %5d\n" "Server" 42
printf "%08.2f\n" 3.14159   # "00003.14"
printf "%b" "\x41\x42\x43"  # "ABC" (escape sequences)
```

---

## Процеси та сигнали {#процеси}

```bash
# Фонові процеси
command &           # запустити у фоні
pid=$!              # PID останнього фонового процесу
wait $pid           # чекати завершення
wait                # чекати всіх дочірніх

# Job control
sleep 100 &
jobs                # список фонових задач
fg %1               # перевести задачу 1 на передній план
bg %1               # продовжити зупинену задачу у фоні
kill %1             # вбити job 1
disown %1           # від'єднати від shell (не отримає SIGHUP)

# Таблиця сигналів (обов'язково знати)
# SIGHUP  (1)  — термінал закрито; перезавантаження конфігу (за конвенцією)
# SIGINT  (2)  — Ctrl+C
# SIGQUIT (3)  — Ctrl+\ (+ core dump)
# SIGKILL (9)  — неперехоплюваний! Ядро вбиває примусово
# SIGTERM (15) — стандартний "зупинись" (перехоплюється!)
# SIGUSR1 (10) — користувацький сигнал 1
# SIGUSR2 (12) — користувацький сигнал 2
# SIGCHLD (17) — дочірній процес змінив стан
# SIGSTOP (19) — неперехоплюваний! Зупинити процес
# SIGCONT (18) — продовжити зупинений

# Надсилання сигналів
kill -15 $PID        # SIGTERM (default)
kill -9 $PID         # SIGKILL (last resort!)
kill -HUP $PID       # SIGHUP (reload config)
kill -0 $PID         # перевірити чи процес існує (не надсилає сигнал)
killall nginx        # за ім'ям
pkill -f "python.*myapp"  # regex по cmdline

# Subshell vs child process
( cd /tmp && ls )    # subshell: зміни не впливають на батьківський shell
bash -c "cd /tmp && ls"  # child process: теж ізольований

# exec — замінює поточний процес
exec python3 myapp.py  # bash process стає python3, no return!
exec 3> /tmp/logfile   # відкрити FD 3 для запису
exec 3>&-              # закрити FD 3
```

---

## Відлагодження {#відлагодження}

### Інструменти відлагодження bash

```bash
# set -x: трасування (виводить кожну команду перед виконанням)
set -x
# ... код ...
set +x  # вимкнути

# Запуск з трасуванням без зміни скрипта
bash -x script.sh
bash -xv script.sh  # -v також виводить вихідний код

# Тільки частина скрипта
{
    set -x
    problematic_function
    set +x
} 2>&1 | tee debug.log

# Показати рядок помилки
trap 'echo "Error at line $LINENO: $BASH_COMMAND" >&2' ERR

# Повний debug output
PS4='+(${BASH_SOURCE}:${LINENO}): ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'
set -x
```

### Типові помилки та їх діагностика

```bash
# 1. "command not found" (127)
which command          # чи є в PATH?
type -a command        # де знаходиться і який тип?
echo $PATH             # перевірити PATH

# 2. Проблеми з IFS і пробілами
IFS=$'\n\t'           # безпечний IFS
for file in *; do
    echo "File: '$file'"  # лапки щоб бачити пробіли
done

# 3. Glob expansion у порожній директорії
ls /empty/*
# bash: /empty/*: No such file or directory
shopt -s nullglob     # повертає порожній список замість помилки

# 4. Перевірка типу змінної
declare -p VAR        # виводить тип і значення: declare -a VAR='(...)'

# 5. Помилки арифметики
echo $((10/0))        # division by zero — EXIT при set -e!
result=$(echo "scale=2; 10/3" | bc)  # float division через bc

# 6. Проблеми з кодуванням
file script.sh        # перевірити encoding
cat -A script.sh      # бачити \r (^M) — Windows line endings
dos2unix script.sh    # виправити
hexdump -C script.sh | head  # побачити байти

# 7. Subshell variable scope
result=""
command | while read line; do
    result="$line"   # ВТРАЧАЄТЬСЯ! subshell
done
echo "$result"  # порожній!

# Правильно:
while read line; do
    result="$line"
done < <(command)
echo "$result"  # OK!

# 8. bashdb — повноцінний дебагер
# apt install bashdb
# bashdb script.sh
```

### ShellCheck — статичний аналіз

```bash
# Встановити
apt install shellcheck   # Ubuntu
brew install shellcheck  # macOS

# Перевірити скрипт
shellcheck script.sh

# Найчастіші попередження ShellCheck:
# SC2006: Use $(...) instead of backticks
# SC2046: Quote this to prevent word splitting
# SC2086: Double quote to prevent globbing and word splitting  ← найчастіше!
# SC2181: Check exit code directly, not $?
# SC2236: Use -n instead of ! -z
```

---

## Каверзні питання {#каверзні-питання}

### Питання 1: Що виведе?

```bash
x=5
(( x++ ))
echo $?
```

**Відповідь:** `1`!

Арифметичний вираз `(( x++ ))` повертає **exit code 1** якщо результат 0. Тут `x++` повертає старе значення — `5`, що є ненульовим, тому `$?` = 0. Але:

```bash
x=0
(( x++ ))  # повертає 0 (старе значення x), тому exit code = 1!
echo $?    # 1 ← ПАСТКА! Якщо set -e — скрипт упаде!
```

---

### Питання 2: Різниця між `[ ]` і `[[ ]]`

```bash
# Що станеться?
FILE=""
[ -f $FILE ] && echo "exists"    # Що виведе?
[[ -f $FILE ]] && echo "exists"  # А тут?
```

**Відповідь:** Перший варіант — помилка синтаксису при порожньому `$FILE`:
`[ -f  ]` → `[ -f ]` (без аргументу) → може поводитись несподівано.
Другий: безпечно, `[[ -f "" ]]` → false, нічого не виводить.

---

### Питання 3: Чому це не працює?

```bash
#!/bin/bash
set -e
count=0
let count++    # ← ПРОБЛЕМА!
echo $count
```

**Відповідь:** `let count++` повертає exit code `1` коли результат є `0` (false в арифметиці). Оскільки `count=0`, `let count++` повертає попереднє значення `0` → exit code `1` → `set -e` завершує скрипт!

Правильно:
```bash
(( count++ )) || true   # безпечно
(( ++count ))           # pre-increment завжди > 0 (якщо не overflow)
count=$(( count + 1 ))  # найбезпечніший варіант
```

---

### Питання 4: UUOC — Useless Use of Cat

```bash
# Погано (зайвий процес):
cat file.txt | grep "pattern"
cat file.txt | wc -l

# Правильно:
grep "pattern" file.txt
wc -l < file.txt
```

---

### Питання 5: Лапки і glob

```bash
# Що виведе якщо є файли foo1.txt, foo2.txt?
echo "foo*.txt"   # літерально: foo*.txt (в лапках немає glob!)
echo foo*.txt     # foo1.txt foo2.txt (glob спрацьовує)
ls "foo*.txt"     # ПОМИЛКА: no such file (glob в лапках не розкривається)
```

---

### Питання 6: `&&` vs `;` vs `\n`

```bash
cmd1 && cmd2   # cmd2 виконується тільки якщо cmd1 успішна
cmd1 || cmd2   # cmd2 виконується тільки якщо cmd1 провалилась
cmd1; cmd2     # cmd2 виконується завжди (незалежно від cmd1)
cmd1 &; cmd2   # ПОМИЛКА синтаксису
cmd1 & cmd2    # cmd1 у фоні, cmd2 одразу
```

---

### Питання 7: Різниця `$@` і `$*`

```bash
args_test() {
    echo "Using \$@:"
    for arg in "$@"; do echo "  [$arg]"; done

    echo "Using \$*:"
    for arg in "$*"; do echo "  [$arg]"; done
}

args_test "hello world" "foo" "bar baz"

# Вивід:
# Using $@:
#   [hello world]
#   [foo]
#   [bar baz]
# Using $*:
#   [hello world foo bar baz]   ← один рядок!
```

`"$@"` — розкривається у **окремі слова**, зберігаючи пробіли всередині аргументів.
`"$*"` — розкривається в **один рядок**, слова з'єднані `$IFS`.

---

### Питання 8: Що означає `2>&1 1>/dev/null`?

```bash
command 2>&1 1>/dev/null   # НЕПРАВИЛЬНО!
command 1>/dev/null 2>&1   # ПРАВИЛЬНО
```

Перенаправлення обробляються зліва направо. У першому варіанті: спочатку `stderr → stdout` (ще термінал), потім `stdout → /dev/null`. В результаті stderr залишається на терміналі! Правильно: спочатку направити stdout в /dev/null, потім stderr туди ж.

---

### Питання 9: Безпечне видалення файлів

```bash
# НЕБЕЗПЕЧНО:
dir="/var/log/myapp"
rm -rf $dir/    # Якщо $dir порожній → rm -rf / !!!

# БЕЗПЕЧНО:
rm -rf "${dir:?Variable is empty}/"
# Або:
[[ -n "$dir" ]] && rm -rf "$dir/"
# Або (bash 4.3+):
[[ -v dir ]] && rm -rf "$dir/"
```

---

### Питання 10: printf vs echo

```bash
echo -e "line1\nline2"      # -e не POSIX, не скрізь працює
echo "line1\nline2"         # На деяких системах виведе літерально \n
printf "line1\nline2\n"     # ЗАВЖДИ правильно (POSIX)

echo -n "no newline"        # -n не POSIX
printf "%s" "no newline"    # Портабельно

# printf підтримує форматування:
printf "%-10s %5.2f%%\n" "CPU:" 87.6
# CPU:       87.60%
```

---

## Реальні задачі зі співбесід {#задачі}

### Задача 1: Парсинг логів

**Завдання:** Знайти топ-10 IP адрес у Nginx лозі.

```bash
#!/bin/bash
# Рішення 1: класика (швидке)
awk '{print $1}' /var/log/nginx/access.log \
    | sort \
    | uniq -c \
    | sort -rn \
    | head -10

# Рішення 2: bash-only (повільніше, але показує знання)
declare -A ip_count
while IFS= read -r line; do
    ip="${line%% *}"  # перше слово
    (( ip_count[$ip]++ )) || true
done < /var/log/nginx/access.log

for ip in "${!ip_count[@]}"; do
    echo "${ip_count[$ip]} $ip"
done | sort -rn | head -10
```

---

### Задача 2: Rotate logs з архівацією

```bash
#!/bin/bash
set -euo pipefail

LOG_DIR="/var/log/myapp"
MAX_LOGS=7
DATE=$(date +%Y%m%d_%H%M%S)

# Rotate
if [[ -f "${LOG_DIR}/app.log" ]]; then
    mv "${LOG_DIR}/app.log" "${LOG_DIR}/app_${DATE}.log"
    gzip "${LOG_DIR}/app_${DATE}.log"
fi

# Прибрати старі (більше MAX_LOGS)
mapfile -t old_logs < <(
    ls -t "${LOG_DIR}"/app_*.log.gz 2>/dev/null
)

if (( ${#old_logs[@]} > MAX_LOGS )); then
    for old in "${old_logs[@]:$MAX_LOGS}"; do
        rm -f "$old"
        echo "Removed: $old"
    done
fi
```

---

### Задача 3: Retry з backoff

```bash
retry() {
    local max_attempts=$1
    local delay=$2
    local cmd=("${@:3}")
    local attempt=1

    while (( attempt <= max_attempts )); do
        if "${cmd[@]}"; then
            return 0
        fi
        echo "Attempt $attempt/$max_attempts failed. Retrying in ${delay}s..." >&2
        sleep "$delay"
        delay=$(( delay * 2 ))  # exponential backoff
        (( attempt++ ))
    done
    echo "All $max_attempts attempts failed." >&2
    return 1
}

# Використання:
retry 5 1 curl -sf https://api.example.com/health
```

---

### Задача 4: Mutex / lock file

```bash
#!/bin/bash
LOCKFILE="/var/run/myapp.lock"

acquire_lock() {
    # Атомарне створення lock файлу з PID
    if ( set -C; echo $$ > "$LOCKFILE" ) 2>/dev/null; then
        trap 'rm -f "$LOCKFILE"' EXIT
        return 0
    fi
    # Перевірити чи процес живий
    local old_pid
    old_pid=$(<"$LOCKFILE")
    if kill -0 "$old_pid" 2>/dev/null; then
        echo "Already running (PID: $old_pid)" >&2
        return 1
    fi
    # Stale lock — перезаписати
    echo "Removing stale lock (PID: $old_pid)" >&2
    rm -f "$LOCKFILE"
    acquire_lock
}

acquire_lock || exit 1
echo "Running with PID $$"
# ... основна логіка ...
```

---

### Задача 5: Паралельне виконання з обмеженням

```bash
#!/bin/bash
# Паралельно обробити файли, але не більше N одночасно
MAX_JOBS=4

process_file() {
    local file="$1"
    echo "Processing: $file"
    sleep $((RANDOM % 3 + 1))  # імітація роботи
}

active_jobs=0
for file in /data/*.csv; do
    process_file "$file" &
    (( active_jobs++ ))

    if (( active_jobs >= MAX_JOBS )); then
        wait -n 2>/dev/null || wait  # wait for any child (bash 4.3+)
        (( active_jobs-- ))
    fi
done
wait  # дочекатися всіх решти
echo "All done"
```

---

### Задача 6: Читання конфіга

```bash
#!/bin/bash
# Безпечно читати простий key=value конфіг
load_config() {
    local config_file="$1"
    [[ -f "$config_file" ]] || { echo "Config not found: $config_file" >&2; return 1; }

    while IFS='=' read -r key value; do
        # Пропустити коментарі та порожні рядки
        [[ "$key" =~ ^[[:space:]]*# ]] && continue
        [[ -z "$key" ]] && continue

        # Trim пробілів
        key="${key// /}"
        value="${value%%#*}"  # видалити inline коментарі
        value="${value%"${value##*[![:space:]]}"}"  # rtrim

        # Тільки дозволені символи в ключі (безпека!)
        [[ "$key" =~ ^[A-Za-z_][A-Za-z0-9_]*$ ]] || continue

        declare -g "$key=$value"
    done < "$config_file"
}

load_config /etc/myapp.conf
echo "Host: $DB_HOST, Port: $DB_PORT"
```

---

## Корисні one-liners для DevOps

```bash
# Перевірити чи порт відкритий (без netcat)
timeout 1 bash -c 'cat < /dev/null > /dev/tcp/host/port' && echo open

# Швидкий HTTP сервер (Python)
python3 -m http.server 8080

# Encode/decode base64
echo "hello" | base64
echo "aGVsbG8K" | base64 -d

# Генерація паролю
tr -dc 'A-Za-z0-9!@#$%' < /dev/urandom | head -c 20

# Знайти процес на порту
ss -tlnp | grep :80
fuser 80/tcp

# Рекурсивна заміна в файлах
find /etc -name "*.conf" -exec sed -i 's/old/new/g' {} +

# Видалити файли старіше N днів
find /var/log -name "*.log" -mtime +30 -delete

# Прогрес-бар для cp
rsync -ah --progress source dest

# Перевірити SSL сертифікат
echo | openssl s_client -connect host:443 2>/dev/null | openssl x509 -noout -dates

# Diff двох директорій
diff <(ls dir1/) <(ls dir2/)

# JSON pretty-print
cat file.json | python3 -m json.tool

# Watch з кольорами diff
watch -d -n 1 'ss -s'
```

---

## Чеклист для підготовки

- [ ] Знати різницю `[`, `[[`, `(())`
- [ ] Розуміти `$@` vs `$*` (з лапками і без)
- [ ] `set -euo pipefail` — коли потрібен і підводні камені
- [ ] `trap` — EXIT, ERR, сигнали
- [ ] Перенаправлення: порядок `2>&1` vs `>&2`
- [ ] Process substitution `< <()` та `> >()`
- [ ] Масиви: indexed та associative (declare -A)
- [ ] Parameter expansion: `${var:-default}`, `${var##*/}`, `${var//x/y}`
- [ ] `local` у функціях, `export`, `declare`
- [ ] Код виходу 126, 127, 128+N
- [ ] `shellcheck` — вміти пояснити типові попередження
- [ ] `bash -x` для дебагу
- [ ] `mktemp`, lock files, cleanup через trap

---

