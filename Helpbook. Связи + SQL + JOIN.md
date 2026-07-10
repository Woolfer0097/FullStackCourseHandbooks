# Связи моделей и работа с Repository

## Архитектура проекта

Во всём проекте используется одна цепочка вызовов:

```
HTTP Request
      ↓
Handlers (FastAPI)
      ↓
Service (бизнес логика + связь с репозиторием)
      ↓
Repository (работа с Базой Данных)
      ↓
Database
```

### Ответственность каждого слоя

**Handlers**

- получают HTTP-запрос;
- валидируют входные данные через Pydantic;
- вызывают сервис;
- возвращают ответ

**Service**

- содержит бизнес-логику;
- может обращаться к нескольким repository;
- не знает SQL

**Repository**

- работает только с базой данных;
- выполняет SELECT / INSERT / UPDATE / DELETE;
- ничего не знает про HTTP

---

# Связь один к одному (One-to-One)

Например:

```
User
----
id
name

Profile
-------
id
bio
user_id
```

Один пользователь имеет один профиль.

## Модели

```python
class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)

    profile: Mapped["Profile"] = relationship(
        back_populates="user",
        uselist=False,
    )
```

```python
class Profile(Base):
    __tablename__ = "profiles"

    id: Mapped[int] = mapped_column(primary_key=True)

    user_id: Mapped[int] = mapped_column(
        ForeignKey("users.id"),
        unique=True,
    )

    user: Mapped["User"] = relationship(
        back_populates="profile",
    )
```

## Главное

Для связи 1:1 необходимо:

- `ForeignKey`    
- `unique=True`
- `uselist=False` на стороне родителя

---

# Связь один ко многим (One-to-Many)

Например:

```
Author
-------
id
name

Book
-----
id
title
author_id
```

Один автор имеет много книг.

---

## Модели

```python
class Author(Base):
    __tablename__ = "authors"

    id: Mapped[int] = mapped_column(primary_key=True)

    books: Mapped[list["Book"]] = relationship(
        back_populates="author",
    )
```

```python
class Book(Base):
    __tablename__ = "books"

    id: Mapped[int] = mapped_column(primary_key=True)

    author_id: Mapped[int] = mapped_column(
        ForeignKey("authors.id"),
    )

    author: Mapped["Author"] = relationship(
        back_populates="books",
    )
```

## Главное

- ForeignKey хранится на стороне "многие"
- Родитель содержит список объектов.
- Ребёнок содержит один объект родителя

---

# Связь многие ко многим (Many-to-Many)

Например:

```
Book

Author
```

Одна книга может иметь нескольких авторов.

Один автор может написать несколько книг.

---

## Промежуточная таблица

```python
book_author = Table(
    "book_author",
    Base.metadata,
    Column("book_id", ForeignKey("books.id"), primary_key=True),
    Column("author_id", ForeignKey("authors.id"), primary_key=True),
)
```

---

## Модели

```python
class Book(Base):
    __tablename__ = "books"

    id: Mapped[int] = mapped_column(primary_key=True)

    authors: Mapped[list["Author"]] = relationship(
        secondary=book_author,
        back_populates="books",
    )
```

```python
class Author(Base):
    __tablename__ = "authors"

    id: Mapped[int] = mapped_column(primary_key=True)

    books: Mapped[list["Book"]] = relationship(
        secondary=book_author,
        back_populates="authors",
    )
```

---

# JOIN в Repository

Repository отвечает за получение данных из базы.

---

## INNER JOIN

Получить книги вместе с авторами.

```python
stmt = (
    select(Book)
    .join(Book.author)
)
```

Получатся только книги, у которых есть автор.

---

## LEFT JOIN

Получить все книги, даже если автор отсутствует.

```python
stmt = (
    select(Book)
    .outerjoin(Book.author)
)
```

---

# Когда использовать JOIN

Используйте `join()`, если:

- нужно фильтровать по связанной таблице;
    
- нужно сортировать по связанной таблице;
    
- нужно объединить данные нескольких таблиц в одном запросе.
    

Например:

```python
stmt = (
    select(Book)
    .join(Book.author)
    .where(Author.name == "Толстой")
)
```

---

# Когда JOIN не нужен

Если требуется просто получить объект вместе с его связями, лучше использовать загрузку отношений.

Например:

```python
stmt = (
    select(Book)
    .options(selectinload(Book.author))
)
```

или

```python
stmt = (
    select(Book)
    .options(selectinload(Book.authors))
)
```

Это избавляет от проблемы N+1 запросов и делает код чище.

---

# JOIN и отношения — разные вещи

Очень частая ошибка начинающих — считать, что `relationship()` автоматически делает JOIN.

Это не так.

`relationship()` лишь описывает связь между моделями.

Например:

```python
book.author
```

SQLAlchemy знает, как получить автора, но SQL-запрос будет сформирован только в момент обращения к полю (или заранее, если используется `selectinload()` или `joinedload()`).

---

# JOIN при фильтрации

Фильтровать по связанной таблице без JOIN нельзя.

Правильно:

```python
stmt = (
    select(Book)
    .join(Book.author)
    .where(Author.name == "Пушкин")
)
```

Неправильно:

```python
select(Book).where(Author.name == "Пушкин")
```

---

# JOIN при Many-to-Many

SQLAlchemy сам использует промежуточную таблицу.

Достаточно написать:

```python
stmt = (
    select(Book)
    .join(Book.authors)
)
```

Будет выполнено:

```
books
    ↓
book_author
    ↓
authors
```

Вручную соединять промежуточную таблицу обычно не требуется.

---

# Несколько JOIN

Можно объединять несколько таблиц.

```python
stmt = (
    select(Book)
    .join(Book.author)
    .join(Book.publisher)
)
```

Главное — чтобы между моделями были описаны `relationship()`.

---

# Частые ошибки

### 1. Отсутствует `ForeignKey`

Без внешнего ключа SQLAlchemy не сможет определить связь.

---

### 2. Нет `back_populates`

Навигация будет работать только в одну сторону или не будет работать корректно.

---

### 3. Для One-to-One забыли `unique=True`

Тогда база данных позволит создать несколько записей, хотя связь должна быть один к одному.

---

### 4. Для One-to-One забыли `uselist=False`

Вместо одного объекта будет возвращаться список.

---

### 5. Использование JOIN вместо загрузки отношений

Если фильтрация или сортировка не нужны, чаще всего лучше использовать:

```python
selectinload(...)
```

или

```python
joinedload(...)
```

а не писать `join()` вручную.

---

# Краткая памятка

|Связь|Где находится ForeignKey|relationship|
|---|---|---|
|1 → 1|у зависимой модели (`unique=True`)|`uselist=False`|
|1 → N|у стороны "многие"|список ↔ объект|
|N → N|в промежуточной таблице|`secondary=`|

## Когда использовать JOIN

|Задача|Решение|
|---|---|
|Фильтрация по связанной таблице|`join()`|
|Сортировка по связанной таблице|`join()`|
|Просто получить связанные объекты|`selectinload()` / `joinedload()`|
|Many-to-Many|`join(Model.relationship)`|

Главное правило: **Repository работает только с SQLAlchemy и базой данных. Вся бизнес-логика остаётся в Service, а Handlers только принимают запросы и возвращают ответы.**


---
---
---


# Примеры Repository для связей моделей

Все примеры используют синхронную SQLAlchemy-сессию:

```python
from sqlalchemy.orm import Session
```

Каждый раздел является независимым примером.

---

# Один к одному: User и Profile

Предполагается, что в моделях определены отношения:

```python
User.profile
Profile.user
```

## `app/repositories/user_repository.py`

```python
from sqlalchemy.orm import (
    Session,
    contains_eager,
    joinedload,
)

from app.models.profile import Profile
from app.models.user import User


class UserRepository:

    def __init__(self, db: Session):
        self.db = db

    def create(self, user: User) -> User:
        self.db.add(user)
        self.db.commit()
        self.db.refresh(user)

        return user

    def get_by_id(
        self,
        user_id: int,
    ) -> User | None:
        return (
            self.db.query(User)
            .filter(User.id == user_id)
            .first()
        )

    def get_all(self) -> list[User]:
        return self.db.query(User).all()

    def get_with_profile(
        self,
        user_id: int,
    ) -> User | None:
        return (
            self.db.query(User)
            .options(joinedload(User.profile))
            .filter(User.id == user_id)
            .first()
        )

    def get_all_with_profiles(
        self,
    ) -> list[User]:
        return (
            self.db.query(User)
            .options(joinedload(User.profile))
            .all()
        )

    def get_by_profile_city(
        self,
        city: str,
    ) -> list[User]:
        return (
            self.db.query(User)
            .join(User.profile)
            .options(contains_eager(User.profile))
            .filter(Profile.city == city)
            .all()
        )

    def get_all_with_optional_profile(
        self,
    ) -> list[User]:
        return (
            self.db.query(User)
            .outerjoin(User.profile)
            .options(contains_eager(User.profile))
            .all()
        )

    def update(self, user: User) -> User:
        self.db.add(user)
        self.db.commit()
        self.db.refresh(user)

        return user

    def delete(self, user: User) -> None:
        self.db.delete(user)
        self.db.commit()
```

## `app/repositories/profile_repository.py`

```python
from sqlalchemy.orm import (
    Session,
    joinedload,
)

from app.models.profile import Profile


class ProfileRepository:

    def __init__(self, db: Session):
        self.db = db

    def create(self, profile: Profile) -> Profile:
        self.db.add(profile)
        self.db.commit()
        self.db.refresh(profile)

        return profile

    def get_by_id(
        self,
        profile_id: int,
    ) -> Profile | None:
        return (
            self.db.query(Profile)
            .filter(Profile.id == profile_id)
            .first()
        )

    def get_by_user_id(
        self,
        user_id: int,
    ) -> Profile | None:
        return (
            self.db.query(Profile)
            .filter(Profile.user_id == user_id)
            .first()
        )

    def get_with_user(
        self,
        profile_id: int,
    ) -> Profile | None:
        return (
            self.db.query(Profile)
            .options(joinedload(Profile.user))
            .filter(Profile.id == profile_id)
            .first()
        )

    def update(self, profile: Profile) -> Profile:
        self.db.add(profile)
        self.db.commit()
        self.db.refresh(profile)

        return profile

    def delete(self, profile: Profile) -> None:
        self.db.delete(profile)
        self.db.commit()
```

## Особенности JOIN для связи один к одному

Получить только пользователей с профилем:

```python
self.db.query(User).join(User.profile).all()
```

Получить всех пользователей, включая пользователей без профиля:

```python
self.db.query(User).outerjoin(User.profile).all()
```

Получить пользователей по полю профиля:

```python
return (
    self.db.query(User)
    .join(User.profile)
    .filter(Profile.city == city)
    .all()
)
```

---

# Один ко многим: Author и Book

## `app/repositories/author_repository.py`

```python
from sqlalchemy.orm import (
    Session,
    selectinload,
)

from app.models.author import Author
from app.models.book import Book


class AuthorRepository:

    def __init__(self, db: Session):
        self.db = db

    def create(self, author: Author) -> Author:
        self.db.add(author)
        self.db.commit()
        self.db.refresh(author)

        return author

    def get_by_id(
        self,
        author_id: int,
    ) -> Author | None:
        return (
            self.db.query(Author)
            .filter(Author.id == author_id)
            .first()
        )

    def get_all(self) -> list[Author]:
        return self.db.query(Author).all()

    def get_with_books(
        self,
        author_id: int,
    ) -> Author | None:
        return (
            self.db.query(Author)
            .options(selectinload(Author.books))
            .filter(Author.id == author_id)
            .first()
        )

    def get_all_with_books(
        self,
    ) -> list[Author]:
        return (
            self.db.query(Author)
            .options(selectinload(Author.books))
            .all()
        )

    def get_by_book_title(
        self,
        title: str,
    ) -> list[Author]:
        return (
            self.db.query(Author)
            .join(Author.books)
            .filter(Book.title == title)
            .options(selectinload(Author.books))
            .distinct()
            .all()
        )

    def update(self, author: Author) -> Author:
        self.db.add(author)
        self.db.commit()
        self.db.refresh(author)

        return author

    def delete(self, author: Author) -> None:
        self.db.delete(author)
        self.db.commit()
```

## `app/repositories/book_repository.py`

```python
from sqlalchemy.orm import (
    Session,
    contains_eager,
    joinedload,
)

from app.models.author import Author
from app.models.book import Book


class BookRepository:

    def __init__(self, db: Session):
        self.db = db

    def create(self, book: Book) -> Book:
        self.db.add(book)
        self.db.commit()
        self.db.refresh(book)

        return book

    def get_by_id(
        self,
        book_id: int,
    ) -> Book | None:
        return (
            self.db.query(Book)
            .filter(Book.id == book_id)
            .first()
        )

    def get_all(self) -> list[Book]:
        return self.db.query(Book).all()

    def get_with_author(
        self,
        book_id: int,
    ) -> Book | None:
        return (
            self.db.query(Book)
            .options(joinedload(Book.author))
            .filter(Book.id == book_id)
            .first()
        )

    def get_all_with_authors(
        self,
    ) -> list[Book]:
        return (
            self.db.query(Book)
            .options(joinedload(Book.author))
            .all()
        )

    def get_by_author_id(
        self,
        author_id: int,
    ) -> list[Book]:
        return (
            self.db.query(Book)
            .filter(Book.author_id == author_id)
            .all()
        )

    def get_by_author_name(
        self,
        author_name: str,
    ) -> list[Book]:
        return (
            self.db.query(Book)
            .join(Book.author)
            .options(contains_eager(Book.author))
            .filter(Author.name == author_name)
            .all()
        )

    def get_all_ordered_by_author_name(
        self,
    ) -> list[Book]:
        return (
            self.db.query(Book)
            .join(Book.author)
            .options(contains_eager(Book.author))
            .order_by(Author.name)
            .all()
        )

    def update(self, book: Book) -> Book:
        self.db.add(book)
        self.db.commit()
        self.db.refresh(book)

        return book

    def delete(self, book: Book) -> None:
        self.db.delete(book)
        self.db.commit()
```

## Особенности JOIN для связи один ко многим

Если нужно получить книги конкретного автора по `author_id`, JOIN не требуется:

```python
return (
    self.db.query(Book)
    .filter(Book.author_id == author_id)
    .all()
)
```

Если нужно фильтровать по имени автора, JOIN необходим:

```python
return (
    self.db.query(Book)
    .join(Book.author)
    .filter(Author.name == author_name)
    .all()
)
```

Если нужно найти автора по книге и загрузить все его книги:

```python
return (
    self.db.query(Author)
    .join(Author.books)
    .filter(Book.title == title)
    .options(selectinload(Author.books))
    .distinct()
    .all()
)
```

Здесь `join()` используется для поиска автора, а `selectinload()` загружает полный список его книг.

---

# Многие ко многим: Book и Author

## `app/repositories/book_repository.py`

```python
from sqlalchemy.orm import (
    Session,
    selectinload,
)

from app.models.author import Author
from app.models.book import Book


class BookRepository:

    def __init__(self, db: Session):
        self.db = db

    def create(self, book: Book) -> Book:
        self.db.add(book)
        self.db.commit()
        self.db.refresh(book)

        return book

    def get_by_id(
        self,
        book_id: int,
    ) -> Book | None:
        return (
            self.db.query(Book)
            .filter(Book.id == book_id)
            .first()
        )

    def get_all(self) -> list[Book]:
        return self.db.query(Book).all()

    def get_with_authors(
        self,
        book_id: int,
    ) -> Book | None:
        return (
            self.db.query(Book)
            .options(selectinload(Book.authors))
            .filter(Book.id == book_id)
            .first()
        )

    def get_all_with_authors(
        self,
    ) -> list[Book]:
        return (
            self.db.query(Book)
            .options(selectinload(Book.authors))
            .all()
        )

    def get_by_author_id(
        self,
        author_id: int,
    ) -> list[Book]:
        return (
            self.db.query(Book)
            .join(Book.authors)
            .filter(Author.id == author_id)
            .distinct()
            .all()
        )

    def get_by_author_name(
        self,
        author_name: str,
    ) -> list[Book]:
        return (
            self.db.query(Book)
            .join(Book.authors)
            .filter(Author.name == author_name)
            .distinct()
            .all()
        )

    def get_by_author_name_with_authors(
        self,
        author_name: str,
    ) -> list[Book]:
        return (
            self.db.query(Book)
            .join(Book.authors)
            .filter(Author.name == author_name)
            .options(selectinload(Book.authors))
            .distinct()
            .all()
        )

    def add_author(
        self,
        book: Book,
        author: Author,
    ) -> Book:
        if author not in book.authors:
            book.authors.append(author)

        self.db.add(book)
        self.db.commit()

        return self.get_with_authors(book.id)

    def remove_author(
        self,
        book: Book,
        author: Author,
    ) -> Book:
        if author in book.authors:
            book.authors.remove(author)

        self.db.add(book)
        self.db.commit()

        return self.get_with_authors(book.id)

    def update(self, book: Book) -> Book:
        self.db.add(book)
        self.db.commit()
        self.db.refresh(book)

        return book

    def delete(self, book: Book) -> None:
        self.db.delete(book)
        self.db.commit()
```

## `app/repositories/author_repository.py`

```python
from sqlalchemy.orm import (
    Session,
    selectinload,
)

from app.models.author import Author
from app.models.book import Book


class AuthorRepository:

    def __init__(self, db: Session):
        self.db = db

    def create(self, author: Author) -> Author:
        self.db.add(author)
        self.db.commit()
        self.db.refresh(author)

        return author

    def get_by_id(
        self,
        author_id: int,
    ) -> Author | None:
        return (
            self.db.query(Author)
            .filter(Author.id == author_id)
            .first()
        )

    def get_all(self) -> list[Author]:
        return self.db.query(Author).all()

    def get_with_books(
        self,
        author_id: int,
    ) -> Author | None:
        return (
            self.db.query(Author)
            .options(selectinload(Author.books))
            .filter(Author.id == author_id)
            .first()
        )

    def get_all_with_books(
        self,
    ) -> list[Author]:
        return (
            self.db.query(Author)
            .options(selectinload(Author.books))
            .all()
        )

    def get_by_book_id(
        self,
        book_id: int,
    ) -> list[Author]:
        return (
            self.db.query(Author)
            .join(Author.books)
            .filter(Book.id == book_id)
            .distinct()
            .all()
        )

    def get_by_book_title(
        self,
        book_title: str,
    ) -> list[Author]:
        return (
            self.db.query(Author)
            .join(Author.books)
            .filter(Book.title == book_title)
            .distinct()
            .all()
        )

    def add_book(
        self,
        author: Author,
        book: Book,
    ) -> Author:
        if book not in author.books:
            author.books.append(book)

        self.db.add(author)
        self.db.commit()

        return self.get_with_books(author.id)

    def remove_book(
        self,
        author: Author,
        book: Book,
    ) -> Author:
        if book in author.books:
            author.books.remove(book)

        self.db.add(author)
        self.db.commit()

        return self.get_with_books(author.id)

    def update(self, author: Author) -> Author:
        self.db.add(author)
        self.db.commit()
        self.db.refresh(author)

        return author

    def delete(self, author: Author) -> None:
        self.db.delete(author)
        self.db.commit()
```

## Особенности JOIN для связи многие ко многим

SQLAlchemy самостоятельно использует промежуточную таблицу:

```python
return (
    self.db.query(Book)
    .join(Book.authors)
    .filter(Author.name == author_name)
    .all()
)
```

Соединение будет выполнено через:

```text
books
    ↓
book_author
    ↓
authors
```

Промежуточную таблицу не нужно указывать вручную.

После JOIN основной объект может повторяться в SQL-результате. Поэтому для связи многие ко многим обычно добавляется:

```python
.distinct() # то есть уникальный
```

Пример:

```python
return (
    self.db.query(Book)
    .join(Book.authors)
    .filter(Author.name == author_name)
    .distinct()
    .all()
)
```

Обычный `join()` используется для фильтрации, но не для загрузки полного списка авторов книги.

Правильно:

```python
return (
    self.db.query(Book)
    .join(Book.authors)
    .filter(Author.name == author_name)
    .options(selectinload(Book.authors))
    .distinct()
    .all()
)
```

Здесь:

```python
.join(Book.authors)
```

находит книги по указанному автору.

```python
.options(selectinload(Book.authors))
```

загружает всех авторов каждой найденной книги.

---

# Памятка по JOIN в Repository

| Задача                                      | Решение                 |
| ------------------------------------------- | ----------------------- |
| Фильтрация по внешнему ключу текущей модели | JOIN не нужен           |
| Фильтрация по полю связанной модели         | `join()`                |
| Сортировка по полю связанной модели         | `join()`                |
| Получение объектов без обязательной связи   | `outerjoin()`           |
| Загрузка одиночной связи                    | `joinedload()`          |
| Загрузка коллекции                          | `selectinload()`        |
| Удаление повторов после Many-to-Many JOIN   | `distinct()`            |
| Добавление Many-to-Many связи               | `relationship.append()` |
| Удаление Many-to-Many связи                 | `relationship.remove()` |

Главное правило:

```text
join() используется для фильтрации и сортировки.

joinedload() и selectinload()
используются для загрузки связанных объектов.
```
