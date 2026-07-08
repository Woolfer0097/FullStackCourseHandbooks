## Что получится в конце занятия

В результате занятия API будет работать с настоящей SQLite базой данных через слои:

```text
HTTP Request
    ↓
Handler
    ↓
Service
    ↓
Repository
    ↓
SQLite
```

Структура проекта:

```text
app/
│
├── database.py
├── main.py
│
├── handlers/
│   └── books.py
│
├── models/
│   └── book.py
│
├── repositories/
│   └── book_repository.py
│
├── schemas/
│   └── book.py
│
└── services/
    └── book_service.py
```

---

# Шаг 1. Создать папки для новых слоёв

Создать папки:

```text
app/handlers
app/schemas
app/services
```

---

# Шаг 2. Создать Pydantic-схемы

Создать файл:

```text
app/schemas/book.py
```

Добавить код:

```python
from pydantic import BaseModel, ConfigDict, Field


class BookCreate(BaseModel):
    title: str = Field(min_length=1, max_length=200)
    author: str = Field(min_length=1, max_length=200)


class BookUpdate(BaseModel):
    title: str | None = Field(default=None, min_length=1, max_length=200)
    author: str | None = Field(default=None, min_length=1, max_length=200)


class BookResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    title: str
    author: str
```

---

# Шаг 3. Создать Service

Создать файл:

```text
app/services/book_service.py
```

Добавить начало файла:

```python
from fastapi import HTTPException, status
from sqlalchemy.orm import Session

from app.models.book import Book
from app.repositories.book_repository import BookRepository
from app.schemas.book import BookCreate, BookUpdate


class BookService:

    def __init__(self, db: Session):
        self.repository = BookRepository(db)
```

---

# Шаг 4. Добавить создание книги

Файл:

```text
app/services/book_service.py
```

Добавить метод внутрь класса `BookService`:

```python
def create_book(self, schema: BookCreate) -> Book:
    book = Book(
        title=schema.title,
        author=schema.author,
    )

    return self.repository.create(book)
```

---

# Шаг 5. Добавить получение списка книг

Файл:

```text
app/services/book_service.py
```

Добавить метод внутрь класса `BookService`:

```python
def get_books(self) -> list[Book]:
    return self.repository.get_all()
```

---

# Шаг 6. Добавить получение одной книги

Файл:

```text
app/services/book_service.py
```

Добавить метод внутрь класса `BookService`:

```python
def get_book(self, book_id: int) -> Book:
    book = self.repository.get_by_id(book_id)

    if book is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Book not found",
        )

    return book
```

---

# Шаг 7. Добавить обновление книги

Файл:

```text
app/services/book_service.py
```

Добавить метод внутрь класса `BookService`:

```python
def update_book(
    self,
    book_id: int,
    schema: BookUpdate,
) -> Book:

    book = self.get_book(book_id)

    if schema.title is None and schema.author is None:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="At least one field must be provided",
        )

    if schema.title is not None:
        book.title = schema.title

    if schema.author is not None:
        book.author = schema.author

    return self.repository.update(book)
```

---

# Шаг 8. Добавить удаление книги

Файл:

```text
app/services/book_service.py
```

Добавить метод внутрь класса `BookService`:

```python
def delete_book(self, book_id: int) -> None:
    book = self.get_book(book_id)

    self.repository.delete(book)
```

---

# Итоговый файл `app/services/book_service.py`

К этому моменту файл должен выглядеть так:

```python
from fastapi import HTTPException, status
from sqlalchemy.orm import Session

from app.models.book import Book
from app.repositories.book_repository import BookRepository
from app.schemas.book import BookCreate, BookUpdate


class BookService:

    def __init__(self, db: Session):
        self.repository = BookRepository(db)

    def create_book(self, schema: BookCreate) -> Book:
        book = Book(
            title=schema.title,
            author=schema.author,
        )

        return self.repository.create(book)

    def get_books(self) -> list[Book]:
        return self.repository.get_all()

    def get_book(self, book_id: int) -> Book:
        book = self.repository.get_by_id(book_id)

        if book is None:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail="Book not found",
            )

        return book

    def update_book(
        self,
        book_id: int,
        schema: BookUpdate,
    ) -> Book:

        book = self.get_book(book_id)

        if schema.title is None and schema.author is None:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail="At least one field must be provided",
            )

        if schema.title is not None:
            book.title = schema.title

        if schema.author is not None:
            book.author = schema.author

        return self.repository.update(book)

    def delete_book(self, book_id: int) -> None:
        book = self.get_book(book_id)

        self.repository.delete(book)
```

---

# Шаг 9. Создать Handler

Создать файл:

```text
app/handlers/books.py
```

Добавить начало файла:

```python
from fastapi import APIRouter, Depends, status
from sqlalchemy.orm import Session

from app.database import get_db
from app.schemas.book import BookCreate, BookResponse, BookUpdate
from app.services.book_service import BookService

router = APIRouter(
    prefix="/books",
    tags=["books"],
)

def get_book_service(
    db: Session = Depends(get_db),
) -> BookService:
    return BookService(db)
```

---

# Шаг 10. Добавить endpoint создания книги

Файл:

```text
app/handlers/books.py
```

Добавить endpoint:

```python
@router.post(
    "/",
    response_model=BookResponse,
    status_code=status.HTTP_201_CREATED,
)
def create_book(
    schema: BookCreate,
    service: BookService = Depends(get_book_service),
):
    return service.create_book(schema)
```

---

# Шаг 11. Добавить endpoint получения всех книг

Файл:

```text
app/handlers/books.py
```

Добавить endpoint:

```python
@router.get(
    "/",
    response_model=list[BookResponse],
)
def get_books(
    service: BookService = Depends(get_book_service),
):
    return service.get_books()
```

---

# Шаг 12. Добавить endpoint получения книги по id

Файл:

```text
app/handlers/books.py
```

Добавить endpoint:

```python
@router.get(
    "/{book_id}",
    response_model=BookResponse,
)
def get_book(
    book_id: int,
    service: BookService = Depends(get_book_service),
):
    return service.get_book(book_id)
```

---

# Шаг 13. Добавить endpoint обновления книги

Файл:

```text
app/handlers/books.py
```

Добавить endpoint:

```python
@router.patch(
    "/{book_id}",
    response_model=BookResponse,
)
def update_book(
    book_id: int,
    schema: BookUpdate,
    service: BookService = Depends(get_book_service),
):
    return service.update_book(book_id, schema)
```

---

# Шаг 14. Добавить endpoint удаления книги

Файл:

```text
app/handlers/books.py
```

Добавить endpoint:

```python
@router.delete(
    "/{book_id}",
    status_code=status.HTTP_204_NO_CONTENT,
)
def delete_book(
    book_id: int,
    service: BookService = Depends(get_book_service),
) -> None:
    service.delete_book(book_id)
```

---

# Итоговый файл `app/handlers/books.py`

К этому моменту файл должен выглядеть так:

```python
from fastapi import APIRouter, Depends, status
from sqlalchemy.orm import Session

from app.database import get_db
from app.schemas.book import BookCreate, BookResponse, BookUpdate
from app.services.book_service import BookService

router = APIRouter(
    prefix="/books",
    tags=["books"],
)


def get_book_service(
    db: Session = Depends(get_db),
) -> BookService:
    return BookService(db)


@router.post(
    "/",
    response_model=BookResponse,
    status_code=status.HTTP_201_CREATED,
)
def create_book(
    schema: BookCreate,
    service: BookService = Depends(get_book_service),
):
    return service.create_book(schema)


@router.get(
    "/",
    response_model=list[BookResponse],
)
def get_books(
    service: BookService = Depends(get_book_service),
):
    return service.get_books()


@router.get(
    "/{book_id}",
    response_model=BookResponse,
)
def get_book(
    book_id: int,
    service: BookService = Depends(get_book_service),
):
    return service.get_book(book_id)


@router.patch(
    "/{book_id}",
    response_model=BookResponse,
)
def update_book(
    book_id: int,
    schema: BookUpdate,
    service: BookService = Depends(get_book_service),
):
    return service.update_book(book_id, schema)


@router.delete(
    "/{book_id}",
    status_code=status.HTTP_204_NO_CONTENT,
)
def delete_book(
    book_id: int,
    service: BookService = Depends(get_book_service),
) -> None:
    service.delete_book(book_id)
```

---

# Шаг 15. Подключить router в `main.py`

Открыть файл:

```text
app/main.py
```

Привести файл к следующему виду:

```python
from fastapi import FastAPI

from app.database import Base, engine
from app.handlers.books import router as books_router
from app.models.book import Book

app = FastAPI()

Base.metadata.create_all(bind=engine)

app.include_router(books_router)


@app.get("/")
def root():
    return {"message": "Hello World"}
```

---

# Шаг 16. Запустить приложение

Запустить проект:

```powershell
fastapi dev
```

Не забудь активировать виртуальное окружение
Linux/MacOS
```
source ./.venv/bin/activate
```
Windows
```
.\.venv\Scripts\Activate.ps1
```

Открыть документацию:

```text
http://127.0.0.1:8000/docs
```

---

# Шаг 17. Проверить создание книги

В Swagger открыть:

```text
POST /books/
```

Тело запроса:

```json
{
  "title": "1984",
  "author": "George Orwell"
}
```

Ожидаемый результат:

```json
{
  "id": 1,
  "title": "1984",
  "author": "George Orwell"
}
```

---

# Шаг 18. Проверить получение всех книг

В Swagger открыть:

```text
GET /books/
```

Ожидаемый результат:

```json
[
  {
    "id": 1,
    "title": "1984",
    "author": "George Orwell"
  }
]
```

---

# Шаг 19. Проверить получение книги по id

В Swagger открыть:

```text
GET /books/1
```

Ожидаемый результат:

```json
{
  "id": 1,
  "title": "1984",
  "author": "George Orwell"
}
```

Если книги нет, API вернёт:

```json
{
  "detail": "Book not found"
}
```

---

# Шаг 20. Проверить обновление книги

В Swagger открыть:

```text
PATCH /books/1
```

Тело запроса:

```json
{
  "title": "Animal Farm"
}
```

Ожидаемый результат:

```json
{
  "id": 1,
  "title": "Animal Farm",
  "author": "George Orwell"
}
```

---

# Шаг 21. Проверить удаление книги

В Swagger открыть:

```text
DELETE /books/1
```

Ожидаемый результат:

```text
204 No Content
```

После удаления запрос:

```text
GET /books/1
```

должен вернуть:

```json
{
  "detail": "Book not found"
}
```

---

# Шаг 22. Проверить валидацию

Проверить запрос с пустым названием:

```json
{
  "title": "",
  "author": "George Orwell"
}
```

API должен вернуть ошибку валидации.

Проверить пустое обновление:

```json
{}
```

API должен вернуть:

```json
{
  "detail": "At least one field must be provided"
}
```

---

# Итог занятия

К концу занятия должно быть реализовано:

- Созданы Pydantic-схемы:
  - `BookCreate`
  - `BookUpdate`
  - `BookResponse`
- Создан `BookService`
- Создан `books` handler
- Repository подключён к API через Service
- Реализованы endpoint'ы:
  - `POST /books/`
  - `GET /books/`
  - `GET /books/{book_id}`
  - `PATCH /books/{book_id}`
  - `DELETE /books/{book_id}`
- Добавлена валидация входных данных
- Добавлена обработка ошибок:
  - `404 Book not found`
  - `400 At least one field must be provided`
- API работает с настоящей SQLite базой данных
