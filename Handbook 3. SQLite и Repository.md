## Что получится в конце занятия

В результате занятия будет создана следующая структура проекта:

```text
app/
│
├── database.py
├── main.py
│
├── models/
│   └── book.py
│
└── repositories/
    └── book_repository.py

test_repository.py
books.db
```

---
# Шаг 0. Создать новую ветку feat/database от main
Перед выполнением команд убедитесь что вы смёржили предыдущую ветку (feat/base-structure) с базовой структурой в main
```bash
git checkout main
git fetch
git pull
git checkout -b feat/database
```

# Шаг 1. Установить SQLAlchemy

Установить библиотеку:

```powershell
pip install sqlalchemy
```

Обновить зависимости проекта:

```powershell
pip freeze > requirements.txt
```

---

# Шаг 2. Создать файл `app/database.py`

Создать файл:

```text
app/database.py
```

Добавить следующий код:

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import DeclarativeBase, sessionmaker

DATABASE_URL = "sqlite:///books.db"

engine = create_engine(
    DATABASE_URL,
    connect_args={"check_same_thread": False},
)

SessionLocal = sessionmaker(
    autocommit=False,
    autoflush=False,
    bind=engine,
)


class Base(DeclarativeBase):
    pass


def get_db():
    db = SessionLocal()

    try:
        yield db
    finally:
        db.close()
```

---

# Шаг 3. Создать модель `Book`

Создать файл:

```text
app/models/book.py
```

Добавить модель книги:

```python
from sqlalchemy import String
from sqlalchemy.orm import Mapped, mapped_column

from app.database import Base


class Book(Base):
    __tablename__ = "books"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)

    title: Mapped[str] = mapped_column(String, nullable=False)

    author: Mapped[str] = mapped_column(String, nullable=False)
```

Для поля `id` не нужно добавлять `unique=True` или `nullable=False`, потому что `primary_key=True` уже делает поле уникальным и обязательным.

`unique=True` стоит добавлять только если по логике проекта значение не должно повторяться. Для книг обычно не стоит делать `title` уникальным, потому что разные книги могут иметь одинаковое название.

---

# Шаг 4. Создать таблицы

Открыть файл:

```text
app/main.py
```

Добавить импорт:

```python
from app.database import Base, engine
```

После создания приложения выполнить создание таблиц:

```python
Base.metadata.create_all(bind=engine)
```

Пример файла `app/main.py`:

```python
from fastapi import FastAPI

from app.database import Base, engine

app = FastAPI()

Base.metadata.create_all(bind=engine)


@app.get("/")
def root():
    return {"message": "Hello World"}
```

Запустить проект.

После запуска рядом с проектом должен появиться файл:

```text
books.db
```

---

# Шаг 5. Создать Repository

Создать файл:

```text
app/repositories/book_repository.py
```

Добавить начало класса:

```python
from sqlalchemy.orm import Session

from app.models.book import Book


class BookRepository:

    def __init__(self, db: Session):
        self.db = db
```

---

# Шаг 6. Добавить метод `create()`

Файл:

```text
app/repositories/book_repository.py
```

Добавить метод внутрь класса `BookRepository`:

```python
def create(self, book: Book) -> Book:
    self.db.add(book)
    self.db.commit()
    self.db.refresh(book)

    return book
```

---

# Шаг 7. Добавить метод `update()`

Файл:

```text
app/repositories/book_repository.py
```

Добавить метод внутрь класса `BookRepository`:

```python
def update(self, book: Book) -> Book:
    self.db.add(book)
    self.db.commit()
    self.db.refresh(book)

    return book
```

---

# Шаг 8. Альтернатива: убрать дублирование через `_upsert()`

Файл:

```text
app/repositories/book_repository.py
```

Методы `create()` и `update()` содержат одинаковую логику:

```python
self.db.add(book)
self.db.commit()
self.db.refresh(book)
```

Эту логику можно вынести в единый приватный метод `_upsert()`.

Добавить метод внутрь класса `BookRepository`:

```python
def _upsert(self, book: Book) -> Book:
    self.db.add(book)
    self.db.commit()
    self.db.refresh(book)

    return book
```

После этого обновить метод `create()`:

```python
def create(self, book: Book) -> Book:
    return self._upsert(book)
```

И обновить метод `update()`:

```python
def update(self, book: Book) -> Book:
    return self._upsert(book)
```

Итог: код сохранения объекта находится в одном месте, а `create()` и `update()` используют общий метод.

---

# Шаг 9. Добавить метод `get_all()`

Файл:

```text
app/repositories/book_repository.py
```

Добавить метод внутрь класса `BookRepository`:

```python
def get_all(self) -> list[Book]:
    return self.db.query(Book).all()
```

---

# Шаг 10. Добавить метод `get_by_id()`

Файл:

```text
app/repositories/book_repository.py
```

Добавить метод внутрь класса `BookRepository`:

```python
def get_by_id(
    self,
    book_id: int,
) -> Book | None:

    return (
        self.db.query(Book)
        .filter(Book.id == book_id)
        .first()
    )
```

---

# Шаг 11. Добавить метод `delete()`

Файл:

```text
app/repositories/book_repository.py
```

Добавить метод внутрь класса `BookRepository`:

```python
def delete(self, book: Book) -> None:
    self.db.delete(book)
    self.db.commit()
```

---

# Итоговый файл `app/repositories/book_repository.py`

К этому моменту файл должен выглядеть так, если использована альтернатива с `_upsert()` из шага 8:

```python
from sqlalchemy.orm import Session

from app.models.book import Book


class BookRepository:

    def __init__(self, db: Session):
        self.db = db

    def create(self, book: Book) -> Book:
        return self._upsert(book)

    def update(self, book: Book) -> Book:
        return self._upsert(book)

    def _upsert(self, book: Book) -> Book:
        self.db.add(book)
        self.db.commit()
        self.db.refresh(book)

        return book

    def get_all(self) -> list[Book]:
        return self.db.query(Book).all()

    def get_by_id(
        self,
        book_id: int,
    ) -> Book | None:

        return (
            self.db.query(Book)
            .filter(Book.id == book_id)
            .first()
        )

    def delete(self, book: Book) -> None:
        self.db.delete(book)
        self.db.commit()
```

---

# Шаг 12. Проверить работу Repository

Создать новый файл:

```text
test_repository.py
```

Добавить код:

```python
from app.database import Base, SessionLocal, engine
from app.models.book import Book
from app.repositories.book_repository import BookRepository

Base.metadata.create_all(bind=engine)

db = SessionLocal()

repository = BookRepository(db)

book = Book(
    title="Высоконагруженные приложения",
    author="Мартин Клеппман",
)

repository.create(book)

books = repository.get_all()

for book in books:
    print(book.id, book.title, book.author)

db.close()
```

Запустить файл:

```powershell
python test_repository.py
```

Если всё сделано правильно:

- создастся файл `books.db`;
- в базе появится таблица `books`;
- в таблицу добавится книга;
- книга выведется в консоль.

---

# Итог занятия

К концу занятия должно быть реализовано:

- [ ] Подключение SQLite
- [ ] Создан `database.py`
- [ ] Создана модель `Book`
- [ ] Создан файл `books.db`
- [ ] Создан `BookRepository`
- [ ] Реализованы методы:
  - `create()`
  - `update()`
  - `get_all()`
  - `get_by_id()`
  - `delete()`
- [ ] Показан альтернативный рефакторинг через `_upsert()`
- [ ] Repository успешно работает с настоящей SQLite базой данных
