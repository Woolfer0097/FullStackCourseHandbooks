# SQLite Studio
https://github.com/pawelsalawa/letos/releases
1. Подключиться к БД
2. Выбрать файл БД из вашего проекта
3. Изначально если вы нажмете на таблицу в вашей базе данных вы увидете структуру вашей таблицы
4. Нажав на Data вы можете увидеть содержимое вашей Базы Данных и менять значения в ней
5. Также вы можете выполнять запросы к Базе Данных на языке SQL

# Сбор и импорт данных

## Что нужно сделать

В этом занятии необходимо:

1. Получить данные с HTML-страницы.
2. Преобразовать полученные данные в JSON.
3. Прочитать JSON.
4. Сопоставить данные со схемой создания сущности.
5. Вызвать метод существующего сервиса.
6. Проверить результат вручную через SQL

```text
HTML -> BeautifulSoup -> JSON -> Pydantic-схема -> Service -> Repository -> SQLite
```

---

# Подготовка

Установите библиотеки:

```powershell
pip install requests beautifulsoup4
```

Обновите `requirements.txt`:

```powershell
pip freeze > requirements.txt
```

Создайте каталоги и файлы:

```text
project/
├── app/
├── frontend/
├── data/
│   └── books.json
└── scripts/
    ├── parse_books.py
    └── import_books.py
```

Каталог `scripts` предназначен для отдельных запускаемых скриптов.

---

# Парсинг из HTML в JSON

Будем получать книги со страницы:

```text
https://books.toscrape.com/catalogue/category/books_1/page-2.html
```

Для каждой книги получим:

- название;
- цену;
- рейтинг;
- наличие;
- ссылку на страницу книги;
- ссылку на изображение.

## HTML-элементы страницы

Карточка одной книги:

```html
<article class="product_pod">
    ...
</article>
```

Название и ссылка:

```html
<h3>
    <a href="..." title="Book title">...</a>
</h3>
```

Цена:

```html
<p class="price_color">£12.84</p>
```

Рейтинг:

```html
<p class="star-rating Three"></p>
```

Наличие:

```html
<p class="instock availability">
    In stock
</p>
```

---

## Скрипт парсинга

Создайте файл:

```text
scripts/parse_books.py
```

```python
import json
from pathlib import Path
from urllib.parse import urljoin

import requests
from bs4 import BeautifulSoup


PAGE_URL = (
    "https://books.toscrape.com/"
    "catalogue/category/books_1/page-2.html"
)

OUTPUT_FILE = Path("data/books.json")

RATING_MAP = {
    "One": 1,
    "Two": 2,
    "Three": 3,
    "Four": 4,
    "Five": 5,
}


def get_page_html(url: str) -> str:
    response = requests.get(
        url,
        timeout=10,
        headers={
            "User-Agent": "Mozilla/5.0",
        },
    )

    response.raise_for_status()

    return response.text


def parse_books(html: str) -> dict[str, dict]:
    soup = BeautifulSoup(html, "html.parser")

    books = {}

    book_cards = soup.select("article.product_pod")

    for card in book_cards:
        link_element = card.select_one("h3 a")
        price_element = card.select_one("p.price_color")
        rating_element = card.select_one("p.star-rating")
        availability_element = card.select_one(
            "p.instock.availability"
        )
        image_element = card.select_one("img.thumbnail")

        relative_book_url = link_element["href"]
        relative_image_url = image_element["src"]

        book_url = urljoin(PAGE_URL, relative_book_url)
        image_url = urljoin(PAGE_URL, relative_image_url)

        book_key = relative_book_url.split("/")[-2]

        rating_name = rating_element.get("class", [])[-1]

        price_text = price_element.get_text(strip=True)
        price = float(price_text.replace("£", ""))

        availability_text = availability_element.get_text(
            strip=True
        )

        books[book_key] = {
            "title": link_element["title"],
            "price": price,
            "rating": RATING_MAP.get(rating_name),
            "is_available": availability_text == "In stock",
            "source_url": book_url,
            "image_url": image_url,
        }

    return books


def save_to_json(
    data: dict[str, dict],
    file_path: Path,
) -> None:
    file_path.parent.mkdir(
        parents=True,
        exist_ok=True,
    )

    with file_path.open(
        "w",
        encoding="utf-8",
    ) as file:
        json.dump(
            data,
            file,
            ensure_ascii=False,
            indent=4,
        )


def main() -> None:
    html = get_page_html(PAGE_URL)
    books = parse_books(html)
    save_to_json(books, OUTPUT_FILE)

    print(f"Получено книг: {len(books)}")
    print(f"Данные сохранены в {OUTPUT_FILE}")


if __name__ == "__main__":
    main()
```

Запустите скрипт из корня проекта:

```powershell
python scripts/parse_books.py
```

После запуска должен появиться файл:

```text
data/books.json
```

---

## Результат парсинга

Пример содержимого `books.json`:

```json
{
    "in-her-wake_980": {
        "title": "In Her Wake",
        "price": 12.84,
        "rating": 1,
        "is_available": true,
        "source_url": "https://books.toscrape.com/catalogue/in-her-wake_980/index.html",
        "image_url": "https://books.toscrape.com/media/cache/example.jpg"
    },
    "how-music-works_979": {
        "title": "How Music Works",
        "price": 37.32,
        "rating": 2,
        "is_available": true,
        "source_url": "https://books.toscrape.com/catalogue/how-music-works_979/index.html",
        "image_url": "https://books.toscrape.com/media/cache/example.jpg"
    }
}
```

JSON представляет собой словарь:

```python
{
    "ключ": {
        "поле": "значение"
    }
}
```

Ключом является идентификатор книги, полученный из ссылки.

---

# Импорт из JSON в базу данных

## Алгоритм импорта

1. Открыть JSON-файл.
    
2. Преобразовать JSON в Python-словарь.
    
3. Получить пары `key`, `value` через метод `.items()`.
    
4. Сопоставить данные со схемой создания сущности.
    
5. Создать объект Pydantic-схемы.
    
6. Передать объект в метод существующего сервиса.
    
7. Сохранить изменения в базе данных.
    

```python
with open("data/books.json", "r", encoding="utf-8") as file:
    data = json.load(file)

for key, value in data.items():
    schema = BookCreateRequest(
        external_id=key,
        title=value["title"],
    )

    service.create(schema)
```

Не передавайте словарь непосредственно в repository. Импорт должен использовать ту же бизнес-логику, что и handlers:

```text
Import script -> Service -> Repository -> Database
```

---

## Скрипт импорта

Создайте файл:

```text
scripts/import_books.py
```

Названия импортов необходимо заменить на названия файлов и классов своего проекта.

```python
# асинхронность не используется
import json
from pathlib import Path

from app.database import session_maker
from app.repositories.books import BookRepository
from app.schemas.books import BookCreateRequest
from app.services.books import BookService


JSON_FILE = Path("data/books.json")


def read_json(file_path: Path) -> dict[str, dict]:
    with file_path.open(
        "r",
        encoding="utf-8",
    ) as file:
        return json.load(file)


def import_books() -> None:
    books_data = read_json(JSON_FILE)

    created_count = 0
    error_count = 0

    with session_maker() as session:
        repository = BookRepository(session=session)
        service = BookService(repository=repository)

        for external_id, book_data in books_data.items():
            try:
                book_create = BookCreateRequest(
                    external_id=external_id,
                    title=book_data["title"],
                    price=book_data["price"],
                    rating=book_data["rating"],
                    is_available=book_data["is_available"],
                    source_url=book_data["source_url"],
                    image_url=book_data["image_url"],
                )

                service.create(book_create)

                created_count += 1

            except Exception as error:
                error_count += 1

                print(
                    f"Не удалось импортировать "
                    f"{external_id}: {error}"
                )

        session.commit()

    print(f"Добавлено записей: {created_count}")
    print(f"Ошибок: {error_count}")


if __name__ == "__main__":
    import_books()
```

Запустите скрипт из корня проекта:

```powershell
python scripts/import_books.py
```

---

## Сопоставление JSON со схемой

JSON может содержать больше или меньше полей, чем схема backend.

Например, JSON:

```json
{
    "title": "In Her Wake",
    "price": 12.84,
    "rating": 1,
    "is_available": true
}
```

Схема backend:

```python
class BookCreateRequest(BaseModel):
    title: str = Field(min_length=1, max_length=255)
    description: str | None = Field(default=None, max_length=1000)
    price: float = Field(gt=0)
    rating: int = Field(ge=1, le=5)
    is_available: bool
```

Сопоставление:

```python
book_create = BookCreateRequest(
    title=book_data["title"],
    description=None,
    price=book_data["price"],
    rating=book_data["rating"],
    is_available=book_data["is_available"],
)
```

Названия полей JSON не обязаны совпадать с названиями полей схемы:

```python
book_create = BookCreateRequest(
    name=book_data["title"],
    cost=book_data["price"],
    available=book_data["is_available"],
)
```

Главное — передать в схему backend правильные значения.

---

## Проверка обязательных полей

Если некоторые данные могут отсутствовать, используйте `.get()`:

```python
book_create = BookCreateRequest(
    title=book_data["title"],
    description=book_data.get("description"),
    image_url=book_data.get("image_url"),
)
```

Разница:

```python
book_data["title"]
```

Если поля нет, возникнет `KeyError`.

```python
book_data.get("title")
```

Если поля нет, вернётся `None`.

Для обязательных полей лучше использовать квадратные скобки:

```python
title=book_data["title"]
```

Для необязательных полей можно использовать `.get()`:

```python
description=book_data.get("description")
```

Использование .get() может усложнить отладку, но зато может скрыть внутренние ошибки сервиса от глаз пользователя

---

## Повторный импорт

При повторном запуске скрипта могут появиться дубликаты.

Перед созданием записи можно проверить, существует ли она:

```python
existing_book = await service.get_by_external_id(
    external_id
)

if existing_book is not None:
    print(f"Книга уже существует: {external_id}")
    continue
```

Полный фрагмент:

```python
for external_id, book_data in books_data.items():
    existing_book = await service.get_by_external_id(
        external_id
    )

    if existing_book is not None:
        continue

    book_create = BookCreateRequest(
        external_id=external_id,
        title=book_data["title"],
        price=book_data["price"],
    )

    service.create(book_create)
```

Используйте этот вариант только в том случае, если соответствующий метод уже реализован в сервисе.

---

# Работа с базой данных вручную

Ручные SQL-запросы можно использовать для:

- проверки результата импорта;
- поиска ошибок;
- просмотра таблиц;
- временного изменения тестовых данных;
- удаления ошибочно импортированных записей.

---

# SELECT — получение данных

Получить все записи:

```sql
SELECT * FROM books;
```

Получить отдельные столбцы:

```sql
SELECT id, title, price FROM books;
```

Найти запись по идентификатору:

```sql
SELECT * FROM books WHERE id = 1;
```

Найти книги дороже 30:

```sql
SELECT id, title, price FROM books WHERE price > 30;
```

Найти доступные книги:

```sql
SELECT * FROM books WHERE is_available = 1;
```

В SQLite логические значения обычно представлены числами:

```text
0 — False
1 — True
```

Сортировка по цене:

```sql
SELECT id, title, price FROM books ORDER BY price DESC;
```

Ограничение количества записей:

```sql
SELECT id, title FROM books LIMIT 10;
```

Подсчитать количество записей:

```sql
SELECT COUNT(*) FROM books;
```

Подсчитать количество импортированных книг:

```sql
SELECT COUNT(*) AS books_count FROM books;
```

---

# INSERT — добавление записи

Добавить одну запись:

```sql
INSERT INTO books (title, price, rating, is_available) VALUES ('Test Book', 15.50, 5, 1);
```

Проверить добавленную запись:

```sql
SELECT * FROM books WHERE title = 'Test Book';
```

Поля с автоматическим `id` указывать не нужно.

---

# UPDATE — изменение записи

Изменить цену книги:

```sql
UPDATE books SET price = 20.00 WHERE id = 1;
```

Изменить несколько полей:

```sql
UPDATE books SET price = 25.00, rating = 4, is_available = 1 WHERE id = 1;
```

Проверить результат:

```sql
SELECT * FROM books WHERE id = 1;
```

Перед выполнением `UPDATE` сначала выполните такой же `SELECT`:

```sql
SELECT * FROM books WHERE id = 1;
```

Не запускайте `UPDATE` без `WHERE`:

```sql
UPDATE books SET price = 0;
```

Такой запрос изменит все записи таблицы.

---

# DELETE — удаление записи

Удалить одну запись:

```sql
DELETE FROM books WHERE id = 1;
```

Удалить тестовую запись:

```sql
DELETE FROM books WHERE title = 'Test Book';
```

Удалить все записи с нулевой ценой:

```sql
DELETE FROM books WHERE price = 0;
```

Перед выполнением `DELETE` сначала проверьте записи:

```sql
SELECT * FROM books WHERE price = 0;
```

Не запускайте `DELETE` без `WHERE`:

```sql
DELETE FROM books;
```

Такой запрос удалит все записи таблицы
