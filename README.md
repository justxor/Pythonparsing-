# 🐍 Python Парсинг: Большой продвинутый бесплатный курс

> Всё, что нужно знать про парсинг (web scraping) на Python — от основ HTTP до промышленных пауков, обхода защит и работы с динамическими сайтами. С примерами, структурой и практикой.

[![Python](https://img.shields.io/badge/Python-3.10%2B-blue.svg)](https://www.python.org/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](#)

---

## 📚 Содержание

1. [Введение: что такое парсинг](#-урок-1-введение-что-такое-парсинг)
2. [Как устроен веб: HTTP, HTML, DOM](#-урок-2-как-устроен-веб)
3. [Библиотека requests](#-урок-3-библиотека-requests)
4. [Парсинг HTML: BeautifulSoup](#-урок-4-парсинг-html-с-beautifulsoup)
5. [CSS-селекторы и XPath (lxml)](#-урок-5-css-селекторы-и-xpath)
6. [Регулярные выражения](#-урок-6-регулярные-выражения)
7. [Работа с API и JSON](#-урок-7-работа-с-api-и-json)
8. [Динамические сайты: Selenium и Playwright](#-урок-8-динамические-сайты)
9. [Асинхронный парсинг: aiohttp + asyncio](#-урок-9-асинхронный-парсинг)
10. [Scrapy — промышленный фреймворк](#-урок-10-scrapy)
11. [Обход защит и анти-бан стратегии](#-урок-11-обход-защит)
12. [Хранение данных](#-урок-12-хранение-данных)
13. [Этика и законность](#-урок-13-этика-и-законность)
14. [Практические проекты](#-практические-проекты)
15. [Полезные ресурсы](#-полезные-ресурсы)

---

## 🚀 Быстрый старт

```bash
# Создаём виртуальное окружение
python -m venv venv
source venv/bin/activate        # Linux / macOS
venv\Scripts\activate           # Windows

# Устанавливаем основные библиотеки
pip install requests beautifulsoup4 lxml
pip install aiohttp playwright scrapy pandas
playwright install              # браузеры для Playwright
```

---

## 🎓 Урок 1. Введение: что такое парсинг

**Парсинг (web scraping)** — это автоматизированное извлечение данных с веб-страниц и их преобразование в удобный структурированный формат (CSV, JSON, база данных).

**Где применяется:**
- Мониторинг цен и товаров в интернет-магазинах
- Сбор новостей, отзывов, вакансий
- Аналитика рынка и конкурентов
- Наполнение датасетов для машинного обучения

**Общий пайплайн парсера:**

```
1. Запрос (HTTP)      ->  получаем HTML / JSON
2. Парсинг            ->  извлекаем нужные данные
3. Трансформация      ->  чистим и нормализуем
4. Хранение           ->  CSV / JSON / БД
```

> ⚠️ Перед парсингом всегда проверяйте `robots.txt` и условия использования сайта (см. Урок 13).

---

## 🎓 Урок 2. Как устроен веб

Чтобы парсить, нужно понимать, как браузер общается с сервером.

**HTTP-методы:**
| Метод | Назначение |
|-------|-----------|
| `GET` | Получить данные |
| `POST` | Отправить данные (формы, JSON) |
| `PUT` / `PATCH` | Обновить ресурс |
| `DELETE` | Удалить ресурс |

**Коды ответов:**
- `2xx` — успех (`200 OK`)
- `3xx` — редирект (`301`, `302`)
- `4xx` — ошибка клиента (`403 Forbidden`, `404 Not Found`, `429 Too Many Requests`)
- `5xx` — ошибка сервера

**Структура HTML / DOM:**

```html
<html>
  <body>
    <div class="product" id="p1">
      <h2 class="title">Ноутбук</h2>
      <span class="price">59990</span>
    </div>
  </body>
</html>
```

DOM — это дерево, по которому мы «ходим» селекторами, чтобы добраться до нужных узлов.

---

## 🎓 Урок 3. Библиотека requests

`requests` — основной инструмент для синхронных HTTP-запросов.

```python
import requests

# Простой GET-запрос
response = requests.get("https://example.com")

print(response.status_code)   # 200
print(response.encoding)      # utf-8
print(response.text[:200])    # первые 200 символов HTML
```

**Заголовки, параметры и таймауты:**

```python
import requests

headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
                  "AppleWebKit/537.36 (KHTML, like Gecko) "
                  "Chrome/120.0 Safari/537.36",
    "Accept-Language": "ru-RU,ru;q=0.9",
}
params = {"page": 2, "sort": "price"}

response = requests.get(
    "https://example.com/catalog",
    headers=headers,
    params=params,
    timeout=10,
)
response.raise_for_status()   # выбросит исключение при 4xx/5xx
```

**Сессии (переиспользование cookies и соединений):**

```python
with requests.Session() as session:
    session.headers.update({"User-Agent": "my-parser/1.0"})
    session.get("https://example.com/login")
    session.post("https://example.com/login", data={"user": "a", "pwd": "b"})
    # cookies авторизации сохранятся для последующих запросов
    profile = session.get("https://example.com/profile")
```

---

## 🎓 Урок 4. Парсинг HTML с BeautifulSoup

`BeautifulSoup` — самая популярная библиотека для разбора HTML.

```python
import requests
from bs4 import BeautifulSoup

html = requests.get("https://example.com").text
soup = BeautifulSoup(html, "lxml")   # парсер lxml быстрее встроенного

# Поиск одного элемента
title = soup.find("h1")
print(title.text.strip())

# Поиск по классу
price = soup.find("span", class_="price")

# Поиск всех элементов
products = soup.find_all("div", class_="product")
for p in products:
    name = p.find("h2", class_="title").get_text(strip=True)
    cost = p.find("span", class_="price").get_text(strip=True)
    link = p.find("a")["href"]
    print(name, cost, link)
```

**Полезные методы:**
```python
soup.select("div.product")        # CSS-селекторы -> список
soup.select_one("h1.title")       # первый по CSS-селектору
tag.get("href")                   # атрибут (безопаснее, чем tag["href"])
tag.text / tag.get_text()         # текст внутри
tag.attrs                         # словарь всех атрибутов
tag.parent / tag.children         # навигация по дереву
tag.find_next_sibling()           # соседний элемент
```

---

## 🎓 Урок 5. CSS-селекторы и XPath

**CSS-селекторы** (через `.select()` в BeautifulSoup):
```python
soup.select("div.product > h2.title")     # прямой потомок
soup.select("ul li:nth-child(2)")         # второй элемент
soup.select("a[href^='https']")           # атрибут начинается с
soup.select("div.card span.price")        # вложенность
```

**XPath** (через `lxml` — мощнее для сложной навигации):
```python
from lxml import html
import requests

tree = html.fromstring(requests.get("https://example.com").content)

titles = tree.xpath('//div[@class="product"]/h2/text()')
links  = tree.xpath('//a[contains(@class, "more")]/@href')
price  = tree.xpath('//span[@class="price"]/text()')[0]

# Относительный путь и содержит текст
items = tree.xpath('//li[contains(text(), "Скидка")]')
```

| Задача | CSS | XPath |
|--------|-----|-------|
| По классу | `.price` | `//*[@class="price"]` |
| Прямой потомок | `div > p` | `//div/p` |
| По атрибуту | `a[href]` | `//a[@href]` |
| По тексту | — | `//*[contains(text(),"X")]` |

---

## 🎓 Урок 6. Регулярные выражения

Регулярки удобны для извлечения данных из «грязного» текста.

```python
import re

text = "Телефон: +7 (999) 123-45-67, email: user@mail.ru, цена 59 990 руб."

# Телефон
phone = re.search(r"\+7\s?\(?\d{3}\)?[\s-]?\d{3}[\s-]?\d{2}[\s-]?\d{2}", text)
print(phone.group())     # +7 (999) 123-45-67

# Email
emails = re.findall(r"[\w.-]+@[\w.-]+\.\w+", text)

# Цена (числа с пробелами)
price = re.findall(r"\d[\d\s]*\d", text)

# Компиляция для скорости при многократном использовании
pattern = re.compile(r"\d{4}-\d{2}-\d{2}")
dates = pattern.findall("2024-01-15 и 2025-06-14")
```

> 💡 Не парсите весь HTML регулярками — для этого есть BeautifulSoup/lxml. Регулярки хороши для финальной очистки текста.

---

## 🎓 Урок 7. Работа с API и JSON

Часто данные удобнее брать из API, а не из HTML. Откройте DevTools → вкладка **Network → XHR/Fetch**, чтобы найти скрытые запросы.

```python
import requests

# GET к JSON API
resp = requests.get(
    "https://api.example.com/v1/products",
    params={"category": "laptops", "limit": 50},
    headers={"Accept": "application/json"},
)
data = resp.json()          # автоматический разбор JSON в dict/list

for item in data["results"]:
    print(item["name"], item["price"])

# POST с JSON-телом
payload = {"query": "python", "page": 1}
resp = requests.post("https://api.example.com/search", json=payload)
```

**Пагинация по API:**
```python
all_items = []
page = 1
while True:
    r = requests.get("https://api.example.com/items",
                     params={"page": page}).json()
    if not r["items"]:
        break
    all_items.extend(r["items"])
    page += 1
```

---

## 🎓 Урок 8. Динамические сайты

Если данные подгружаются JavaScript-ом, `requests` вернёт пустой HTML. Решение — управлять реальным браузером.

**Playwright (современный, рекомендуется):**
```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()
    page.goto("https://example.com/spa", wait_until="networkidle")

    # Ждём появления элемента
    page.wait_for_selector("div.product")

    # Извлекаем данные
    titles = page.eval_on_selector_all(
        "h2.title", "els => els.map(e => e.textContent)"
    )
    print(titles)

    # Скролл и клик
    page.click("button.load-more")
    page.mouse.wheel(0, 3000)

    browser.close()
```

**Selenium (классика):**
```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

driver = webdriver.Chrome()
driver.get("https://example.com")

# Явное ожидание
element = WebDriverWait(driver, 10).until(
    EC.presence_of_element_located((By.CSS_SELECTOR, "div.product"))
)

products = driver.find_elements(By.CSS_SELECTOR, "div.product")
for p in products:
    print(p.find_element(By.CSS_SELECTOR, ".title").text)

driver.quit()
```

> 💡 Браузерная автоматизация медленная. Сначала проверьте, нет ли скрытого API (Урок 7) — это в разы быстрее.

---

## 🎓 Урок 9. Асинхронный парсинг

Для сотен и тысяч страниц синхронный код слишком медленный. `asyncio` + `aiohttp` позволяют делать запросы параллельно.

```python
import asyncio
import aiohttp
from bs4 import BeautifulSoup

async def fetch(session, url):
    async with session.get(url) as resp:
        return await resp.text()

async def parse(session, url):
    html = await fetch(session, url)
    soup = BeautifulSoup(html, "lxml")
    title = soup.find("h1")
    return title.get_text(strip=True) if title else None

async def main(urls):
    # Ограничиваем число одновременных соединений
    connector = aiohttp.TCPConnector(limit=10)
    async with aiohttp.ClientSession(connector=connector) as session:
        tasks = [parse(session, url) for url in urls]
        return await asyncio.gather(*tasks)

urls = [f"https://example.com/page/{i}" for i in range(1, 101)]
results = asyncio.run(main(urls))
print(results)
```

**Семафор для контроля нагрузки:**
```python
semaphore = asyncio.Semaphore(5)

async def fetch_limited(session, url):
    async with semaphore:           # не более 5 запросов одновременно
        async with session.get(url) as resp:
            return await resp.text()
```

---

## 🎓 Урок 10. Scrapy

`Scrapy` — полноценный фреймворк для больших проектов: очереди запросов, пайплайны, middlewares, экспорт.

```bash
pip install scrapy
scrapy startproject myparser
cd myparser
scrapy genspider books books.toscrape.com
```

**Пример паука:**
```python
import scrapy

class BooksSpider(scrapy.Spider):
    name = "books"
    start_urls = ["https://books.toscrape.com/"]

    def parse(self, response):
        for book in response.css("article.product_pod"):
            yield {
                "title": book.css("h3 a::attr(title)").get(),
                "price": book.css(".price_color::text").get(),
                "rating": book.css("p.star-rating::attr(class)").get(),
            }
        # Переход на следующую страницу
        next_page = response.css("li.next a::attr(href)").get()
        if next_page:
            yield response.follow(next_page, callback=self.parse)
```

```bash
# Запуск с экспортом в JSON
scrapy crawl books -o books.json
```

**Сильные стороны Scrapy:** автоматическая обработка очередей, повторов, ограничение скорости (`AUTOTHROTTLE`), пайплайны для очистки и сохранения данных.

---

## 🎓 Урок 11. Обход защит

Сайты используют разные методы защиты от ботов. Легальные техники аккуратного парсинга:

**1. Реалистичные заголовки и User-Agent:**
```python
headers = {
    "User-Agent": "Mozilla/5.0 ...",
    "Accept": "text/html,application/xhtml+xml,...",
    "Referer": "https://google.com",
}
```

**2. Задержки и рандомизация:**
```python
import time, random
time.sleep(random.uniform(1.5, 4.0))   # имитация поведения человека
```

**3. Ротация User-Agent:**
```python
import random
USER_AGENTS = [
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...",
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15) ...",
]
headers = {"User-Agent": random.choice(USER_AGENTS)}
```

**4. Прокси:**
```python
proxies = {"http": "http://user:pass@host:port",
           "https": "http://user:pass@host:port"}
requests.get(url, proxies=proxies, timeout=10)
```

**5. Повторные попытки с backoff:**
```python
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

session = requests.Session()
retries = Retry(total=5, backoff_factor=1,
                status_forcelist=[429, 500, 502, 503, 504])
session.mount("https://", HTTPAdapter(max_retries=retries))
```

> ⚠️ Уважайте `robots.txt`, лимиты сервера и закон. Обход CAPTCHA и систем защиты против воли владельца сайта может быть незаконным. Парсите ответственно.

---

## 🎓 Урок 12. Хранение данных

**CSV:**
```python
import csv

with open("data.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["title", "price"])
    writer.writeheader()
    writer.writerows([{"title": "A", "price": 100}])
```

**JSON:**
```python
import json

with open("data.json", "w", encoding="utf-8") as f:
    json.dump(data, f, ensure_ascii=False, indent=2)
```

**pandas (удобно для анализа и экспорта):**
```python
import pandas as pd

df = pd.DataFrame(items)
df.to_csv("data.csv", index=False)
df.to_excel("data.xlsx", index=False)
```

**SQLite (для больших объёмов):**
```python
import sqlite3

conn = sqlite3.connect("data.db")
cur = conn.cursor()
cur.execute("CREATE TABLE IF NOT EXISTS products (title TEXT, price INTEGER)")
cur.executemany("INSERT INTO products VALUES (?, ?)",
                [(i["title"], i["price"]) for i in items])
conn.commit()
conn.close()
```

---

## 🎓 Урок 13. Этика и законность

Парсинг — мощный инструмент, и его нужно использовать ответственно.

**Чек-лист ответственного парсинга:**
- ✅ Проверяйте `robots.txt` (`https://site.com/robots.txt`) и условия использования.
- ✅ Не создавайте чрезмерную нагрузку — добавляйте задержки и ограничивайте параллелизм.
- ✅ Не собирайте персональные данные без правового основания (учитывайте GDPR, 152-ФЗ и др.).
- ✅ Указывайте честный User-Agent с контактом, если это уместно.
- ✅ Кэшируйте результаты, чтобы не дёргать сайт повторно.
- ❌ Не обходите платный доступ, авторизацию и системы защиты против воли владельца.
- ❌ Не публикуйте и не перепродавайте чужой контент, нарушая авторские права.

**Проверка robots.txt в коде:**
```python
import urllib.robotparser

rp = urllib.robotparser.RobotFileParser()
rp.set_url("https://example.com/robots.txt")
rp.read()
print(rp.can_fetch("*", "https://example.com/catalog"))  # True / False
```

> ⚖️ Это образовательный материал, а не юридическая консультация. Законы различаются по странам — при коммерческом парсинге консультируйтесь с юристом.

---

## 🛠 Практические проекты

Тренируйтесь на специальных «песочницах» — они созданы для легального обучения парсингу:

| Проект | Чему учит | Сайт для практики |
|--------|-----------|-------------------|
| Парсер книг | BeautifulSoup, пагинация | `books.toscrape.com` |
| Парсер цитат | CSS-селекторы, JS-версия | `quotes.toscrape.com` |
| Мониторинг цен | requests + расписание | свой тестовый сайт |
| Сбор вакансий через API | JSON, пагинация | публичные открытые API |
| Асинхронный парсер новостей | aiohttp, asyncio | RSS-ленты |
| Парсер на Scrapy | пайплайны, экспорт | `books.toscrape.com` |

**Мини-проект «Парсер книг» целиком:**
```python
import csv
import requests
from bs4 import BeautifulSoup

BASE = "https://books.toscrape.com/catalogue/page-{}.html"
rows = []

for page in range(1, 4):
    soup = BeautifulSoup(requests.get(BASE.format(page)).text, "lxml")
    for book in soup.select("article.product_pod"):
        rows.append({
            "title": book.h3.a["title"],
            "price": book.select_one(".price_color").text,
        })

with open("books.csv", "w", newline="", encoding="utf-8") as f:
    w = csv.DictWriter(f, fieldnames=["title", "price"])
    w.writeheader()
    w.writerows(rows)

print(f"Сохранено книг: {len(rows)}")
```

---

## 📖 Полезные ресурсы

**Документация:**
- [requests](https://requests.readthedocs.io/)
- [BeautifulSoup](https://www.crummy.com/software/BeautifulSoup/bs4/doc/)
- [lxml](https://lxml.de/)
- [Scrapy](https://docs.scrapy.org/)
- [Playwright for Python](https://playwright.dev/python/)
- [aiohttp](https://docs.aiohttp.org/)

**Сайты для практики:**
- [books.toscrape.com](https://books.toscrape.com/)
- [quotes.toscrape.com](https://quotes.toscrape.com/)
- [httpbin.org](https://httpbin.org/) — тестирование запросов

**Инструменты:**
- DevTools браузера (вкладка Network) — поиск скрытых API
- [regex101.com](https://regex101.com/) — отладка регулярок
- Postman / [Insomnia](https://insomnia.rest/) — работа с API

---

## 🗺 Дорожная карта обучения

```
Новичок      ->  Уроки 1-4   (requests + BeautifulSoup)
Средний      ->  Уроки 5-7   (CSS/XPath, регулярки, API)
Продвинутый  ->  Уроки 8-11  (динамика, async, Scrapy, защиты)
Профи        ->  Уроки 12-13 + свои production-проекты
```

---

## 🤝 Вклад в проект

PR и предложения приветствуются! Открывайте Issue с идеями новых уроков или примеров.

## 📄 Лицензия

Материалы распространяются под лицензией **MIT** — используйте свободно в учебных целях.

---

⭐ Если курс оказался полезным — поставьте звезду репозиторию!
