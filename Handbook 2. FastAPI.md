# FastAPI

## Создание виртуального окружения

Создать виртуальное окружение:

```powershell
python -m venv .venv
```

Активировать виртуальное окружение:

```powershell
.\.venv\Scripts\Activate.ps1
```

---

## Установка зависимостей

Установить FastAPI и зависимости для `.env`:

```powershell
pip install "fastapi[standard]" pydantic-settings python-dotenv
```

Сохранить зависимости в `requirements.txt`:

```powershell
pip freeze > requirements.txt
```

Установка зависимостей из `requirements.txt`:

```powershell
pip install -r requirements.txt
```

---

## Базовая структура проекта

```text
project/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── api/
│   │   ├── __init__.py
│   │   └── health.py
│   └── config/
│       ├── __init__.py
│       └── config.py
├── .env
├── .env.example
├── .gitignore
└── requirements.txt
```

---

## `.gitignore`

Файл `.gitignore`:

```gitignore
.venv/
__pycache__/
*.pyc

.env

.idea/
.vscode/

*.db
```

Файл `.env` нельзя добавлять в GitHub, потому что в нём могут быть секреты, пароли и токены.

---

## `.env`

Файл `.env`:

```env
APP_NAME=FullStack API
APP_VERSION=0.1.0
DEBUG=true

HOST=127.0.0.1
PORT=8000

DATABASE_URL=sqlite:///./database.db
```

---

## `.env.example`

Файл `.env.example`:

```env
APP_NAME=FullStack API
APP_VERSION=0.1.0
DEBUG=true

HOST=127.0.0.1
PORT=8000

DATABASE_URL=sqlite:///./database.db
```

`.env.example` можно добавлять в GitHub.

Этот файл показывает, какие переменные окружения нужны проекту.

---

## `app/config/config.py`

Файл `app/config/config.py`:

```python
from functools import lru_cache

from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    app_name: str = "FullStack API"
    app_version: str = "0.1.0"
    debug: bool = True

    host: str = "127.0.0.1"
    port: int = 8000

    database_url: str = "sqlite:///./database.db"

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )


@lru_cache
def get_settings() -> Settings:
    return Settings()
```

---

## `app/api/health.py`

Файл `app/api/health.py`:

```python
from fastapi import APIRouter

from app.config.config import get_settings


router = APIRouter(
    prefix="/health",
    tags=["health"],
)


@router.get("")
def health_check():
    settings = get_settings()

    return {
        "status": "ok",
        "app_name": settings.app_name,
        "version": settings.app_version,
        "debug": settings.debug,
    }
```

---

## `app/main.py`

Файл `app/main.py`:

```python
from fastapi import FastAPI

from app.api.health import router as health_router
from app.config.config import get_settings


settings = get_settings()


app = FastAPI(
    title=settings.app_name,
    version=settings.app_version,
    debug=settings.debug,
)


app.include_router(health_router)


@app.get("/")
def root():
    return {
        "message": f"{settings.app_name} is running",
    }
```

---

## Запуск приложения

Запустить приложение:

```powershell
uvicorn app.main:app --reload
```

После запуска открыть в браузере:

```text
http://127.0.0.1:8000
```

Документация API:

```text
http://127.0.0.1:8000/docs
```

Остановить сервер:

```powershell
Ctrl + C
```

---

## Альтернативный запуск через FastAPI CLI

Запуск в режиме разработки:

```powershell
fastapi dev app/main.py
```

Для курса можно использовать основной вариант:

```powershell
uvicorn app.main:app --reload
```

---

## Проверка API

Проверить главную ручку:

```text
GET http://127.0.0.1:8000/
```

Ожидаемый ответ:

```json
{
  "message": "FullStack API is running"
}
```

Проверить health-check:

```text
GET http://127.0.0.1:8000/health
```

Ожидаемый ответ:

```json
{
  "status": "ok",
  "app_name": "FullStack API",
  "version": "0.1.0",
  "debug": true
}
```

---

## Проверка переменных окружения

Изменить в `.env`:

```env
APP_NAME=Books API
```

Перезапустить сервер.

Проверить:

```text
http://127.0.0.1:8000/health
```

Ожидаемый результат:

```json
{
  "status": "ok",
  "app_name": "Books API",
  "version": "0.1.0",
  "debug": true
}
```

---

## Добавление новой ручки

Пример простой ручки в `app/main.py`:

```python
@app.get("/hello")
def hello():
    return {
        "message": "Hello, FastAPI!",
    }
```

Проверить в браузере:

```text
http://127.0.0.1:8000/hello
```

---

## Пример ручки с path parameter

```python
@app.get("/items/{item_id}")
def get_item(item_id: int):
    return {
        "item_id": item_id,
    }
```

Пример запроса:

```text
http://127.0.0.1:8000/items/1
```

---

## Пример ручки с query parameter

```python
@app.get("/search")
def search_items(query: str, limit: int = 10):
    return {
        "query": query,
        "limit": limit,
    }
```

Пример запроса:

```text
http://127.0.0.1:8000/search?query=python&limit=5
```

---

## Пример Pydantic-схемы

```python
from pydantic import BaseModel


class ItemCreate(BaseModel):
    title: str
    description: str | None = None
    price: float
```

Пример POST-ручки:

```python
@app.post("/items")
def create_item(item: ItemCreate):
    return {
        "message": "Item created",
        "item": item,
    }
```

Пример тела запроса:

```json
{
  "title": "Keyboard",
  "description": "Mechanical keyboard",
  "price": 5000
}
```

---

## Вынос ручек в отдельный router

Пример файла:

```text
app/api/items.py
```

Код:

```python
from fastapi import APIRouter
from pydantic import BaseModel


router = APIRouter(
    prefix="/items",
    tags=["items"],
)


class ItemCreate(BaseModel):
    title: str
    description: str | None = None
    price: float


@router.get("")
def get_items():
    return [
        {
            "id": 1,
            "title": "Keyboard",
            "price": 5000,
        }
    ]


@router.post("")
def create_item(item: ItemCreate):
    return {
        "message": "Item created",
        "item": item,
    }
```

Подключить router в `app/main.py`:

```python
from app.api.items import router as items_router

app.include_router(items_router)
```

---

## Минимальная проверка перед коммитом

Проверить структуру проекта:

```powershell
ls
```

Проверить запуск:

```powershell
uvicorn app.main:app --reload
```

Проверить документацию:

```text
http://127.0.0.1:8000/docs
```

Проверить статус Git:

```powershell
git status
```

Добавить изменения:

```powershell
git add .
```

Создать коммит:

```powershell
git commit -m "feat: add base fastapi app"
```

Убедиться что изменения находятся в новой ветке feat/base-structure
```
git checkout -b feat/base-structure
```

Запушить изменения
```powershell
git push
```

Затем в гитхабе открыть новый pull-request и слить изменения в main ветку