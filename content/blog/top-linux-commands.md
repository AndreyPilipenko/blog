+++
title = "Top Linux Commands"
date = "2026-02-01T00:27:57+03:00"
draft = false
+++

Linux Commands Cheatsheet

Особистий довідник по адмініструванню Linux-серверів


📦 Диски та файлова система
bash# Огляд дисків та розділів
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

# inode vs файловий дескриптор
# inode — структура ФС, описує файл (тип, власник, права, лічильник посилань)
# FD (file descriptor) — ціле число, що повертає syscall open(); індекс у таблиці відкритих файлів ядра
# Прямого доступу до файлів у процесу немає — лише через ядро за FD

# Файлові дескриптори
sysctl fs.file-nr                      # [allocated, unused, max] дескрипторів у системі
lsof | awk '{print $1}' | sort | uniq -c | sort -r | head   # Топ-10 процесів за FD

# Розмір блоку
lockdev --getpbsz /dev/sdb

# Коли створена ФС
dumpe2fs $(mount | grep 'on \/ ' | awk '{print $1}') | grep 'Filesystem created:'

💿 S.M.A.R.T. та здоров'я дисків
bashsmartctl --info /dev/sda               # Базова інформація
smartctl -a /dev/sda                   # Повний звіт
smartctl -x /dev/sda                   # Розширений звіт
smartctl -t long /dev/sdb              # Запустити тривалий self-test
smartctl -A /dev/sda | grep -i temp    # Температура диску

# Диски в RAID (MegaRAID)
smartctl -a -d megaraid,5 /dev/sda

# Ключові атрибути
# Used_Rsvd_Blk_Cnt_Tot — використані блоки з резервного пулу (remapped blocks)
# Runtime_Bad_Block     — бед-блоки в основному пулі
# Uncorrectable_Error_Cnt — помилки, які не вдалось виправити
# ECC_Error_Rate        — помилки читання з перезаписом через ECC

# RAID
cat /proc/mdstat
vim /etc/mdadm.conf

# Перезаписати диск
dd if=/dev/urandom of=/dev/sda bs=1M status=progress

🌐 Мережа
bash# Інтерфейси
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

⚙️ Процеси та сервіси
bash# Огляд процесів
ps axf                                 # Дерево з аргументами
ps aux | grep defunct                  # Зомбі-процеси
cat /proc/<pid>/cmdline                # Командний рядок процесу
ll /proc/<pid>/exe                     # Де живе бінарник

# Пошук команди
which top

# Вбити процеси користувача
pkill -u username

# Завантаження системи
vmstat 1
# Дивитись: wa (I/O wait, має бути ~0), id (idle), si/so (swap in/out)

# Файлові дескриптори процесів
lsof | awk '{print $1}' | sort | uniq -c | sort -r | head

🖥️ Залізо та середовище
bash# Тип віртуалізації
systemd-detect-virt
dmesg | grep -i hypervisor
dmidecode -s system-manufacturer

# Параметри ядра при завантаженні
cat /proc/cmdline

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

📁 Архіви
bash# tar
tar -czvf archive.tar.gz /path/to/dir   # Запакувати
tar -xzvf archive.tar.gz                # Розпакувати
tar -tf archive.tar.gz                  # Переглянути вміст без розпакування

# xz
xz -z file.txt                          # Запакувати
unxz file.txt.xz                        # Розпакувати

🔗 Символічні посилання та пакети
bash# Символьне посилання
ln -s /full/path/to/file /usr/local/bin/alias_name

# Пакети
apt-cache show nginx                    # Версія в репозиторії
apt-mark hold nginx                     # Заморозити версію (не оновлювати)

# PHP
php -v
php -i

📊 Швидкі однострочники
bash# SSH в циклі (автоперепідключення)
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
# або
netstat -tlnp

🔐 Безпека та аудит
bash# Перевірити відкриті файли / сокети процесу
lsof -p <pid>

# Переглянути всі LISTEN-порти
ss -tlnp

# Хто і що робить на сервері
w

# Sudo-активність
grep sudo /var/log/auth.log | tail -30

# Перевірити crontab всіх користувачів
for u in $(cut -f1 -d: /etc/passwd); do crontab -l -u $u 2>/dev/null && echo "--- $u"; done

🛠️ Налагодження
bash# Трасування syscall процесу
strace -p <pid>
strace -e trace=network -p <pid>       # Тільки мережеві syscall

# Дампінг мережі
tcpdump -i eth0 -nn port 80
tcpdump -i any -nn -w /tmp/dump.pcap   # Зберегти в файл

# Перевірити journald
journalctl -u nginx --since "1 hour ago"
journalctl -f                          # Follow (як tail -f)
journalctl -xe                         # Останні помилки з контекстом

# Активні з'єднання
ss -s                                  # Статистика по протоколах
ss -tnp state established              # Всі встановлені TCP

tags = ["debug","linux"]

This is a page about »Top Linux Commands«.
