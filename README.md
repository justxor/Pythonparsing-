# 🐍 Python Парсинг: Большой продвинутый бесплатный курс

> Полное практическое руководство по веб-скрейпингу на Python — от основ HTTP до production-grade пауков, обхода антибот-защит, асинхронности и проектирования надёжных пайплайнов. Каждый раздел содержит рабочие примеры, типовые ошибки и продвинутые практики.

[![Python](https://img.shields.io/badge/Python-3.10%2B-blue.svg)](https://www.python.org/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](#)
[![Made with ❤](https://img.shields.io/badge/Made%20with-%E2%9D%A4-red.svg)](#)

---

## 📚 Содержание

**Основы**
1. [Введение: что такое парсинг](#-урок-1-введение-что-такое-парсинг)
2. [Как устроен веб: HTTP, HTML, DOM](#-урок-2-как-устроен-веб)
3. [Библиотека requests: глубокое погружение](#-урок-3-библиотека-requests)
4. [Парсинг HTML: BeautifulSoup](#-урок-4-парсинг-html-с-beautifulsoup)
5. [CSS-селекторы и XPath (lxml)](#-урок-5-css-селекторы-и-xpath)
6. [Регулярные выражения](#-урок-6-регулярные-выражения)

**Продвинутый уровень**
7. [Работа с API, JSON и авторизацией](#-урок-7-работа-с-api-и-json)
8. [Динамические сайты: Playwright и Selenium](#-урок-8-динамические-сайты)
9. [Асинхронный парсинг: aiohttp + asyncio](#-урок-9-асинхронный-парсинг)
10. [Scrapy — промышленный фреймворк](#-урок-10-scrapy)
11. [Обход защит и анти-бан стратегии](#-урок-11-обход-защит)

**Инженерия данных**
12. [Хранение и валидация данных](#-урок-12-хранение-данных)
13. [Надёжность: ошибки, ретраи, логирование](#-урок-13-надёжность-и-обработка-ошибок)
14. [Тестирование парсеров](#-урок-14-тестирование-парсеров)
15. [Этика и законность](#-урок-15-этика-и-законность)

**Практика**
16. [Архитектура production-парсера](#-урок-16-архитектура-production-парсера)
17. [Практические проекты](#-практические-проекты)
18. [Антипаттерны и частые ошибки](#-антипаттерны-и-частые-ошибки)
19. [Полезные ресурсы](#-полезные-ресурсы)

---

## 🚀 Быстрый старт

```bash
# Создаём виртуальное окружение
python -m venv venv
source venv/bin/activate        # Linux / macOS
venv\Scripts\activate           # Windows

# Базовый стек
pip install requests beautifulsoup4 lxml
# Продвинутый стек
pip install aiohttp[speedups] playwright scrapy pandas pydantic tenacity
pip install fake-useragent loguru httpx selectolax
playwright install chromium

# Зафиксировать зависимости
pip freeze > requirements.txt
```

> 💡 `selectolax` — очень быстрый HTML-парсер на C (в 5–10 раз быстрее BeautifulSoup), `httpx` — современная замена requests с поддержкой async и HTTP/2.

---

## 🎓 Урок 1. Введение: что такое парсинг

**Парсинг (web scraping)** — автоматизированное извлечение данных с веб-страниц и их превращение в структурированный формат.

**Когда парсинг оправдан:**
- У сайта нет публичного API, а данные нужны в объёме.
- Нужен мониторинг изменений (цены, наличие, рейтинги).
- Сбор обучающих датасетов.

**Когда лучше НЕ парсить:**
- Есть официальный API — используйте его (стабильнее и легально).
- Данные защищены авторским правом и нет лицензии.
- Объём нагрузки навредит сайту.

**Архитектура любого парсера:**

```
┌──────────┐   ┌──────────┐   ┌────────────┐   ┌──────────┐   ┌─────────┐
│ Источник │ → │  Загруз- │ → │  Парсинг   │ → │ Валидация│ → │Хранение │
│  (URL)   │   │   чик    │   │ (extract)  │   │ (clean)  │   │ (store) │
└──────────┘   └──────────┘   └────────────┘   └──────────┘   └─────────┘
     ↑              │ retry         │ schema         │ dedupe       │
     └──────────────┘ rate-limit    │ normalize      │              │
```

Хороший парсер разделяет эти слои: загрузку, извлечение, валидацию и сохранение — так его легко тестировать и поддерживать.

> ⚠️ Перед началом всегда читайте `robots.txt` и Terms of Service (см. Урок 15).

---

## 🎓 Урок 2. Как устроен веб

**Жизненный цикл запроса:**

```
Клиент ──HTTP-запрос──▶ DNS ──▶ TCP/TLS ──▶ Сервер
Клиент ◀─HTTP-ответ─── (статус + заголовки + тело)
```

**HTTP-методы:**

| Метод | Назначение | Идемпотентен |
|-------|-----------|:------------:|
| `GET` | Получить данные | ✅ |
| `POST` | Создать / отправить | ❌ |
| `PUT` | Полностью заменить | ✅ |
| `PATCH` | Частично обновить | ❌ |
| `DELETE` | Удалить | ✅ |

**Ключевые коды ответов для парсера:**
- `200` — OK, `204` — нет содержимого
- `301/302` — редирект (следите за `Location`)
- `401/403` — нужна авторизация / доступ запрещён
- `404` — не найдено
- `429` — слишком много запросов (читайте заголовок `Retry-After`!)
- `5xx` — ошибка сервера (имеет смысл повторить запрос)

**Важные заголовки:**
```
User-Agent      — кто делает запрос (браузер/бот)
Cookie          — сессия, авторизация
Referer         — откуда пришёл запрос
Accept          — какой формат ждём (text/html, application/json)
Content-Type    — формат тела запроса
Retry-After     — через сколько повторить (при 429/503)
```

**DOM как дерево:**
```html
<article class="product" data-id="42">
  <h2 class="title">Ноутбук Pro</h2>
  <span class="price" data-currency="RUB">59990</span>
  <a href="/product/42">Подробнее</a>
</article>
```
Здесь `data-id` и `data-currency` — атрибуты данных, которые часто удобнее парсить, чем видимый текст.

---

## 🎓 Урок 3. Библиотека requests

### Базовые запросы
```python
import requests

r = requests.get("https://httpbin.org/get", timeout=10)
print(r.status_code, r.elapsed.total_seconds())
print(r.json())          # если ответ JSON
print(r.headers['Content-Type'])
```

### Сессия с настройками по умолчанию (рекомендуется)
```python
import requests

def make_session() -> requests.Session:
    s = requests.Session()
    s.headers.update({
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
                      "AppleWebKit/537.36 (KHTML, like Gecko) "
                      "Chrome/120.0 Safari/537.36",
        "Accept-Language": "ru-RU,ru;q=0.9",
    })
    return s

session = make_session()
resp = session.get("https://example.com", timeout=(3.05, 27))  # (connect, read)
```

> 💡 Таймаут — кортеж `(connect, read)`. Никогда не делайте запрос без таймаута: зависший сокет повесит весь парсер.

### Автоматические ретраи с backoff
```python
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

retry = Retry(
    total=5,
    backoff_factor=1,                 # 1s, 2s, 4s, 8s...
    status_forcelist=[429, 500, 502, 503, 504],
    allowed_methods=["GET", "POST"],
    respect_retry_after_header=True,
)
session.mount('https://', HTTPAdapter(max_retries=retry))
session.mount('http://', HTTPAdapter(max_retries=retry))
```

### Потоковая загрузка больших файлов
```python
with session.get("https://example.com/big.csv", stream=True) as r:
    r.raise_for_status()
    with open("big.csv", "wb") as f:
        for chunk in r.iter_content(chunk_size=8192):
            f.write(chunk)
```

### Авторизация
```python
# Basic Auth
session.get(url, auth=("user", "password"))

# Bearer-токен
session.headers["Authorization"] = "Bearer <token>"

# Формы логина (CSRF-токен часто прячут в HTML)
login_page = session.get("https://example.com/login")
# ... извлекаем csrf из login_page ...
session.post("https://example.com/login",
             data={"username": "u", "password": "p", "csrf": csrf})
```

---

## 🎓 Урок 4. Парсинг HTML с BeautifulSoup

### Поиск элементов — все способы
```python
from bs4 import BeautifulSoup

soup = BeautifulSoup(html, "lxml")

# По тегу/классу/id
soup.find("h1")
soup.find_all("div", class_="product", limit=10)
soup.find(id="main")

# По нескольким классам и атрибутам
soup.find_all("a", class_=["btn", "primary"])
soup.find_all(attrs={"data-id": True})            # есть атрибут data-id
soup.find_all("input", attrs={"type": "hidden"})

# CSS-селекторы (мощно)
soup.select("div.card > h2.title")
soup.select("article[data-id] span.price")
soup.select_one("nav a:nth-of-type(2)")
```

### Безопасное извлечение (защита от None)
```python
def safe_text(node, selector, default=None):
    """Возвращает текст по CSS-селектору или default, не падая на None."""
    el = node.select_one(selector)
    return el.get_text(strip=True) if el else default

def safe_attr(node, selector, attr, default=None):
    el = node.select_one(selector)
    return el.get(attr, default) if el else default

# Использование
for card in soup.select('article.product'):
    product = {
        "title": safe_text(card, "h2.title"),
        "price": safe_text(card, "span.price", default="0"),
        "url": safe_attr(card, "a", "href"),
    }
    print(product)
```

> 💡 90% «случайных» падений парсера — это `AttributeError: NoneType has no attribute`. Хелперы `safe_*` решают проблему раз и навсегда.

### Навигация по дереву
```python
price = soup.find("span", class_="price")
price.parent                 # родитель
price.find_parent('article') # ближайший предок-article
price.next_sibling           # следующий узел
price.find_next('a')         # следующая ссылка в документе
[c for c in price.parents]   # все предки до корня
```

### Извлечение таблиц целиком
```python
import pandas as pd

# pandas сам распарсит все <table> на странице в список DataFrame
tables = pd.read_html(html)
df = tables[0]
```

---

## 🎓 Урок 5. CSS-селекторы и XPath

### Шпаргалка соответствий
| Задача | CSS | XPath |
|--------|-----|-------|
| По классу | `.price` | `//*[@class="price"]` |
| Прямой потомок | `div > p` | `//div/p` |
| Любой потомок | `div p` | `//div//p` |
| N-й элемент | `li:nth-child(2)` | `(//li)[2]` |
| По атрибуту | `a[href]` | `//a[@href]` |
| Атрибут начинается с | `a[href^="/p"]` | `//a[starts-with(@href,"/p")]` |
| Содержит текст | — | `//*[contains(text(),"X")]` |
| Родитель | — | `//span/..` |

### lxml + XPath (быстро и мощно)
```python
from lxml import html

tree = html.fromstring(page_html)

# Извлекаем сразу списками
titles = tree.xpath('//article[@class="product"]/h2/text()')
prices = tree.xpath('//span[@class="price"]/text()')
links  = tree.xpath('//a[contains(@class,"more")]/@href')

# Относительные пути от найденного узла
for card in tree.xpath('//article[@class="product"]'):
    title = card.xpath('.//h2/text()')          # точка = относительно card
    price = card.xpath('.//span[@class="price"]/text()')
    print(title, price)

# XPath по тексту и осям
tree.xpath('//th[text()="Цена"]/following-sibling::td/text()')
```

> 💡 XPath-ось `following-sibling` незаменима для таблиц «ключ-значение», где данные лежат в соседней ячейке.

### selectolax — когда нужна скорость
```python
from selectolax.parser import HTMLParser

tree = HTMLParser(page_html)
for node in tree.css('article.product'):
    title = node.css_first('h2.title')
    print(title.text(strip=True) if title else None)
```

---

## 🎓 Урок 6. Регулярные выражения

Регулярки — для финальной очистки текста, НЕ для парсинга структуры HTML.

```python
import re

text = "Тел: +7 (999) 123-45-67, email: a.user@mail.ru, цена 59 990 ₽, скидка 15%"

# Именованные группы — читаемее
phone = re.search(r"\+7\s?\(?(?P<code>\d{3})\)?[\s-]?(?P<num>\d{3}[\s-]?\d{2}[\s-]?\d{2})", text)
if phone:
    print(phone.group("code"), phone.group("num"))

# Цена: убираем пробелы-разделители и приводим к int
raw = re.search(r"\d[\d\s]*\d", text).group()
price = int(re.sub(r"\s", "", raw))   # 59990

# Все email
emails = re.findall(r"[\w.+-]+@[\w-]+\.[\w.-]+", text)

# Компиляция + флаги для многократного использования
TAG_RE = re.compile(r"<[^>]+>")
clean = TAG_RE.sub("", "<b>Привет</b>")     # удалить теги: Привет
```

**Полезные приёмы:**
```python
re.findall(r"(?<=id=)\d+", "id=42 id=99")   # lookbehind -> [42, 99]
re.split(r"[,;]\s*", "a, b; c")             # ["a","b","c"]
```

---

## 🎓 Урок 7. Работа с API и JSON

Часто данные уже есть в JSON — откройте DevTools → **Network → Fetch/XHR** и найдите запрос, который возвращает нужные данные. Это в разы быстрее и стабильнее парсинга HTML.

### Базовый цикл с пагинацией
```python
import requests

def fetch_all(base_url: str, session: requests.Session) -> list[dict]:
    items, page = [], 1
    while True:
        r = session.get(base_url, params={'page': page, 'per_page': 100}, timeout=15)
        r.raise_for_status()
        data = r.json()
        if not data['results']:
            break
        items.extend(data['results'])
        # курсорная пагинация:
        if not data.get('next'):
            break
        page += 1
    return items
```

### Безопасный доступ к вложенному JSON
```python
def deep_get(d: dict, path: str, default=None):
    """deep_get(obj, "user.address.city")"""
    cur = d
    for key in path.split("."):
        if isinstance(cur, dict) and key in cur:
            cur = cur[key]
        else:
            return default
    return cur

city = deep_get(payload, "user.address.city", default="—")
```

### GraphQL и POST-API
```python
query = {
    "query": "{ products(first: 50) { edges { node { title price } } } }"
}
r = session.post("https://shop.example/graphql", json=query, timeout=15)
nodes = [e['node'] for e in r.json()['data']['products']['edges']]
```

> 💡 Если API требует подпись/токен — он почти всегда виден в заголовках запроса в DevTools. Скопируйте запрос как cURL (правой кнопкой → Copy as cURL) и преобразуйте в Python через сайт `curlconverter.com`.

---

## 🎓 Урок 8. Динамические сайты

Если контент рендерится JS — `requests` вернёт пустой каркас. Нужен реальный браузер.

### Playwright (рекомендуется)
```python
from playwright.sync_api import sync_playwright

def scrape_spa(url: str) -> list[str]:
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        ctx = browser.new_context(
            user_agent="Mozilla/5.0 ... Chrome/120.0 Safari/537.36",
            viewport={'width': 1920, 'height': 1080},
            locale='ru-RU',
        )
        page = ctx.new_page()
        page.goto(url, wait_until='networkidle', timeout=30000)

        # Ждём конкретный селектор, а не фиксированный sleep
        page.wait_for_selector('article.product')

        # Бесконечный скролл
        prev = 0
        while True:
            page.mouse.wheel(0, 5000)
            page.wait_for_timeout(1000)
            count = page.locator('article.product').count()
            if count == prev:
                break
            prev = count

        titles = page.locator('h2.title').all_inner_texts()
        browser.close()
        return titles
```

### Перехват сетевых ответов (мощнейшая техника)
```python
# Вместо парсинга DOM — ловим JSON, который грузит сама страница
captured = []

def handle_response(response):
    if '/api/products' in response.url:
        captured.append(response.json())

page.on('response', handle_response)
page.goto(url, wait_until='networkidle')
# captured теперь содержит чистый JSON без всякого HTML
```

### Selenium (если требуется legacy-совместимость)
```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

opts = webdriver.ChromeOptions()
opts.add_argument('--headless=new')
opts.add_argument('--disable-blink-features=AutomationControlled')
driver = webdriver.Chrome(options=opts)
try:
    driver.get(url)
    el = WebDriverWait(driver, 15).until(
        EC.presence_of_element_located((By.CSS_SELECTOR, 'article.product')))
    for p in driver.find_elements(By.CSS_SELECTOR, 'article.product'):
        print(p.find_element(By.CSS_SELECTOR, '.title').text)
finally:
    driver.quit()
```

> ⚡ **Порядок выбора инструмента:** скрытый API (Урок 7) → перехват ответов в Playwright → парсинг DOM. Браузер в 50–100 раз медленнее HTTP-запроса.

---

## 🎓 Урок 9. Асинхронный парсинг

Для тысяч страниц asyncio + aiohttp дают кратный прирост скорости.

### Полноценный асинхронный воркер с ограничением скорости
```python
import asyncio
import aiohttp
from bs4 import BeautifulSoup

class AsyncScraper:
    def __init__(self, concurrency: int = 10, delay: float = 0.2):
        self.sem = asyncio.Semaphore(concurrency)
        self.delay = delay

    async def fetch(self, session: aiohttp.ClientSession, url: str) -> str | None:
        async with self.sem:                       # ограничение параллелизма
            try:
                async with session.get(url, timeout=aiohttp.ClientTimeout(total=20)) as r:
                    if r.status == 429:
                        await asyncio.sleep(int(r.headers.get('Retry-After', 5)))
                        return await self.fetch(session, url)
                    r.raise_for_status()
                    await asyncio.sleep(self.delay)
                    return await r.text()
            except (aiohttp.ClientError, asyncio.TimeoutError) as e:
                print(f'[!] {url}: {e}')
                return None

    def parse(self, html: str) -> dict:
        soup = BeautifulSoup(html, 'lxml')
        h1 = soup.find('h1')
        return {'title': h1.get_text(strip=True) if h1 else None}

    async def run(self, urls: list[str]) -> list[dict]:
        connector = aiohttp.TCPConnector(limit=20, ttl_dns_cache=300)
        async with aiohttp.ClientSession(connector=connector) as session:
            htmls = await asyncio.gather(*(self.fetch(session, u) for u in urls))
        return [self.parse(h) for h in htmls if h]

urls = [f'https://example.com/page/{i}' for i in range(1, 1001)]
results = asyncio.run(AsyncScraper(concurrency=15).run(urls))
print(f'Собрано: {len(results)}')
```

### Прогресс-бар для длинных задач
```python
from tqdm.asyncio import tqdm_asyncio

htmls = await tqdm_asyncio.gather(*(self.fetch(s, u) for u in urls))
```

> ⚠️ Async ускоряет I/O, но не парсинг CPU. Тяжёлый разбор HTML выносите в `loop.run_in_executor` или `ProcessPoolExecutor`.

---

## 🎓 Урок 10. Scrapy

Scrapy — полноценный фреймворк: очереди, ретраи, throttling, пайплайны, экспорт «из коробки».

```bash
scrapy startproject shop && cd shop
scrapy genspider books books.toscrape.com
```

### Паук с пагинацией и переходом на карточку
```python
import scrapy

class BooksSpider(scrapy.Spider):
    name = 'books'
    start_urls = ['https://books.toscrape.com/']
    custom_settings = {
        'DOWNLOAD_DELAY': 0.5,
        'AUTOTHROTTLE_ENABLED': True,
        'CONCURRENT_REQUESTS': 8,
    }

    def parse(self, response):
        for book in response.css('article.product_pod'):
            detail_url = book.css('h3 a::attr(href)').get()
            yield response.follow(detail_url, callback=self.parse_book)
        next_page = response.css('li.next a::attr(href)').get()
        if next_page:
            yield response.follow(next_page, callback=self.parse)

    def parse_book(self, response):
        yield {
            'title': response.css('h1::text').get(),
            'price': response.css('p.price_color::text').get(),
            'stock': response.css('p.availability::text').re_first(r'' + bs+'d+'),
            'desc':  response.css('#product_description + p::text').get(),
        }
```

### Item Pipeline — валидация и очистка
```python
# pipelines.py
from itemadapter import ItemAdapter

class PricePipeline:
    def process_item(self, item, spider):
        adapter = ItemAdapter(item)
        price = adapter.get('price', '')
        adapter['price'] = float(price.replace('£', '').strip() or 0)
        if adapter['price'] <= 0:
            from scrapy.exceptions import DropItem
            raise DropItem('Нет цены')
        return item

# settings.py
ITEM_PIPELINES = {'shop.pipelines.PricePipeline': 300}
```

```bash
scrapy crawl books -o books.json --logfile scrape.log
```

---

## 🎓 Урок 11. Обход защит

Легальные техники аккуратного и устойчивого парсинга.

### Ротация User-Agent и заголовков
```python
from fake_useragent import UserAgent
ua = UserAgent()

def realistic_headers() -> dict:
    return {
        'User-Agent': ua.random,
        'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
        'Accept-Language': 'ru-RU,ru;q=0.9,en;q=0.8',
        'Accept-Encoding': 'gzip, deflate, br',
        'Connection': 'keep-alive',
        'Upgrade-Insecure-Requests': '1',
    }
```

### Пул прокси с ротацией и проверкой
```python
import random, itertools

class ProxyPool:
    def __init__(self, proxies: list[str]):
        self.proxies = proxies
        self.bad = set()

    def get(self) -> str | None:
        alive = [p for p in self.proxies if p not in self.bad]
        return random.choice(alive) if alive else None

    def mark_bad(self, proxy: str):
        self.bad.add(proxy)

pool = ProxyPool(['http://1.2.3.4:8080', 'http://5.6.7.8:3128'])
proxy = pool.get()
try:
    r = requests.get(url, proxies={'http': proxy, 'https': proxy}, timeout=10)
except requests.RequestException:
    pool.mark_bad(proxy)
```

### Вежливый rate-limiting
```python
import time, random

class RateLimiter:
    def __init__(self, rps: float = 1.0):
        self.min_interval = 1.0 / rps
        self.last = 0.0

    def wait(self):
        elapsed = time.monotonic() - self.last
        sleep_for = self.min_interval - elapsed + random.uniform(0, 0.3)
        if sleep_for > 0:
            time.sleep(sleep_for)
        self.last = time.monotonic()
```

### Маскировка автоматизации в браузере
```python
# Playwright: убираем navigator.webdriver
context.add_init_script(
    "Object.defineProperty(navigator,'webdriver',{get:()=>undefined})")
# Либо используйте playwright-stealth / undetected-chromedriver
```

> ⚠️ Уважайте `robots.txt`, лимиты и закон. Обход CAPTCHA и защит против воли владельца сайта может быть незаконным. Эти техники — для снижения нагрузки и стабильности, а не для атак.

---


## 🎓 Урок 12. Хранение данных

### Валидация через dataclass / pydantic
```python
from pydantic import BaseModel, field_validator, HttpUrl

class Product(BaseModel):
    title: str
    price: float
    url: HttpUrl
    in_stock: bool = True

    @field_validator('price')
    @classmethod
    def price_positive(cls, v):
        if v < 0:
            raise ValueError('Цена не может быть отрицательной')
        return round(v, 2)

# Невалидные записи отсеются автоматически
raw = {'title': 'Книга', 'price': '19.99', 'url': 'https://shop/x'}
product = Product(**raw)        # price приведётся к float
print(product.model_dump())
```

### CSV / JSON Lines (потоковая запись)
```python
import csv, json

# JSON Lines — удобно дозаписывать построчно, не держа всё в памяти
with open('data.jsonl', 'a', encoding='utf-8') as f:
    for item in items:
        f.write(json.dumps(item, ensure_ascii=False) + '\n')

# CSV
with open('data.csv', 'w', newline='', encoding='utf-8-sig') as f:
    w = csv.DictWriter(f, fieldnames=['title', 'price'])
    w.writeheader()
    w.writerows(items)
```
> 💡 `utf-8-sig` добавляет BOM — Excel корректно откроет кириллицу.

### SQLite с защитой от дублей (UPSERT)
```python
import sqlite3

conn = sqlite3.connect('shop.db')
conn.execute('''
    CREATE TABLE IF NOT EXISTS products (
        url   TEXT PRIMARY KEY,
        title TEXT,
        price REAL,
        updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )''')

conn.executemany('''
    INSERT INTO products (url, title, price) VALUES (:url, :title, :price)
    ON CONFLICT(url) DO UPDATE SET
        title=excluded.title, price=excluded.price,
        updated_at=CURRENT_TIMESTAMP
''', items)
conn.commit()
```

### pandas для постобработки
```python
import pandas as pd

df = pd.DataFrame(items)
df = df.drop_duplicates(subset='url').dropna(subset=['price'])
df['price'] = pd.to_numeric(df['price'], errors='coerce')
df.to_parquet('data.parquet')      # компактно и быстро для больших данных
```

---

## 🎓 Урок 13. Надёжность и обработка ошибок

### Декоратор ретраев через tenacity
```python
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type
import requests

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential(multiplier=1, min=2, max=30),
    retry=retry_if_exception_type(requests.RequestException),
    reraise=True,
)
def fetch(url: str, session: requests.Session) -> str:
    r = session.get(url, timeout=15)
    r.raise_for_status()
    return r.text
```

### Структурное логирование (loguru)
```python
from loguru import logger

logger.add('scraper.log', rotation='10 MB', retention='7 days', level='INFO')

logger.info('Старт: {} URL', len(urls))
try:
    html = fetch(url, session)
except Exception as e:
    logger.error('Ошибка на {}: {}', url, e)
```

### Чекпоинты — продолжение после сбоя
```python
import json, os

def load_done(path='done.json') -> set:
    return set(json.load(open(path))) if os.path.exists(path) else set()

def save_done(done: set, path='done.json'):
    json.dump(list(done), open(path, 'w'))

done = load_done()
for url in urls:
    if url in done:
        continue            # пропускаем уже обработанное
    process(url)
    done.add(url)
    if len(done) % 50 == 0:
        save_done(done)     # периодически сохраняем прогресс
```

> 💡 Чекпоинты обязательны для долгих задач: если парсер упадёт на 9000-й из 10000 страниц, вы не начнёте с нуля.

---

## 🎓 Урок 14. Тестирование парсеров

Парсеры ломаются, когда сайт меняет вёрстку. Тесты на сохранённом HTML ловят это мгновенно.

```python
# tests/test_parser.py
import pytest
from myparser import parse_product

@pytest.fixture
def sample_html():
    return open('tests/fixtures/product.html', encoding='utf-8').read()

def test_parse_extracts_title(sample_html):
    result = parse_product(sample_html)
    assert result['title'] == 'Ноутбук Pro'
    assert result['price'] == 59990.0

def test_parse_handles_missing_price():
    html = '<article><h2>Без цены</h2></article>'
    result = parse_product(html)
    assert result['price'] is None      # не падает, возвращает None
```

### Мокаем HTTP, чтобы не ходить в сеть
```python
import responses, requests

@responses.activate
def test_fetch_retries_on_500():
    responses.add(responses.GET, 'https://x.com', status=500)
    responses.add(responses.GET, 'https://x.com', body='OK', status=200)
    # ваш fetch с ретраями должен вернуть 'OK'
```

> 💡 Сохраняйте «эталонные» HTML-страницы в `tests/fixtures/`. При изменении сайта обновляйте их и сразу видите, что сломалось.

---

## 🎓 Урок 15. Этика и законность

**Чек-лист ответственного парсинга:**
- ✅ Читайте `robots.txt` и Terms of Service.
- ✅ Делайте задержки, не кладите сайт нагрузкой.
- ✅ Кэшируйте — не запрашивайте одно и то же дважды.
- ✅ Честный `User-Agent`, при возможности — с контактом.
- ✅ Учитывайте GDPR / 152-ФЗ при персональных данных.
- ❌ Не обходите платный доступ и авторизацию против воли владельца.
- ❌ Не перепродавайте чужой контент, нарушая авторские права.

```python
import urllib.robotparser as urp

rp = urp.RobotFileParser()
rp.set_url('https://example.com/robots.txt')
rp.read()
if rp.can_fetch('*', 'https://example.com/catalog'):
    ...   # парсим
crawl_delay = rp.crawl_delay('*')   # уважайте указанную задержку
```

> ⚖️ Это образовательный материал, а не юридическая консультация. При коммерческом использовании консультируйтесь с юристом.

---

## 🎓 Урок 16. Архитектура production-парсера

Соберём всё вместе — модульный, тестируемый, устойчивый парсер.

```
project/
├── scraper/
│   ├── __init__.py
│   ├── client.py      # HTTP: сессия, ретраи, rate-limit, прокси
│   ├── parsers.py     # чистые функции HTML -> dict (легко тестировать)
│   ├── models.py      # pydantic-схемы валидации
│   ├── storage.py     # запись в БД/файлы, дедупликация
│   └── pipeline.py    # оркестрация: fetch -> parse -> validate -> store
├── tests/
│   └── fixtures/
├── config.py          # настройки, секреты из переменных окружения
└── main.py
```

```python
# pipeline.py — оркестратор
from loguru import logger
from .client import HttpClient
from .parsers import parse_product
from .models import Product
from .storage import Storage

class Pipeline:
    def __init__(self, client: HttpClient, storage: Storage):
        self.client = client
        self.storage = storage

    def process(self, url: str) -> None:
        html = self.client.get(url)          # ретраи + лимиты внутри
        if not html:
            return
        raw = parse_product(html)            # чистая функция
        try:
            product = Product(**raw)         # валидация
        except ValueError as e:
            logger.warning('Невалидно {}: {}', url, e)
            return
        self.storage.upsert(product.model_dump())

    def run(self, urls: list[str]) -> None:
        for url in urls:
            try:
                self.process(url)
            except Exception as e:
                logger.exception('Сбой на {}: {}', url, e)
```

**Принципы:** загрузка, парсинг, валидация и хранение разделены. Парсеры — чистые функции (вход HTML → выход dict), их легко тестировать без сети. Секреты — в переменных окружения, а не в коде.

---

## 🛠 Практические проекты

От простого к сложному. Все площадки ниже легальны для обучения.

| Уровень | Проект | Навыки | Площадка |
|:-------:|--------|--------|----------|
| 🟢 | Парсер книг | requests + BeautifulSoup, пагинация | `books.toscrape.com` |
| 🟢 | Цитаты + авторы | переход по ссылкам, связи | `quotes.toscrape.com` |
| 🟡 | Тестирование запросов | заголовки, статусы, cookies | `httpbin.org` |
| 🟡 | Мониторинг цен | расписание, дифф, уведомления | свой тестовый сайт |
| 🟡 | Парсер через скрытый API | DevTools, JSON, пагинация | открытые публичные API |
| 🔴 | Async-агрегатор новостей | aiohttp, дедуп, RSS | публичные RSS |
| 🔴 | SPA на Playwright | перехват ответов, скролл | демо-SPA |
| 🔴 | Scrapy + pipelines | очереди, валидация, экспорт | `books.toscrape.com` |

### Готовый мини-проект «Мониторинг цен» с уведомлением об изменении
```python
import json, os, requests
from bs4 import BeautifulSoup

STATE = 'prices.json'

def get_price(url: str) -> float:
    html = requests.get(url, timeout=10,
                        headers={'User-Agent': 'Mozilla/5.0'}).text
    soup = BeautifulSoup(html, 'lxml')
    raw = soup.select_one('.price').get_text(strip=True)
    return float(''.join(c for c in raw if c.isdigit() or c == '.'))

def load() -> dict:
    return json.load(open(STATE)) if os.path.exists(STATE) else {}

def check(url: str):
    old = load()
    new_price = get_price(url)
    if url in old and new_price < old[url]:
        print(f'📉 Цена упала: {old[url]} -> {new_price}')
    old[url] = new_price
    json.dump(old, open(STATE, 'w'))

check('https://books.toscrape.com/catalogue/a-light-in-the-attic_1000/index.html')
```

---

## ⚠️ Антипаттерны и частые ошибки

| ❌ Ошибка | ✅ Как правильно |
|----------|-----------------|
| Запрос без `timeout` | Всегда задавайте `(connect, read)` |
| `tag.text` без проверки на `None` | Хелперы `safe_text/safe_attr` |
| Парсинг HTML регулярками | BeautifulSoup / lxml / selectolax |
| Фиксированный `time.sleep(5)` для ожидания JS | `wait_for_selector` |
| Браузер там, где хватит HTTP | Сначала ищите скрытый API |
| Хранение всего в памяти | Потоковая запись (JSONL) + чекпоинты |
| Хардкод `User-Agent`/токенов | Конфиг + переменные окружения |
| Игнор `429`/`Retry-After` | Backoff и уважение лимитов |
| Нет дедупликации | `PRIMARY KEY` / `drop_duplicates` |
| Парсер без тестов | pytest на сохранённых фикстурах |

---

## 📖 Полезные ресурсы

**Документация:**
- [requests](https://requests.readthedocs.io/) · [httpx](https://www.python-httpx.org/)
- [BeautifulSoup](https://www.crummy.com/software/BeautifulSoup/bs4/doc/) · [lxml](https://lxml.de/) · [selectolax](https://github.com/rushter/selectolax)
- [Scrapy](https://docs.scrapy.org/) · [Playwright](https://playwright.dev/python/) · [aiohttp](https://docs.aiohttp.org/)
- [pydantic](https://docs.pydantic.dev/) · [tenacity](https://tenacity.readthedocs.io/) · [loguru](https://loguru.readthedocs.io/)

**Площадки для практики:**
- [books.toscrape.com](https://books.toscrape.com/) · [quotes.toscrape.com](https://quotes.toscrape.com/) · [httpbin.org](https://httpbin.org/)

**Инструменты:**
- DevTools → Network (поиск скрытых API)
- [regex101.com](https://regex101.com/) — отладка регулярок
- [curlconverter.com](https://curlconverter.com/) — cURL → Python
- [Insomnia](https://insomnia.rest/) / Postman — работа с API

---

## 🗺 Дорожная карта обучения

```
🟢 Новичок      →  Уроки 1–6    (requests, BeautifulSoup, CSS/XPath, regex)
🟡 Средний      →  Уроки 7–11   (API, динамика, async, Scrapy, обход защит)
🔴 Продвинутый  →  Уроки 12–14  (валидация, надёжность, тесты)
⚫ Профи         →  Уроки 15–16  (этика + production-архитектура)
```

---

## 🤝 Вклад в проект

PR и идеи приветствуются! Открывайте Issue с предложениями новых уроков, примеров или площадок для практики.

## 📄 Лицензия

Материалы распространяются под лицензией **MIT** — используйте свободно в учебных целях.

---

⭐ Если курс оказался полезным — поставьте звезду репозиторию!
