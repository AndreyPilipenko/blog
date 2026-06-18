+++
title = "Top Linux Commands"
date = "2025-02-01T00:27:57+03:00"
draft = false
+++

Linux Commands Cheatsheet

Особистий довідник по адмініструванню Linux-серверів

## 🧰 База: те, без чого нікуди

```bash
# Навігація та базові операції з файлами
pwd                                    # Поточна директорія
cd -                                   # Повернутись у попередню директорію
ls -lah                                # Список з прихованими, людським розміром
cp -v src dst                          # Копіювання з виводом дій
mv -v src dst                          # Переміщення/перейменування
rm -i file                             # Видалення з підтвердженням (захист від помилки)

# Перегляд вмісту
cat file
less file                              # Пагінація (q — вихід, / — пошук)
head -n 50 file
tail -n 50 -f /var/log/syslog          # -f — слідкувати за новими рядками

# Права доступу
chmod 750 file                         # rwx для власника, r-x для групи, нічого іншим
chmod u+x script.sh                    # Символьна форма: додати +x власнику
chown user:group file                  # Власник:група
chown -R user:group /path              # Рекурсивно
```

> **Чому 750, а не 777?**
> Цифра — це сума прав: read=4, write=2, execute=1. `7` (4+2+1) = rwx, `5` (4+1) = r-x, `0` = нічого. Завжди давай мінімум необхідних прав (принцип найменших привілеїв).

```bash
# Довідка та пошук команд
man command                            # Повна документація
command --help                         # Короткий список флагів
type command                           # Чи це alias, builtin чи бінарник
which command                          # Шлях до бінарника (тільки $PATH)
whereis command                        # Бінарник + man-сторінка + джерела

# Історія та alias
history | grep ssh                     # Пошук у історії команд
alias ll='ls -lah'                     # Тимчасовий alias (додай в ~/.bashrc для постійного)

# Змінні середовища
export VAR=value                       # Встановити для поточної сесії та дочірніх процесів
env | grep PATH                        # Список усіх env-змінних
printenv PATH
```

## 📦 Диски та файлова система

```bash
# Огляд дисків та розділів
lsblk -o NAME,MAJ:MIN,SIZE,TYPE,FSTYPE,MOUNTPOINT,UUID
fdisk -l
smartctl --scan                        # Визначити фізичні диски (якщо /dev/sda — залізний сервер)

# Місце на диску
df -h                                  # Файлові системи
df -hi                                 # По inode
du -h --max-depth=1 /var/virtualizor/ | sort -h
du -sh /home                           # Вага директорії
du -sch .[!.]* * | sort -h            # З прихованими директоріями
sudo du -h -d1 /root/ | sort -nr
ncdu -e                                # Інтерактивно + приховані директорії

# Метадані конкретного файлу
stat file                              # inode, права, час доступу/зміни/створення
```

> **inode vs файловий дескриптор**
> - **inode** — структура ФС, описує файл (тип, власник, права, лічильник посилань)
> - **FD (file descriptor)** — ціле число, що повертає syscall `open()`; індекс у таблиці відкритих файлів ядра

```bash
# Файлові дескриптори
sysctl fs.file-nr                      # [allocated, unused, max] дескрипторів у системі
lsof | awk '{print $1}' | sort | uniq -c | sort -r | head   # Топ-10 процесів за FD

# Розмір блоку
lockdev --getpbsz /dev/sdb

# Коли створена ФС
dumpe2fs $(mount | grep 'on \/ ' | awk '{print $1}') | grep 'Filesystem created:'
```

## 🔍 Пошук файлів (find / locate / xargs)

`find` — найпотужніший інструмент для пошуку за метаданими, а не вмістом (для вмісту — `grep -r`).

| Флаг | Що робить | Приклад |
|---|---|---|
| `-name` / `-iname` | Пошук за іменем (iname — без урахування регістру) | `find /var/log -iname "*.log"` |
| `-type f/d/l` | Тип: файл / директорія / симлінк | `find . -type d` |
| `-size +100M` / `-1k` | Більше/менше за розмір | `find / -size +1G` |
| `-mtime +30` / `-mmin -60` | Змінено більше 30 днів тому / менше 60 хв тому | `find /tmp -mtime +30` |
| `-ctime` / `-atime` | Час зміни inode / час доступу | `find . -atime +90` |
| `-perm -4000` | SUID-біт встановлено | `find / -perm -4000` |
| `-perm -2000` | SGID-біт | `find / -perm -2000` |
| `-perm -o+w` | Файли, доступні на запис будь-кому | `find / -perm -o+w -type f` |
| `-user` / `-group` | Власник / група | `find /home -user olduser` |
| `-newer ref` | Змінено пізніше за файл-референс | `find . -newer /tmp/marker` |
| `-maxdepth N` / `-mindepth N` | Обмеження глибини рекурсії | `find . -maxdepth 1` |
| `-xtype l` | Зламані симлінки (ціль не існує) | `find . -xtype l` |
| `-prune` | Виключити директорію з обходу | див. приклад нижче |

```bash
# Знайти і видалити логи старші 30 днів
find /var/log/myapp -name "*.log" -mtime +30 -print     # спочатку перевір список
find /var/log/myapp -name "*.log" -mtime +30 -delete    # тільки після перевірки

# Виключити директорію з пошуку (наприклад node_modules)
find . -path ./node_modules -prune -o -name "*.js" -print

# Файли більші за 1GB у всій системі
find / -xdev -size +1G -exec ls -lh {} \;

# Безпечна передача результатів у іншу команду (працює з пробілами в іменах)
find . -name "*.tmp" -print0 | xargs -0 rm -v

# locate — швидше, але працює з кешем (updatedb), може бути неактуальним
locate nginx.conf
sudo updatedb                          # Оновити кеш locate вручну
```

> ⚠️ **WARNING:** `-delete` видаляє файли безповоротно і без підтвердження. Завжди спочатку запускай `find` без `-delete`/`-exec rm`, перевір вивід, і лише тоді додавай деструктивний флаг.

## 💿 S.M.A.R.T. та здоров'я дисків

```bash
smartctl --info /dev/sda               # Базова інформація
smartctl -a /dev/sda                   # Повний звіт
smartctl -x /dev/sda                   # Розширений звіт
smartctl -t long /dev/sdb              # Запустити тривалий self-test
smartctl -A /dev/sda | grep -i temp    # Температура диску

# Диски в RAID (MegaRAID)
smartctl -a -d megaraid,5 /dev/sda
```

> **Ключові атрибути**
> - `Used_Rsvd_Blk_Cnt_Tot` — використані блоки з резервного пулу (remapped blocks)
> - `Runtime_Bad_Block` — бед-блоки в основному пулі
> - `Uncorrectable_Error_Cnt` — помилки, які не вдалось виправити
> - `ECC_Error_Rate` — помилки читання з перезаписом через ECC

```bash
# RAID
cat /proc/mdstat
vim /etc/mdadm.conf
```

> ⚠️ **WARNING:** наступна команда безповоротно перезаписує диск випадковими даними — використовується тільки для безпечного знищення даних перед утилізацією/продажем носія, ніколи на робочому диску.

```bash
dd if=/dev/urandom of=/dev/sda bs=1M status=progress
```

## 🌐 Мережа

```bash
# Інтерфейси
ip link ls up
ip -4 a
ip addr show dev eth0

# Швидкість інтерфейсу
sudo ethtool eth0 | grep Speed:
cat /sys/class/net/eth0/speed
dmesg | grep eth                       # Історія інтерфейсів

# Перевірка з'єднання
nc -vz google.com 80                   # telnet-аналог
mtr -rc 50 8.8.8.8                     # Трасування з 50 пакетами

# BGP lookup (ASN, prefix, origin)
echo -e "130.254.117.4" | nc bgp.tools 43

# Трафік
iftop                                  # Real-time мережева активність
vnstat -m --config <(cat /etc/vnstat.conf; echo "UnitMode 2")   # Статистика по місяцях

# conntrack (NAT / state таблиця)
cat /proc/sys/net/netfilter/nf_conntrack_max
cat /proc/sys/net/netfilter/nf_conntrack_count
conntrack -C                           # Поточна кількість сесій
```

### 🔧 ethtool — діагностика та тюнінг NIC

```bash
ethtool eth0                           # Speed, Duplex, Auto-negotiation, Link detected
ethtool -i eth0                        # Драйвер NIC + версія firmware — важливо при багах драйвера

# Лічильники помилок на самій карті (нижче рівня ОС)
ethtool -S eth0 | grep -Ei 'drop|error|miss'
```

> **Навіщо `ethtool -S`, якщо вже є `ip -s link`?**
> `ip -s link` показує статистику з боку ядра. `ethtool -S` показує лічильники з самого драйвера/чипа NIC — там видно втрати ще до того, як пакет дійшов до стека ядра (наприклад `rx_missed_errors` через переповнення ring buffer при високому PPS на проксі-серверах).

```bash
# Ring buffer (буфер на карті) — типова причина drops при високому PPS
ethtool -g eth0                        # Поточні/максимальні значення rx/tx
ethtool -G eth0 rx 4096 tx 4096        # Збільшити (макс. залежить від драйвера)

# Offload-фічі (TSO/GSO/GRO/LRO/checksumming)
ethtool -k eth0                        # Список усіх offload-фіч та їх стану
ethtool -K eth0 gro off                # Вимкнути GRO (часто потрібно для коректного tcpdump
                                        # або коли карта виступає роутером/бриджем)
```

> ⚠️ **WARNING:** `ethtool -s` (force speed/duplex/autoneg) може миттєво обірвати лінк, якщо налаштування не співпадають зі свічем. Перевіряй з консолі/IPMI, а не по SSH через той самий інтерфейс.

```bash
ethtool -s eth0 speed 10000 duplex full autoneg off
```

### 📏 MTU та Jumbo Frames

```bash
ip link set eth0 mtu 9000              # Jumbo frames (мають підтримуватись усім шляхом: NIC, свіч, апстрім)

# Path MTU discovery: -s 1472 + 28 байт (20 IP + 8 ICMP) = 1500 без фрагментації
ping -M do -s 1472 8.8.8.8             # -M do — заборонити фрагментацію, виявити реальний MTU шляху
```

### 🔗 Bonding / LACP

```bash
cat /proc/net/bonding/bond0            # Режим, активні слейви, MII status
ip -d link show bond0
```

> **Hash policy `layer3+4`**
> Визначає, як трафік розподіляється між слейвами в режимах `xor`/`802.3ad`. `layer2` дивиться лише на MAC (погана балансировка при одному gateway), `layer3+4` враховує IP+порт — потрібно для реальної агрегації пропускної здатності >1Gbps на одне джерело/призначення.

### 🧭 Маршрутизація

```bash
ip route show table all
ip rule list                           # Policy routing rules (fwmark-based)
ip route get 8.8.8.8                   # Який маршрут реально обере ядро для цієї цілі
```

### 🌍 DNS

```bash
dig +short example.com
dig -x 8.8.8.8                         # Reverse DNS (PTR)
resolvectl status                      # Якщо використовується systemd-resolved
```

### ⏱️ HTTP/TLS діагностика

```bash
# Розбивка часу запиту по фазах
curl -o /dev/null -s -w 'dns:%{time_namelookup} connect:%{time_connect} ttfb:%{time_starttransfer} total:%{time_total}\n' https://example.com

# Дата експірації TLS-сертифіката без браузера
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | openssl x509 -noout -dates
```

### 🚦 Traffic shaping

```bash
tc qdisc show dev eth0                 # Поточна дисципліна черги
```

> ⚠️ **WARNING:** додавання `tc qdisc` змінює поведінку трафіку миттєво для всього інтерфейсу — тестуй на некритичному interface/VLAN.

```bash
tc qdisc add dev eth0 root tbf rate 100mbit burst 32kbit latency 400ms
```

### ⚙️ sysctl тюнінг мережі

```bash
sysctl net.core.somaxconn                       # Глибина backlog для accept() — мала=connection refused під навантаженням
sysctl net.ipv4.tcp_congestion_control           # bbr/cubic/reno
sysctl net.ipv4.ip_local_port_range              # Діапазон ефемерних портів для вихідних з'єднань
sysctl net.netfilter.nf_conntrack_max            # Ліміт NAT-таблиці — занадто малий = "table full, dropping packet"

# Персистентно (не забувай комент "що/чому" в самому файлі — для майбутнього себе)
echo "net.ipv4.tcp_congestion_control = bbr" >> /etc/sysctl.d/99-proxy.conf
sysctl --system                                  # Застосувати всі sysctl.d/* без перезавантаження
```

## ⚙️ Процеси та сервіси

```bash
# Огляд процесів
ps axf                                 # Дерево з аргументами
ps aux | grep defunct                  # Зомбі-процеси
cat /proc/<pid>/cmdline                # Командний рядок процесу
cat /proc/<pid>/status                 # VmRSS, State, Threads, тощо
ll /proc/<pid>/exe                     # Де живе бінарник
ls -l /proc/<pid>/fd                   # Усі відкриті файлові дескриптори процесу
cat /proc/<pid>/limits                 # Поточні ulimit для процесу (часто причина "too many open files")

# Пошук команди
which top

# Вбити процеси користувача
pkill -u username
```

### 🎚️ Пріоритети процесів: nice / renice / ionice

> **nice vs renice**
> - `nice` — задає пріоритет CPU **при старті** процесу. Шкала від `-20` (найвищий пріоритет) до `19` (найнижчий), за замовчуванням `0`.
> - `renice` — змінює пріоритет **уже запущеного** процесу за PID.
> - `ionice` — те саме, але для пріоритету **дискового I/O** (окрема підсистема, не пов'язана з CPU nice).

```bash
nice -n 10 ./heavy_script.sh           # Запустити з пониженим пріоритетом (не заважати іншим процесам)
renice -n -5 -p 1234                   # Підняти пріоритет вже запущеного PID 1234 (потрібен root для від'ємних значень)

ionice -c2 -n7 -p 1234                 # Клас 2 (best-effort), найнижчий пріоритет у класі — для фонових backup-задач
```

```bash
# ulimit — ліміти ресурсів для поточної shell-сесії
ulimit -a                              # Всі поточні ліміти
ulimit -n 65535                        # Підняти ліміт відкритих файлів (тимчасово, для сесії)
# Персистентно — через /etc/security/limits.conf або systemd unit (LimitNOFILE=)
```

```bash
# Завантаження системи
vmstat 1
# Дивитись: wa (I/O wait, має бути ~0), id (idle), si/so (swap in/out)

# Файлові дескриптори процесів
lsof | awk '{print $1}' | sort | uniq -c | sort -r | head

#pstree
# Не згортати ідентичні гілки (показати кожен процес окремо)
pstree -c
# Показати PID поруч з іменем процесу
pstree -p
# Виділити поточний процес та всіх його предків
pstree -h
# Показати процеси конкретного користувача
pstree username
```

### 🔁 systemd

```bash
systemctl list-units --failed          # Швидко знайти, що впало
systemctl show nginx -p MainPID        # PID без парсингу status
systemctl cat nginx                    # Показати unit-файл як він є після override'ів
systemd-analyze blame                  # Що найдовше вантажиться при старті системи
systemd-cgtop                          # Real-time CPU/RAM по cgroups (control groups systemd)
```

## 🖥️ Залізо та середовище

```bash
# Тип віртуалізації
systemd-detect-virt
dmesg | grep -i hypervisor
dmidecode -s system-manufacturer

# Параметри ядра при завантаженні
cat /proc/cmdline

# CPU
lscpu                                  # Архітектура, кількість ядер/потоків, кеші
nproc                                  # Скільки ядер бачить ОС (враховує cgroup-ліміти контейнера)
numactl --hardware                     # NUMA-топологія: скільки нод, скільки RAM на кожній

# RAM
dmidecode -t 17                        # Деталі по планках пам'яті (тип, частота, слот)
dmidecode --type 17                    # Аналог
memtester 100M 2                       # Тест пам'яті

# Температура
sensors                                # Ядра CPU
smartctl -A /dev/sda | grep -i temp   # Диск

# Відеокарта
lshw -c video | grep driver
glxinfo -B
```

## 📁 Архіви та копіювання

```bash
# tar
tar -czvf archive.tar.gz /path/to/dir   # Запакувати
tar -xzvf archive.tar.gz                # Розпакувати
tar -tf archive.tar.gz                  # Переглянути вміст без розпакування

# xz
xz -z file.txt                          # Запакувати
unxz file.txt.xz                        # Розпакувати

# zip
zip -r archive.zip /path/to/dir
unzip -l archive.zip                    # Переглянути вміст без розпакування

# rsync — копіювання з дельтою (синхронізація, а не повне перезаписування)
rsync -avzP --delete src/ user@host:/dst/   # -a архів, -z стиснення, -P progress+resume, --delete видаляє зайве на приймачі
rsync -avz --dry-run src/ dst/              # Симуляція без реальних змін — завжди спочатку так
```

> ⚠️ **WARNING:** `rsync --delete` видаляє на приймачі файли, яких немає у джерелі. Завжди перевіряй спочатку через `--dry-run`.

```bash
# scp — простий разовий перенос
scp -P 2222 file.txt user@host:/path/
```

## 🔗 Символічні посилання та пакети

```bash
# Символьне посилання
ln -s /full/path/to/file /usr/local/bin/alias_name

# APT (Debian/Ubuntu)
apt-cache show nginx                    # Версія в репозиторії
apt-mark hold nginx                     # Заморозити версію (не оновлювати)
apt list --upgradable                   # Що можна оновити
dpkg -L nginx                           # Які файли встановив пакет
dpkg -S /usr/sbin/nginx                 # Якому пакету належить файл

# YUM/DNF (RHEL/CentOS)
rpm -qa | grep nginx                    # Чи встановлений пакет
rpm -ql nginx                           # Файли пакету
yum history                             # Історія транзакцій yum

# Динамічні бібліотеки
ldd /usr/sbin/nginx                     # Які .so-залежності потрібні бінарнику
ldconfig -p | grep ssl                  # Кеш динамічного лінкера

# PHP
php -v
php -i
```

## 📊 Швидкі однострочники

```bash
# SSH в циклі (автоперепідключення)
while true; do date; ssh user@host; done

# Скільки з'єднань до конкретного порту
ss -tnp | grep ':80' | wc -l

# Топ процесів за CPU (один знімок)
ps aux --sort=-%cpu | head -10

# Топ процесів за RAM
ps aux --sort=-%mem | head -10

# Хто залогінений
who -a

# Останні логіни
last -20

# Перевірити відкриті порти локально
ss -tlnp
netstat -tlnp
```

### 🧵 grep / awk / sed / sort / xargs — текстова "база"

```bash
grep -i "error" file.log               # Без урахування регістру
grep -v "debug" file.log               # Інвертувати — показати все, ОКРІМ debug
grep -rn "TODO" ./src                  # Рекурсивно по директорії, з номерами рядків

awk '{print $1, $3}' file.log          # Витягти 1-й і 3-й стовпці (роздільник — пробіл за замовчуванням)
awk -F',' '{print $2}' file.csv        # Свій роздільник

sed -n '10,20p' file                   # Показати тільки рядки 10-20
sed -i 's/old/new/g' file              # Заміна in-place по всьому файлу

sort -k2 -n file | uniq -c             # Сортувати за 2-м стовпцем чисельно, рахувати унікальні

xargs -n1 echo < list.txt              # Виконати команду для кожного рядка вхідних даних

watch -n2 'ss -tnp | wc -l'            # Повторювати команду кожні 2с (моніторинг у реальному часі)
```

## 🔐 Безпека та аудит

```bash
# Перевірити відкриті файли / сокети процесу
lsof -p <pid>

# Переглянути всі LISTEN-порти
ss -tlnp

# Хто і що робить на сервері
w

# Sudo-активність
grep sudo /var/log/auth.log | tail -30

# Невдалі спроби логіну
lastb -20

# Перевірити crontab всіх користувачів
for u in $(cut -f1 -d: /etc/passwd); do crontab -l -u $u 2>/dev/null && echo "--- $u"; done

# Файли із SUID/SGID — потенційний privilege escalation вектор
find / -xdev -perm -4000 -o -perm -2000 2>/dev/null

# Firewall: поточні правила
iptables -L -n -v
ufw status verbose

# fail2ban: хто заблокований
fail2ban-client status sshd
```

## 🕐 Cron та systemd timers

```bash
crontab -l                             # Поточні задачі користувача
crontab -e                             # Редагувати
# Формат: мин(0-59) год(0-23) день_міс(1-31) місяць(1-12) день_тиж(0-6, 0=нд)
# */15 * * * *  → кожні 15 хвилин

systemctl list-timers --all            # Системний аналог cron у systemd
journalctl -u myservice.timer          # Логи конкретного таймера
```

## 🛠️ Налагодження

```bash
# Трасування syscall процесу
strace -p <pid>
strace -e trace=network -p <pid>       # Тільки мережеві syscall
strace -c -p <pid>                     # Підсумок: скільки разів і скільки часу на кожен syscall

# Дампінг мережі
tcpdump -i eth0 -nn port 80
tcpdump -i any -nn -w /tmp/dump.pcap   # Зберегти в файл

# Перевірити journald
journalctl -u nginx --since "1 hour ago"
journalctl -f                          # Follow (як tail -f)
journalctl -xe                         # Останні помилки з контекстом
journalctl --disk-usage                # Скільки місця займають логи
journalctl --vacuum-time=7d            # Почистити логи старші 7 днів

# dmesg з людськими таймстампами
dmesg -T | tail -50

# Активні з'єднання
ss -s                                  # Статистика по протоколах
ss -tnp state established              # Всі встановлені TCP
```
