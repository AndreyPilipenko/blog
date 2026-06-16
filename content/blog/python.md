+++
title = "Python з поясненнями та прикладами"
date = "2023-06-16"
draft = false
tags = ["python", "programming"]
+++
Ця стаття — не просто список команд. Кожна концепція пояснена з відповіддю на питання **"чому саме так?"** — щоб ти розумів логіку, а не просто запам'ятав синтаксис.

---

## 🧠 1. Основи Python

### Змінні та типи даних

Python — **динамічно типізована** мова. Це означає: тип змінної визначається автоматично в момент присвоєння, не треба писати `int x = 5` як у Java.

```python
# Числа
age = 25          # int — ціле число
price = 19.99     # float — число з крапкою
big = 10 ** 18    # Python не має обмеження на розмір int (на відміну від C)

# Рядки
name = "Andrii"
greeting = f"Hello, {name}!"   # f-string — найсучасніший спосіб форматування

# Булевий тип
is_active = True
is_empty = False

# None — відсутність значення (аналог null в інших мовах)
result = None
```

> **Чому f-string, а не `%s` або `.format()`?**  
> f-string — читається найлегше, виконується найшвидше, підтримує будь-які вирази прямо в `{}`. Завжди використовуй f-string у Python 3.6+.

**Перевірка типу:**

```python
print(type(age))      # <class 'int'>
print(isinstance(age, int))   # True — правильний спосіб перевірки типу
```

> `isinstance()` краще ніж `type() ==`, бо враховує наслідування класів.

---

### Умови `if / elif / else`

```python
score = 75

if score >= 90:
    grade = "A"
elif score >= 75:
    grade = "B"
elif score >= 60:
    grade = "C"
else:
    grade = "F"

print(f"Оцінка: {grade}")   # Оцінка: B
```

**Тернарний оператор** (одна умова в одному рядку):

```python
status = "active" if score >= 60 else "failed"
```

> Використовуй тернарний оператор лише для простих умов — складна логіка в одному рядку стає нечитабельною.

---

### Цикли `for` і `while`

**`for` — коли знаєш кількість ітерацій або ітеруєш по колекції:**

```python
fruits = ["apple", "banana", "cherry"]

for fruit in fruits:
    print(fruit)

# З індексом — використовуй enumerate(), НЕ range(len(...))
for i, fruit in enumerate(fruits):
    print(f"{i}: {fruit}")

# range() — числовий діапазон
for i in range(5):       # 0, 1, 2, 3, 4
    print(i)

for i in range(2, 10, 2):  # 2, 4, 6, 8 (start, stop, step)
    print(i)
```

**`while` — коли умова зупинки невідома заздалегідь:**

```python
attempts = 0
max_attempts = 3

while attempts < max_attempts:
    user_input = input("Введи пароль: ")
    if user_input == "secret":
        print("Доступ надано")
        break
    attempts += 1
else:
    # else у while виконується якщо цикл завершився БЕЗ break
    print("Доступ заблоковано")
```

> **`break`** — зупиняє цикл негайно.  
> **`continue`** — пропускає поточну ітерацію, переходить до наступної.  
> **`else` після циклу** — маловідома фіча: виконується якщо цикл завершився природньо (без `break`).

---

### Функції

```python
# Базова функція
def greet(name):
    return f"Hello, {name}!"

# Аргументи зі значенням за замовчуванням
def connect(host, port=5432, timeout=30):
    return f"Connecting to {host}:{port} (timeout={timeout}s)"

# Виклик — можна передавати іменовані аргументи в будь-якому порядку
connect("localhost")                    # використає дефолтні port і timeout
connect("db.prod", port=3306)          # перевизначаємо тільки port
connect(host="db.prod", timeout=60)   # іменовані аргументи
```

**`*args` і `**kwargs`** — коли кількість аргументів невідома:

```python
def sum_all(*args):
    # args — це tuple з усіх переданих позиційних аргументів
    return sum(args)

print(sum_all(1, 2, 3, 4))   # 10

def log_event(event, **kwargs):
    # kwargs — це dict з іменованих аргументів
    print(f"Event: {event}")
    for key, value in kwargs.items():
        print(f"  {key}: {value}")

log_event("login", user="andrii", ip="192.168.1.1", status=200)
```

**Lambda** — анонімна функція для простих операцій:

```python
square = lambda x: x ** 2
print(square(5))   # 25

# Найчастіше використовується як аргумент sorted/filter/map
numbers = [3, 1, 4, 1, 5, 9, 2]
sorted_nums = sorted(numbers, key=lambda x: -x)  # сортування від більшого
print(sorted_nums)  # [9, 5, 4, 3, 2, 1, 1]
```

---

### Робота з файлами

```python
# Запис у файл
with open("data.txt", "w", encoding="utf-8") as f:
    f.write("Рядок перший\n")
    f.write("Рядок другий\n")

# Читання всього файлу
with open("data.txt", "r", encoding="utf-8") as f:
    content = f.read()
    print(content)

# Читання рядок за рядком (ефективно для великих файлів)
with open("data.txt", "r", encoding="utf-8") as f:
    for line in f:
        print(line.strip())   # strip() прибирає \n в кінці

# Додавання до файлу (не перезаписуючи)
with open("data.txt", "a", encoding="utf-8") as f:
    f.write("Рядок третій\n")
```

> **Чому `with open(...)`, а не `f = open(...)`?**  
> `with` автоматично закриває файл навіть якщо виникне виняток. Це **контекстний менеджер** — він гарантує cleanup. Завжди використовуй `with`.

---

### Обробка помилок `try / except`

```python
# Базовий try/except
try:
    result = 10 / 0
except ZeroDivisionError:
    print("Ділення на нуль!")

# Кілька типів винятків
try:
    value = int("not a number")
except ValueError as e:
    print(f"Невірне значення: {e}")
except TypeError as e:
    print(f"Невірний тип: {e}")

# finally — виконується ЗАВЖДИ (і при помилці, і без)
# Типове використання: закриття з'єднань, файлів
try:
    conn = open_db_connection()
    conn.execute("SELECT 1")
except Exception as e:
    print(f"Помилка: {e}")
finally:
    conn.close()   # завжди закриємо з'єднання

# else — виконується тільки якщо помилки НЕ було
try:
    data = fetch_data()
except RequestError as e:
    print(f"Не вдалося отримати дані: {e}")
else:
    process(data)   # виконується тільки при успіху
```

**Власні виключення:**

```python
class ValidationError(Exception):
    """Виняток для помилок валідації даних."""
    pass

def validate_age(age):
    if age < 0 or age > 150:
        raise ValidationError(f"Невірний вік: {age}")
    return age

try:
    validate_age(-5)
except ValidationError as e:
    print(e)   # Невірний вік: -5
```

---

### Імпорти

```python
# Повний імпорт модуля
import os
import sys

# Імпорт конкретних функцій/класів
from pathlib import Path
from datetime import datetime, timedelta

# Аліас для довгих назв
import json as j

# НЕ рекомендується (забруднює namespace):
from os import *
```

---

## 🧱 2. Структури даних

### List (список)

**Список** — впорядкована, змінювана колекція. Дозволяє дублікати.

```python
servers = ["web-01", "web-02", "db-01"]

# Основні операції
servers.append("cache-01")       # додати в кінець: O(1)
servers.insert(1, "web-00")      # вставити за індексом: O(n)
servers.remove("db-01")          # видалити перший збіг: O(n)
popped = servers.pop()           # видалити та повернути останній: O(1)
popped = servers.pop(0)          # видалити та повернути за індексом: O(n)

# Зрізи (slicing) — повертають новий список
first_two = servers[:2]          # перші два елементи
last_two = servers[-2:]          # останні два
reversed_list = servers[::-1]    # реверс

# Корисні методи
print(len(servers))              # довжина
print("web-01" in servers)       # перевірка наявності: O(n)
servers.sort()                   # сортування на місці
sorted_copy = sorted(servers)    # новий відсортований список
servers.reverse()                # реверс на місці
```

**Коли використовувати list:**
- потрібен порядок елементів
- потрібен доступ за індексом
- елементи можуть повторюватися

---

### Tuple (кортеж)

**Tuple** — впорядкована, **незмінна** колекція.

```python
# Незмінний — після створення не можна змінити
coordinates = (50.4501, 30.5234)  # широта/довгота Києва
rgb = (255, 128, 0)

# Розпакування (unpacking)
lat, lon = coordinates
print(f"Lat: {lat}, Lon: {lon}")

# Функція може повернути кілька значень через tuple
def get_min_max(numbers):
    return min(numbers), max(numbers)

low, high = get_min_max([3, 1, 4, 1, 5, 9])
print(f"Min: {low}, Max: {high}")   # Min: 1, Max: 9

# Tuple як ключ dict (list не можна — він не hashable)
location_data = {
    (50.45, 30.52): "Kyiv",
    (48.46, 35.05): "Dnipro",
}
```

> **Чому tuple, а не list?**  
> Tuple незмінний → Python може оптимізувати пам'ять і швидкість. Семантично tuple означає "ця структура не буде змінюватись" — координати, RGB, повернені значення функції.

---

### Dict (словник)

**Dict** — колекція пар ключ-значення. Ключі унікальні. З Python 3.7+ порядок вставки зберігається.

```python
server = {
    "hostname": "web-01",
    "ip": "192.168.1.10",
    "port": 80,
    "active": True,
}

# Доступ до значень
print(server["hostname"])              # "web-01" — KeyError якщо ключ відсутній
print(server.get("hostname"))          # "web-01" — None якщо ключ відсутній
print(server.get("cpu", "unknown"))    # "unknown" — дефолтне значення

# Модифікація
server["port"] = 443                   # оновити значення
server["ssl"] = True                   # додати новий ключ
del server["active"]                   # видалити ключ

# Ітерація
for key in server:                     # ітерація по ключах
    print(key)

for value in server.values():          # ітерація по значеннях
    print(value)

for key, value in server.items():      # ітерація по парах
    print(f"{key}: {value}")

# Злиття словників (Python 3.9+)
defaults = {"timeout": 30, "retries": 3}
config = {"host": "db.prod"} | defaults   # новий об'єднаний словник
```

**Вкладені словники:**

```python
cluster = {
    "web": {
        "servers": ["web-01", "web-02"],
        "port": 80,
    },
    "db": {
        "servers": ["db-master", "db-replica"],
        "port": 5432,
    },
}

print(cluster["web"]["servers"][0])   # "web-01"
```

**`defaultdict`** — автоматично створює дефолтне значення:

```python
from collections import defaultdict

# Групування за першою буквою
words = ["apple", "ant", "banana", "bear", "cherry"]
grouped = defaultdict(list)

for word in words:
    grouped[word[0]].append(word)

print(dict(grouped))
# {'a': ['apple', 'ant'], 'b': ['banana', 'bear'], 'c': ['cherry']}
```

---

### Set (множина)

**Set** — невпорядкована колекція **унікальних** елементів. Перевірка наявності `O(1)` (на відміну від list `O(n)`).

```python
active_ips = {"192.168.1.1", "192.168.1.2", "192.168.1.3"}
blocked_ips = {"192.168.1.2", "10.0.0.5"}

# Операції теорії множин
union = active_ips | blocked_ips          # об'єднання
intersection = active_ips & blocked_ips   # перетин: {'192.168.1.2'}
difference = active_ips - blocked_ips     # в active але не в blocked
sym_diff = active_ips ^ blocked_ips       # є в одному але не в обох

# Перевірка
print("192.168.1.1" in active_ips)   # True — O(1)!

# Видалення дублікатів зі списку
numbers = [1, 2, 2, 3, 3, 3, 4]
unique = list(set(numbers))           # [1, 2, 3, 4] — порядок не гарантований
```

> **Коли set замість list?**  
> Коли треба перевіряти наявність (`in`) — set у 100x швидший на великих даних. Коли треба унікальність. Коли треба операції з множинами.

---

### Comprehensions (генераторні вирази)

Компактний спосіб створити нову колекцію з існуючої. Читабельніший і часто швидший за `for`-цикл.

**List comprehension:**

```python
# Традиційний спосіб
squares = []
for x in range(10):
    squares.append(x ** 2)

# List comprehension — еквівалентно, але коротше
squares = [x ** 2 for x in range(10)]

# З умовою (фільтрація)
even_squares = [x ** 2 for x in range(10) if x % 2 == 0]
# [0, 4, 16, 36, 64]

# З перетворенням рядків
servers = ["WEB-01", "WEB-02", "DB-01"]
lower_servers = [s.lower() for s in servers]
# ["web-01", "web-02", "db-01"]
```

**Dict comprehension:**

```python
# Перевернути ключі та значення
original = {"a": 1, "b": 2, "c": 3}
inverted = {v: k for k, v in original.items()}
# {1: "a", 2: "b", 3: "c"}

# Фільтрація по значенню
servers = {"web-01": True, "web-02": False, "db-01": True}
active = {name: status for name, status in servers.items() if status}
# {"web-01": True, "db-01": True}
```

**Set comprehension:**

```python
# Унікальні домени з email-адрес
emails = ["user@gmail.com", "admin@ukr.net", "dev@gmail.com"]
domains = {email.split("@")[1] for email in emails}
# {"gmail.com", "ukr.net"}
```

**Generator expression** — не створює список одразу, генерує по одному:

```python
# Якщо треба просто обчислити суму — не треба створювати список
total = sum(x ** 2 for x in range(1_000_000))   # ефективно по пам'яті
```

---

### Коли що використовувати — шпаргалка

| Структура | Порядок | Змінна | Унікальність | Ключ-значення | Пошук |
|-----------|---------|--------|--------------|---------------|-------|
| `list`    | ✅      | ✅     | ❌           | ❌            | O(n)  |
| `tuple`   | ✅      | ❌     | ❌           | ❌            | O(n)  |
| `dict`    | ✅ 3.7+ | ✅     | ключі ✅     | ✅            | O(1)  |
| `set`     | ❌      | ✅     | ✅           | ❌            | O(1)  |

---

## 🧩 3. ООП — Об'єктно-орієнтоване програмування

### Класи та об'єкти

**Клас** — шаблон (blueprint). **Об'єкт** — конкретний екземпляр цього шаблону.

```python
class Server:
    # Атрибут класу — спільний для ВСІХ екземплярів
    default_port = 80

    def __init__(self, hostname, ip, port=None):
        # self — посилання на конкретний екземпляр
        # Атрибути екземпляра — унікальні для кожного об'єкта
        self.hostname = hostname
        self.ip = ip
        self.port = port or self.default_port
        self.is_active = False   # дефолтний стан

    def start(self):
        self.is_active = True
        print(f"{self.hostname} запущено на {self.ip}:{self.port}")

    def stop(self):
        self.is_active = False
        print(f"{self.hostname} зупинено")

    def status(self):
        state = "активний" if self.is_active else "зупинений"
        return f"{self.hostname} [{self.ip}:{self.port}] — {state}"

    # __repr__ — repr для розробника (в дебаггері, в shell)
    def __repr__(self):
        return f"Server(hostname={self.hostname!r}, ip={self.ip!r})"

    # __str__ — для print() та str()
    def __str__(self):
        return self.status()


# Створення об'єктів
web1 = Server("web-01", "192.168.1.10")
web2 = Server("web-02", "192.168.1.11", port=8080)

web1.start()
print(web1)          # web-01 [192.168.1.10:80] — активний
print(repr(web1))    # Server(hostname='web-01', ip='192.168.1.10')
```

---

### Наслідування

```python
class DatabaseServer(Server):
    """Спеціалізований сервер БД — розширює базовий Server."""

    def __init__(self, hostname, ip, db_name, port=5432):
        # Викликаємо __init__ батьківського класу
        super().__init__(hostname, ip, port)
        self.db_name = db_name
        self.connections = []

    def connect(self, user):
        self.connections.append(user)
        print(f"Користувач {user} підключився до {self.db_name}")

    def status(self):
        # Перевизначаємо метод батька (override)
        base_status = super().status()
        return f"{base_status} | DB: {self.db_name} | Connections: {len(self.connections)}"


db = DatabaseServer("db-master", "192.168.1.50", "production")
db.start()     # метод від Server
db.connect("admin")
print(db)      # db-master [192.168.1.50:5432] — активний | DB: production | Connections: 1

# Перевірка ієрархії
print(isinstance(db, DatabaseServer))   # True
print(isinstance(db, Server))           # True — db є і Server теж
```

---

### Інкапсуляція

```python
class BankAccount:
    def __init__(self, owner, balance=0):
        self.owner = owner
        self._balance = balance        # одне підкреслення: "не змінюй напряму" (конвенція)
        self.__pin = "0000"            # два підкреслення: name mangling (_BankAccount__pin)

    @property
    def balance(self):
        """Геттер — читаємо _balance як атрибут."""
        return self._balance

    @balance.setter
    def balance(self, amount):
        """Сеттер — з валідацією перед записом."""
        if amount < 0:
            raise ValueError("Баланс не може бути від'ємним")
        self._balance = amount

    def deposit(self, amount):
        if amount <= 0:
            raise ValueError("Сума поповнення має бути більше 0")
        self._balance += amount
        return self._balance

    def withdraw(self, amount):
        if amount > self._balance:
            raise ValueError("Недостатньо коштів")
        self._balance -= amount
        return self._balance


account = BankAccount("Andrii", 1000)
print(account.balance)       # 1000 — через property
account.balance = 2000       # через setter з валідацією
account.deposit(500)
print(account.balance)       # 2500
```

> **Чому `@property`?**  
> Дозволяє звертатись до методу як до атрибута (`account.balance`), але при цьому контролювати читання/запис. Класика ООП: приховати деталі реалізації, надати стабільний інтерфейс.

---

### Статичні методи та методи класу

```python
class MathUtils:
    PI = 3.14159

    @staticmethod
    def add(a, b):
        # Не має доступу до self або cls
        # Просто функція в просторі імен класу
        return a + b

    @classmethod
    def circle_area(cls, radius):
        # cls — посилання на клас (не екземпляр)
        # Може отримати доступ до атрибутів класу
        return cls.PI * radius ** 2


print(MathUtils.add(3, 4))           # 7 — без створення екземпляра
print(MathUtils.circle_area(5))      # 78.53975
```

---

## ⚙️ 4. Бібліотеки та екосистема

### pip та venv

```bash
# Створення ізольованого середовища
python3 -m venv .venv

# Активація
source .venv/bin/activate      # Linux/macOS
.venv\Scripts\activate         # Windows

# Встановлення залежностей
pip install requests flask

# Зберегти поточні залежності
pip freeze > requirements.txt

# Відтворити середовище (на іншій машині)
pip install -r requirements.txt
```

> **Чому `venv`?**  
> Кожен проект має свої залежності. Без venv — конфлікт версій між проектами. З venv — кожен проект в ізоляції.

---

### `requests` — HTTP запити

```python
import requests

# GET запит
response = requests.get(
    "https://api.github.com/users/andriipylypenko",
    headers={"Accept": "application/vnd.github.v3+json"},
    timeout=10   # завжди встановлюй timeout!
)

# Перевірка статусу
response.raise_for_status()   # кине HTTPError якщо 4xx/5xx

data = response.json()        # автоматично парсить JSON
print(data["name"])

# POST запит з JSON тілом
payload = {"username": "andrii", "action": "login"}
response = requests.post(
    "https://api.example.com/auth",
    json=payload,               # автоматично серіалізує та встановить Content-Type
    headers={"Authorization": "Bearer TOKEN"},
    timeout=10
)
```

---

### `json` — серіалізація

```python
import json

# Python dict → JSON рядок
data = {"host": "db-01", "port": 5432, "active": True}
json_string = json.dumps(data, indent=2, ensure_ascii=False)
print(json_string)

# JSON рядок → Python dict
parsed = json.loads(json_string)

# Читання JSON файлу
with open("config.json", "r") as f:
    config = json.load(f)

# Запис JSON файлу
with open("config.json", "w") as f:
    json.dump(config, f, indent=2, ensure_ascii=False)
```

---

### `datetime` — дата і час

```python
from datetime import datetime, timedelta, date

now = datetime.now()
print(now)                              # 2025-06-16 14:30:00.123456
print(now.strftime("%Y-%m-%d %H:%M"))  # "2025-06-16 14:30"

# Парсинг рядка в datetime
dt = datetime.strptime("2025-06-16", "%Y-%m-%d")

# Арифметика
tomorrow = now + timedelta(days=1)
week_ago = now - timedelta(weeks=1)

# Різниця
delta = now - datetime(2025, 1, 1)
print(f"Днів з початку 2025: {delta.days}")

# UTC
from datetime import timezone
utc_now = datetime.now(timezone.utc)
```

---

### `os` та `pathlib` — файлова система

```python
import os
from pathlib import Path

# pathlib — сучасний API (рекомендується)
base = Path("/etc/nginx")
config = base / "nginx.conf"      # оператор / для побудови шляху

print(config.exists())            # True/False
print(config.name)                # "nginx.conf"
print(config.stem)                # "nginx"
print(config.suffix)              # ".conf"
print(config.parent)              # /etc/nginx

# Створення директорій
Path("logs/app").mkdir(parents=True, exist_ok=True)

# Пошук файлів
for conf in Path("/etc").glob("*.conf"):
    print(conf)

# os — для процесів та системних операцій
print(os.getenv("HOME"))          # змінна середовища
print(os.getpid())                # PID поточного процесу
os.environ["DEBUG"] = "true"      # встановити змінну середовища
```

---

## 🌐 5. Backend база: API, HTTP, БД

### HTTP та REST API

| Метод  | Дія              | Ідемпотентний |
|--------|------------------|---------------|
| GET    | Отримати дані    | ✅            |
| POST   | Створити ресурс  | ❌            |
| PUT    | Замінити ресурс  | ✅            |
| PATCH  | Частково змінити | ❌            |
| DELETE | Видалити         | ✅            |

**Коди відповідей:**
- `2xx` — успіх (200 OK, 201 Created, 204 No Content)
- `3xx` — редирект (301 Moved, 302 Found)
- `4xx` — помилка клієнта (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found)
- `5xx` — помилка сервера (500 Internal Error, 502 Bad Gateway, 503 Service Unavailable)

---

### Flask — мінімальний REST API

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

# "База даних" в пам'яті (для прикладу)
servers = [
    {"id": 1, "hostname": "web-01", "active": True},
    {"id": 2, "hostname": "web-02", "active": False},
]

@app.route("/servers", methods=["GET"])
def get_servers():
    """Отримати список усіх серверів."""
    active_only = request.args.get("active", "false").lower() == "true"
    if active_only:
        return jsonify([s for s in servers if s["active"]])
    return jsonify(servers)

@app.route("/servers/<int:server_id>", methods=["GET"])
def get_server(server_id):
    """Отримати конкретний сервер за ID."""
    server = next((s for s in servers if s["id"] == server_id), None)
    if not server:
        return jsonify({"error": "Server not found"}), 404
    return jsonify(server)

@app.route("/servers", methods=["POST"])
def create_server():
    """Створити новий сервер."""
    data = request.get_json()
    if not data or "hostname" not in data:
        return jsonify({"error": "hostname is required"}), 400

    new_server = {
        "id": max(s["id"] for s in servers) + 1,
        "hostname": data["hostname"],
        "active": data.get("active", False),
    }
    servers.append(new_server)
    return jsonify(new_server), 201

if __name__ == "__main__":
    app.run(debug=True, host="0.0.0.0", port=5000)
```

```bash
# Тестування
curl http://localhost:5000/servers
curl http://localhost:5000/servers?active=true
curl -X POST http://localhost:5000/servers \
     -H "Content-Type: application/json" \
     -d '{"hostname": "web-03"}'
```

---

### SQL базово

```python
import sqlite3

# Підключення (sqlite для прикладу, PostgreSQL — аналогічно через psycopg2)
conn = sqlite3.connect("app.db")
cursor = conn.cursor()

# DDL — створення структури
cursor.execute("""
    CREATE TABLE IF NOT EXISTS servers (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        hostname TEXT NOT NULL UNIQUE,
        ip TEXT,
        active BOOLEAN DEFAULT TRUE
    )
""")

# DML — маніпуляція даними
# INSERT — завжди використовуй параметризований запит (захист від SQL injection)
cursor.execute(
    "INSERT INTO servers (hostname, ip) VALUES (?, ?)",
    ("web-01", "192.168.1.10")
)
conn.commit()

# SELECT
cursor.execute("SELECT * FROM servers WHERE active = ?", (True,))
rows = cursor.fetchall()
for row in rows:
    print(row)

# UPDATE
cursor.execute(
    "UPDATE servers SET active = ? WHERE hostname = ?",
    (False, "web-01")
)
conn.commit()

# DELETE
cursor.execute("DELETE FROM servers WHERE hostname = ?", ("web-01",))
conn.commit()

conn.close()
```

> **Чому параметризовані запити?**  
> `"WHERE id = " + user_input` — SQL injection. Зловмисник передає `"1 OR 1=1"` і отримує всю базу.  
> `"WHERE id = ?"` з параметром — драйвер БД екранує значення, injection неможливий.

---

## 🔑 Типові питання на співбесіді

**Q: Яка різниця між `list` і `tuple`?**  
A: list змінюваний, tuple незмінний. Tuple швидший і займає менше пам'яті. Tuple використовують для фіксованих наборів даних і як ключі словників.

**Q: Що таке `*args` і `**kwargs`?**  
A: `*args` — tuple з довільною кількістю позиційних аргументів. `**kwargs` — dict з довільною кількістю іменованих аргументів.

**Q: Що таке `__init__` vs `__new__`?**  
A: `__new__` — створює об'єкт (виділяє пам'ять). `__init__` — ініціалізує вже створений об'єкт. Зазвичай перевизначають тільки `__init__`.

**Q: Яка різниця між `is` і `==`?**  
A: `==` порівнює значення. `is` порівнює ідентичність (чи це один і той самий об'єкт у пам'яті). `None` завжди перевіряють через `is None`, бо `None` — singleton.

**Q: Що таке GIL?**  
A: Global Interpreter Lock — м'ютекс у CPython, який дозволяє виконувати лише один потік Python-коду одночасно. Через GIL threading не дає реального паралелізму для CPU-задач, але нормально працює для I/O-задач. Для CPU-паралелізму — `multiprocessing`.

**Q: Що таке декоратор?**

```python
def log_call(func):
    """Декоратор що логує виклики функції."""
    def wrapper(*args, **kwargs):
        print(f"Виклик: {func.__name__}({args}, {kwargs})")
        result = func(*args, **kwargs)
        print(f"Результат: {result}")
        return result
    return wrapper

@log_call
def add(a, b):
    return a + b

add(2, 3)
# Виклик: add((2, 3), {})
# Результат: 5
```

> Декоратор — це функція що приймає функцію і повертає нову функцію-обгортку. `@log_call` — синтаксичний цукор для `add = log_call(add)`.

---

## Підсумок

| Тема                  | Ключові концепції                                    |
|-----------------------|------------------------------------------------------|
| Типи даних            | int, str, list, dict, set, tuple, bool, None        |
| Керування потоком     | if/elif/else, for, while, break, continue           |
| Функції               | def, return, *args, **kwargs, lambda, декоратори    |
| Структури даних       | list vs tuple vs dict vs set + comprehensions       |
| ООП                   | class, self, __init__, наслідування, @property      |
| Помилки               | try/except/else/finally, raise, власні винятки      |
| Файли і ресурси       | with open(), pathlib, json, os                      |
| HTTP і API            | requests, статус-коди, REST принципи                |
| SQL                   | SELECT/INSERT/UPDATE/DELETE, параметризовані запити |
| Екосистема            | pip, venv, requirements.txt                         |

Вивчи кожну з цих тем до рівня "можу пояснити і написати код без підказок" — і ти готовий до більшості технічних співбесід на Python.
