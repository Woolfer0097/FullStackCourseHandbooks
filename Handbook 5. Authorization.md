## Что получится в конце занятия

В результате занятия в API появится регистрация, логин, JWT-авторизация и простая проверка ролей.

Общая схема останется такой же layered, как в прошлом занятии:

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

Для авторизации flow будет таким:

```text
register -> hash password -> save user
login -> verify password -> create JWT
protected route -> decode JWT -> get current user from DB
admin route -> check role -> allow or return 403
```

Важно: в этом проекте JWT хранит только `user_id` в поле `sub`. Роль пользователя в токен не записывается. На каждом защищённом запросе приложение достаёт пользователя из базы данных и проверяет актуальную роль из БД.

Структура проекта после занятия:

```text
app/
│
├── auth.py
├── database.py
├── main.py
│
├── api/
│   ├── auth.py
│   ├── health.py
│   └── users.py
│
├── config/
│   └── config.py
│
├── handlers/
│   ├── auth.py
│   ├── books.py
│   └── users.py
│
├── models/
│   ├── book.py
│   └── user.py
│
├── repositories/
│   ├── book_repository.py
│   └── user_repository.py
│
├── schemas/
│   ├── book.py
│   ├── token.py
│   └── user.py
│
└── services/
    ├── auth_service.py
    ├── book_service.py
    └── user_service.py
```

Endpoint'ы, которые появятся:

```text
POST /auth/register   -> регистрация пользователя
POST /auth/login      -> получение bearer token
GET /users/me         -> текущий авторизованный пользователь
GET /admin/users      -> список пользователей, только для admin
```

Старые endpoint'ы из прошлого занятия остаются в проекте.

---

# Шаг 1. Обновить зависимости

Открыть файл:

```text
requirements.txt
```

Файл должен содержать зависимости для FastAPI, SQLAlchemy, настроек, JWT и хеширования паролей:

```text
fastapi
uvicorn
sqlalchemy
pydantic-settings
python-jose[cryptography]
passlib[bcrypt]
bcrypt==3.2.2
httpx
```

Здесь:

- `python-jose[cryptography]` нужен для создания и проверки JWT;
- `passlib[bcrypt]` нужен для хеширования паролей;
- `bcrypt==3.2.2` фиксирует совместимую версию bcrypt для учебного проекта.

---

# Шаг 2. Добавить настройки авторизации

Открыть файл:

```text
app/config/config.py
```

Привести файл к следующему виду:

```python
from functools import lru_cache
from pathlib import Path

from pydantic_settings import BaseSettings, SettingsConfigDict


PROJECT_ROOT = Path(__file__).resolve().parents[2]
PROJECT_ENV_FILE = PROJECT_ROOT / ".env"


class Settings(BaseSettings):
    app_name: str
    app_version: str = "0.1.0"
    debug: bool = True
    host: str = "127.0.0.1"
    port: int = 8000
    database_url: str = "sqlite:///./database.db"
    secret_key: str = "change-me"
    algorithm: str = "HS256"
    access_token_expire_minutes: int = 30

    model_config = SettingsConfigDict(
        env_file=PROJECT_ENV_FILE,
        env_file_encoding="utf-8",
        extra="ignore",
    )


@lru_cache
def get_settings() -> Settings:
    return Settings()
```

`.env` загружается по абсолютному пути от корня проекта, поэтому настройки не зависят от папки, из которой запустили `app/main.py`.

---

# Шаг 3. Создать модель пользователя и роли

Создать файл:

```text
app/models/user.py
```

Добавить код:

```python
from enum import Enum

from sqlalchemy import Boolean, String
from sqlalchemy.orm import Mapped, mapped_column

from app.database import Base


class UserRole(str, Enum):
    USER = "user"
    ADMIN = "admin"


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    email: Mapped[str] = mapped_column(
        String(255),
        unique=True,
        index=True,
        nullable=False,
    )
    hashed_password: Mapped[str] = mapped_column(String(255), nullable=False)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True, nullable=False)
    role: Mapped[str] = mapped_column(
        String(50),
        default=UserRole.USER.value,
        nullable=False,
    )
```

В базе хранится не обычный пароль, а `hashed_password`. Обычный пароль нужен только в момент регистрации или логина.

Роли минимальные:

```text
user  -> обычный пользователь
admin -> администратор
```

Через обычную регистрацию пользователь будет получать только роль `user`.

---

# Шаг 4. Создать схемы пользователя

Создать файл:

```text
app/schemas/user.py
```

Добавить код:

```python
from pydantic import BaseModel, ConfigDict, Field, field_validator

from app.models.user import UserRole


class UserCreate(BaseModel):
    email: str = Field(min_length=3, max_length=255)
    password: str = Field(min_length=6, max_length=128)

    @field_validator("email")
    @classmethod
    def normalize_email(cls, value: str) -> str:
        email = value.strip().lower()

        if "@" not in email:
            raise ValueError("Email must contain @")

        return email


class UserLogin(BaseModel):
    email: str = Field(min_length=3, max_length=255)
    password: str = Field(min_length=1, max_length=128)

    @field_validator("email")
    @classmethod
    def normalize_email(cls, value: str) -> str:
        email = value.strip().lower()

        if "@" not in email:
            raise ValueError("Email must contain @")

        return email


class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    email: str
    is_active: bool
    role: UserRole


UserRead = UserResponse
```

Здесь есть три основные схемы:

- `UserCreate` — данные для регистрации;
- `UserLogin` — данные для логина;
- `UserResponse` — данные, которые API возвращает наружу.

Пароль в `UserResponse` не попадает.

---

# Шаг 5. Создать схемы токена

Создать файл:

```text
app/schemas/token.py
```

Добавить код:

```python
from pydantic import BaseModel


class Token(BaseModel):
    access_token: str
    token_type: str = "bearer"


class TokenData(BaseModel):
    user_id: int
```

`Token` — это ответ на успешный логин.

`TokenData` — это данные, которые приложение достаёт из JWT после проверки токена. Сейчас там только `user_id`.

---

# Шаг 6. Создать UserRepository

Создать файл:

```text
app/repositories/user_repository.py
```

Добавить код:

```python
from sqlalchemy.orm import Session

from app.models.user import User


class UserRepository:

    def __init__(self, db: Session):
        self.db = db

    def create(self, user: User) -> User:
        return self._save(user)

    def get_all(self) -> list[User]:
        return self.db.query(User).all()

    def get_by_id(self, user_id: int):
        return (
            self.db.query(User)
            .filter(User.id == user_id)
            .first()
        )

    def get_by_email(self, email: str):
        return (
            self.db.query(User)
            .filter(User.email == email)
            .first()
        )

    def update(self, user: User) -> User:
        return self._save(user)

    def _save(self, user: User) -> User:
        self.db.add(user)
        self.db.commit()
        self.db.refresh(user)

        return user
```

Repository отвечает только за работу с таблицей `users`:

- создать пользователя;
- получить всех пользователей;
- найти пользователя по `id`;
- найти пользователя по `email`;
- сохранить изменения.

Бизнес-логики здесь нет. Например, проверка дубликатов email будет в Service.

---

# Шаг 7. Создать UserService

Создать файл:

```text
app/services/user_service.py
```

Добавить код:

```python
from fastapi import HTTPException, status
from sqlalchemy.orm import Session

from app.auth import hash_password
from app.models.user import User, UserRole
from app.repositories.user_repository import UserRepository
from app.schemas.user import UserCreate


class UserService:

    def __init__(self, db: Session):
        self.repository = UserRepository(db)

    def create_user(self, schema: UserCreate) -> User:
        existing_user = self.repository.get_by_email(schema.email)

        if existing_user is not None:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail="User with this email already exists",
            )

        user = User(
            email=schema.email,
            hashed_password=hash_password(schema.password),
            is_active=True,
            role=UserRole.USER.value,
        )

        return self.repository.create(user)

    def get_users(self) -> list[User]:
        return self.repository.get_all()

    def get_user(self, user_id: int) -> User:
        user = self.repository.get_by_id(user_id)

        if user is None:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail="User not found",
            )

        return user
```

Что делает `UserService`:

- проверяет, что email ещё не занят;
- хеширует пароль через `hash_password()`;
- создаёт пользователя с ролью `user`;
- возвращает список пользователей для admin endpoint'а;
- умеет получить пользователя по `id`.

Важно: роль `admin` не передаётся через регистрацию. Это защищает приложение от ситуации, когда пользователь сам назначает себя администратором.

---

# Шаг 8. Создать AuthService

Создать файл:

```text
app/services/auth_service.py
```

Добавить код:

```python
from fastapi import HTTPException, status
from sqlalchemy.orm import Session

from app.auth import create_access_token, verify_password
from app.models.user import User
from app.repositories.user_repository import UserRepository
from app.schemas.token import Token
from app.schemas.user import UserLogin


class AuthService:

    def __init__(self, db: Session):
        self.repository = UserRepository(db)

    def login(self, schema: UserLogin) -> Token:
        user = self._authenticate_user(schema.email, schema.password)
        access_token = create_access_token(user.id)

        return Token(access_token=access_token)

    def _authenticate_user(self, email: str, password: str) -> User:
        user = self.repository.get_by_email(email)

        if user is None:
            raise self._invalid_credentials_error()

        if not verify_password(password, user.hashed_password):
            raise self._invalid_credentials_error()

        if not user.is_active:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Inactive user",
                headers={"WWW-Authenticate": "Bearer"},
            )

        return user

    def _invalid_credentials_error(self) -> HTTPException:
        return HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect email or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
```

`AuthService` отвечает за логин:

```text
найти пользователя -> проверить пароль -> создать JWT -> вернуть token
```

Если email или пароль неверные, API вернёт одинаковую ошибку:

```json
{
  "detail": "Incorrect email or password"
}
```

---

# Шаг 9. Создать утилиты авторизации

Создать файл:

```text
app/auth.py
```

Добавить код:

```python
from collections.abc import Iterable
from datetime import datetime, timedelta, timezone

from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer
from jose import JWTError, jwt
from passlib.context import CryptContext
from sqlalchemy.orm import Session

from app.config.config import get_settings
from app.database import get_db
from app.models.user import User, UserRole
from app.repositories.user_repository import UserRepository
from app.schemas.token import TokenData

password_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
bearer_scheme = HTTPBearer(auto_error=False)


def hash_password(password: str) -> str:
    return password_context.hash(password)


def verify_password(plain_password: str, hashed_password: str) -> bool:
    return password_context.verify(plain_password, hashed_password)


def create_access_token(user_id: int) -> str:
    settings = get_settings()
    expires_at = datetime.now(timezone.utc) + timedelta(
        minutes=settings.access_token_expire_minutes,
    )
    payload = {
        "sub": str(user_id),
        "exp": expires_at,
    }

    return jwt.encode(
        payload,
        settings.secret_key,
        algorithm=settings.algorithm,
    )


def decode_access_token(token: str) -> TokenData:
    settings = get_settings()

    try:
        payload = jwt.decode(
            token,
            settings.secret_key,
            algorithms=[settings.algorithm],
        )
        subject = payload.get("sub")

        if subject is None:
            raise _credentials_error()

        return TokenData(user_id=int(subject))
    except (JWTError, ValueError) as exc:
        raise _credentials_error() from exc


def get_current_user(
    credentials: HTTPAuthorizationCredentials | None = Depends(bearer_scheme),
    db: Session = Depends(get_db),
) -> User:
    if credentials is None:
        raise _credentials_error()

    if credentials.scheme.lower() != "bearer":
        raise _credentials_error()

    token_data = decode_access_token(credentials.credentials)
    user = UserRepository(db).get_by_id(token_data.user_id)

    if user is None:
        raise _credentials_error()

    if not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Inactive user",
            headers={"WWW-Authenticate": "Bearer"},
        )

    return user


def require_role(required_role: UserRole):
    return require_roles([required_role])


def require_roles(allowed_roles: Iterable[UserRole]):
    allowed_role_values = {role.value for role in allowed_roles}

    def role_checker(current_user: User = Depends(get_current_user)) -> User:
        if current_user.role not in allowed_role_values:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Not enough permissions",
            )

        return current_user

    return role_checker


def _credentials_error() -> HTTPException:
    return HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
```

В этом файле находятся четыре группы функций.

Работа с паролем:

```text
hash_password()   -> превратить обычный пароль в хеш
verify_password() -> проверить обычный пароль против хеша
```

Работа с JWT:

```text
create_access_token() -> создать токен
decode_access_token() -> проверить токен и достать user_id
```

Текущий пользователь:

```text
get_current_user() -> прочитать Authorization: Bearer ..., проверить JWT, найти пользователя в БД
```

Проверка ролей:

```text
require_role(UserRole.ADMIN) -> пропустить только admin
require_roles([...])         -> пропустить несколько ролей
```

Ещё раз: роль не хранится в JWT. Токен хранит только `sub`, то есть `user_id`. Поэтому если роль пользователя изменилась в базе, защищённые endpoint'ы будут проверять актуальную роль из БД.

---

# Шаг 10. Создать auth handler

Создать файл:

```text
app/handlers/auth.py
```

Добавить код:

```python
from fastapi import APIRouter, Depends, status
from sqlalchemy.orm import Session

from app.database import get_db
from app.schemas.token import Token
from app.schemas.user import UserCreate, UserLogin, UserResponse
from app.services.auth_service import AuthService
from app.services.user_service import UserService

router = APIRouter(
    prefix="/auth",
    tags=["auth"],
)


def get_user_service(db: Session = Depends(get_db)) -> UserService:
    return UserService(db)


def get_auth_service(db: Session = Depends(get_db)) -> AuthService:
    return AuthService(db)


@router.post(
    "/register",
    response_model=UserResponse,
    status_code=status.HTTP_201_CREATED,
)
def register_user(
    schema: UserCreate,
    service: UserService = Depends(get_user_service),
):
    return service.create_user(schema)


@router.post(
    "/login",
    response_model=Token,
)
def login_user(
    schema: UserLogin,
    service: AuthService = Depends(get_auth_service),
):
    return service.login(schema)
```

Этот handler создаёт два endpoint'а:

```text
POST /auth/register
POST /auth/login
```

Handler не хеширует пароль и не создаёт JWT сам. Он только принимает HTTP-запрос и передаёт работу в Service.

---

# Шаг 11. Создать users handler

Создать файл:

```text
app/handlers/users.py
```

Добавить код:

```python
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session

from app.auth import get_current_user, require_role
from app.database import get_db
from app.models.user import User, UserRole
from app.schemas.user import UserResponse
from app.services.user_service import UserService

router = APIRouter(tags=["users"])


def get_user_service(db: Session = Depends(get_db)) -> UserService:
    return UserService(db)


@router.get(
    "/users/me",
    response_model=UserResponse,
)
def get_my_user(current_user: User = Depends(get_current_user)):
    return current_user


@router.get(
    "/admin/users",
    response_model=list[UserResponse],
)
def get_admin_users(
    current_user: User = Depends(require_role(UserRole.ADMIN)),
    service: UserService = Depends(get_user_service),
):
    return service.get_users()
```

Здесь есть два уровня доступа:

```text
GET /users/me     -> любой авторизованный пользователь
GET /admin/users  -> только пользователь с role="admin"
```

`current_user` в `get_admin_users()` нужен не потому, что используется внутри функции. Он нужен, чтобы FastAPI выполнил dependency `require_role(UserRole.ADMIN)` перед входом в endpoint.

---

# Шаг 12. Подключить модели и router'ы в `main.py`

Открыть файл:

```text
app/main.py
```

Привести файл к следующему виду:

```python
import sys
from pathlib import Path

import uvicorn
from fastapi import FastAPI

from app.api.health import router as health_router
from app.config.config import get_settings
from app.database import Base, engine
from app.handlers.auth import router as auth_router
from app.handlers.books import router as books_router
from app.handlers.users import router as users_router
from app.models.book import Book
from app.models.user import User

settings = get_settings()
app = FastAPI(
    title=settings.app_name,
    version=settings.app_version,
    debug=settings.debug,
)

Base.metadata.create_all(bind=engine)
app.include_router(auth_router)
app.include_router(books_router)
app.include_router(users_router)
app.include_router(health_router)


@app.get("/")
def read_root() -> dict[str, str]:
    return {"message": f"{settings.app_name} is running"}


if __name__ == '__main__':
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

Импорт `User` нужен, чтобы SQLAlchemy увидел модель перед вызовом:

```python
Base.metadata.create_all(bind=engine)
```

После запуска приложения таблица `users` будет создана в учебной SQLite базе, если её ещё нет.

Если ты раньше экспериментировал и уже создал таблицу `users` с другой структурой, для учебного проекта проще удалить старый `books.db` и запустить приложение снова. В реальных проектах для этого используют миграции, но это не тема текущего занятия.

---

# Шаг 13. Установить зависимости

Установить зависимости из файла:

```bash
pip install -r requirements.txt
```

Не забудь активировать виртуальное окружение.

Linux/MacOS:

```bash
source ./.venv/bin/activate
```

Windows PowerShell:

```powershell
.\.venv\Scripts\Activate.ps1
```

---

# Шаг 14. Запустить приложение и открыть Swagger

Запустить приложение можно так:

```bash
uvicorn app.main:app --reload
```

После запуска открыть Swagger:

```text
http://127.0.0.1:8000/docs
```

Дальше все проверки можно выполнить прямо через Swagger.

---

# Шаг 15. Проверить регистрацию обычного пользователя

В Swagger открыть:

```text
POST /auth/register
```

Тело запроса:

```json
{
  "email": "student@example.com",
  "password": "secret123"
}
```

Ожидаемый результат:

```json
{
  "id": 1,
  "email": "student@example.com",
  "is_active": true,
  "role": "user"
}
```

`id` может отличаться, если в базе уже есть пользователи.

В ответе нет поля `password` и нет поля `hashed_password`. Пароль хранится в базе только в хешированном виде.

---

# Шаг 16. Проверить повторную регистрацию

Повторить тот же запрос:

```text
POST /auth/register
```

С тем же телом:

```json
{
  "email": "student@example.com",
  "password": "secret123"
}
```

Ожидаемый результат:

```json
{
  "detail": "User with this email already exists"
}
```

Статус ответа должен быть:

```text
400 Bad Request
```

Так работает проверка дубликата email в `UserService`.

---

# Шаг 18. Проверить логин и получить bearer token

В Swagger открыть:

```text
POST /auth/login
```

Тело запроса:

```json
{
  "email": "student@example.com",
  "password": "secret123"
}
```

Ожидаемый результат:

```json
{
  "access_token": "eyJ...",
  "token_type": "bearer"
}
```

Скопировать значение `access_token`. Сам токен будет длинной строкой.

Внутри токена сейчас есть:

```text
sub -> user_id
exp -> время истечения токена
```

Роли пользователя внутри JWT нет.

---

# Шаг 19. Авторизоваться в Swagger

В Swagger нажать кнопку:

```text
Authorize
```

Вставить скопированный `access_token`.

Обычно для схемы `HTTPBearer` в Swagger нужно вставить только сам токен, без кавычек и без слова `Bearer`. Если интерфейс явно просит полный заголовок, тогда формат такой:

```text
Bearer <access_token>
```

После авторизации Swagger будет отправлять заголовок:

```text
Authorization: Bearer <access_token>
```

---

# Шаг 20. Проверить `/users/me`

В Swagger открыть:

```text
GET /users/me
```

Ожидаемый результат:

```json
{
  "id": 1,
  "email": "student@example.com",
  "is_active": true,
  "role": "user"
}
```

Если удалить токен или вставить неправильный токен, API вернёт ошибку авторизации:

```json
{
  "detail": "Could not validate credentials"
}
```

`GET /users/me` работает через dependency `get_current_user()`:

```text
прочитать Bearer token -> decode JWT -> достать user_id -> найти пользователя в БД
```

---

# Шаг 21. Проверить запрет `/admin/users` для обычного пользователя

Пока пользователь имеет роль `user`, открыть:

```text
GET /admin/users
```

Ожидаемый результат:

```json
{
  "detail": "Not enough permissions"
}
```

Статус ответа должен быть:

```text
403 Forbidden
```

Это означает, что авторизация прошла успешно, но роли `user` недостаточно для admin endpoint'а.

---

# Шаг 22. Повысить пользователя до admin вручную

В этом учебном проекте регистрация не выдаёт роль `admin`. Для проверки RBAC можно вручную изменить роль пользователя через Python console.

Открыть Python из корня проекта:

```bash
python
```

Выполнить код:

```python
from app.database import SessionLocal
from app.models.user import User, UserRole

db = SessionLocal()

try:
    user = db.query(User).filter(User.email == "student@example.com").first()

    if user is None:
        raise RuntimeError("User not found")

    user.role = UserRole.ADMIN.value
    db.commit()
finally:
    db.close()
```

Этот код нужен только для учебной проверки. В реальном проекте создание админа обычно делают через seed-скрипт, миграцию данных или отдельный защищённый admin-инструмент.

---

# Шаг 23. Залогиниться снова и проверить `/admin/users`

Снова выполнить:

```text
POST /auth/login
```

Тело запроса:

```json
{
  "email": "student@example.com",
  "password": "secret123"
}
```

Скопировать новый `access_token` и снова нажать `Authorize` в Swagger.

Технически роль не хранится в JWT, поэтому приложение читает актуальную роль из БД на каждом защищённом запросе. Но для чистой проверки удобно залогиниться ещё раз и использовать свежий токен.

Теперь открыть:

```text
GET /admin/users
```

Ожидаемый результат:

```json
[
  {
    "id": 1,
    "email": "student@example.com",
    "is_active": true,
    "role": "admin"
  }
]
```

Если в базе есть другие пользователи, они тоже будут в списке.

---

# Шаг 24. Проверить неправильный логин

В Swagger открыть:

```text
POST /auth/login
```

Тело запроса с неправильным паролем:

```json
{
  "email": "student@example.com",
  "password": "wrong-password"
}
```

Ожидаемый результат:

```json
{
  "detail": "Incorrect email or password"
}
```

Статус ответа должен быть:

```text
401 Unauthorized
```

---

# Шаг 25. Проверить, что старые books routes работают

После добавления авторизации старые endpoint'ы книг должны остаться доступными:

```text
POST /books/
GET /books/
GET /books/{book_id}
PATCH /books/{book_id}
DELETE /books/{book_id}
```

Например, проверить:

```text
GET /books/
```

Если приложение отвечает без ошибки импорта и список книг возвращается, значит новые router'ы не сломали предыдущую часть проекта.

---

# Итог занятия

К концу занятия должно быть реализовано:

- Добавлены настройки авторизации:
  - `secret_key`
  - `algorithm`
  - `access_token_expire_minutes`
- Создана модель пользователя `User`.
- Добавлен enum ролей `UserRole`:
  - `user`
  - `admin`
- Созданы Pydantic-схемы:
  - `UserCreate`
  - `UserLogin`
  - `UserResponse`
  - `UserRead`
  - `Token`
  - `TokenData`
- Создан `UserRepository` для работы с таблицей `users`.
- Создан `UserService` для регистрации и получения пользователей.
- Создан `AuthService` для логина.
- Добавлен файл `app/auth.py` с функциями:
  - `hash_password()`
  - `verify_password()`
  - `create_access_token()`
  - `decode_access_token()`
  - `get_current_user()`
  - `require_role()`
  - `require_roles()`
- Реализованы endpoint'ы:
  - `POST /auth/register`
  - `POST /auth/login`
  - `GET /users/me`
  - `GET /admin/users`
- Добавлена обработка ошибок:
  - `400 User with this email already exists`
  - `401 Incorrect email or password`
  - `401 Could not validate credentials`
  - `403 Not enough permissions`
- Пароли сохраняются только в хешированном виде.
- JWT хранит только `user_id` в `sub` и не хранит роль.
- Роль пользователя читается из базы данных на каждом защищённом запросе.
- Обычная регистрация создаёт только пользователя с ролью `user`.
- Admin-доступ проверяется через `require_role(UserRole.ADMIN)`.
