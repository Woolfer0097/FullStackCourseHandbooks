# Git и GitHub

## Установка Git

Проверить, установлен ли Git:

```powershell
git --version
```

Если Git не установлен — скачать и установить Git for Windows с официального сайта Git.

После установки снова проверить:

```powershell
git --version
```

---

## Первичная настройка Git

Указать имя:

```powershell
git config --global user.name "Ivan Ivanov"
```

Указать email:

```powershell
git config --global user.email "ivan@example.com"
```

Проверить настройки:

```powershell
git config --list
```

---

## Настройка SSH для GitHub

SSH нужен, чтобы Git мог работать с GitHub без ввода пароля от аккаунта.

Если при клонировании через HTTPS появляется ошибка:

```powershell
remote: Invalid username or token. Password authentication is not supported for Git operations.
fatal: Authentication failed
```

значит нужно использовать SSH-ссылку или token. В рамках курса используем SSH.

---

### Проверить, доступен ли SSH

```powershell
ssh
```

Если появилась инструкция вида `usage: ssh ...`, значит SSH установлен.

---

### Перейти в папку `.ssh`

```powershell
cd ~\.ssh
```

Посмотреть содержимое папки:

```powershell
ls
```

Если папки `.ssh` нет, создать её:

```powershell
mkdir ~\.ssh
cd ~\.ssh
```

---

### Создать SSH-ключ

```powershell
ssh-keygen -t ed25519 -C "ivan@example.com"
```

Когда появится вопрос:

```powershell
Enter file in which to save the key
```

можно нажать `Enter`.

Ключ сохранится примерно сюда:

```powershell
C:\Users\<имя_пользователя>\.ssh\id_ed25519
```

Когда появится вопрос про passphrase, для учебного проекта можно нажать `Enter`.

После создания ключа появятся два файла:

```powershell
id_ed25519
id_ed25519.pub
```

Важно:

- `id_ed25519` — приватный ключ, его никому нельзя отправлять
- `id_ed25519.pub` — публичный ключ, его нужно добавить в GitHub

---

### Скопировать публичный ключ

```powershell
cat ~\.ssh\id_ed25519.pub
```

Скопировать всю строку целиком. Она начинается примерно так:

```powershell
ssh-ed25519 AAAA...
```

---

### Добавить ключ в GitHub

1. Открыть GitHub
2. Нажать на аватар справа сверху
3. Перейти в `Settings`
4. Открыть `SSH and GPG keys`
5. Нажать `New SSH key`
6. В `Title` написать название компьютера, например `My laptop`
7. В `Key` вставить содержимое файла `id_ed25519.pub`
8. Нажать `Add SSH key`

---

### Проверить подключение к GitHub

```powershell
ssh -T git@github.com
```

При первом подключении может появиться вопрос:

```powershell
Are you sure you want to continue connecting?
```

Нужно написать:

```powershell
yes
```

Если всё настроено правильно, появится сообщение примерно такого вида:

```powershell
Hi username! You've successfully authenticated, but GitHub does not provide shell access.
```

Это значит, что SSH работает.

---

### Частые ошибки SSH

Ошибка:

```powershell
git@github.com: Permission denied (publickey).
fatal: Could not read from remote repository.
```

Что проверить:

1. SSH-ключ создан
2. В GitHub добавлен именно файл `id_ed25519.pub`
3. В репозитории используется SSH-ссылка c припиской git@

---

## Клонирование репозитория

1. Открыть проект на GitHub ![[Pasted image 20260706005918.png]]
2. Нажать на SSH
3. Скопировать ссылку вида:

```powershell
git@github.com:username/repository.git
```

Склонировать репозиторий:

```powershell
git clone git@github.com:username/repository.git <название_папки>
```

Пример:

```powershell
git clone git@github.com:ivan-ivanov/fullstack-project.git fullstack-project
```

Перейти в папку проекта:

```powershell
cd fullstack-project
```

Если Git пишет:

```powershell
warning: You appear to have cloned an empty repository.
```

это не ошибка. Это значит, что репозиторий пока пустой.

---
## Добавление преподавателя в гитхаб репозиторий

![[Pasted image 20260706064256.png]]
![[Pasted image 20260706064312.png]]
- Нажмите add people и пригласите меня по этому нику
```
Woolfer0097
```

## Получение последних изменений

Переключиться на основную ветку:

```powershell
git checkout main
```

Получить последние изменения:

```powershell
git pull
```

---

## Создание рабочей ветки

Создать новую ветку:

```powershell
git checkout -b feature/hello-world
```

Проверить текущую ветку:

```powershell
git branch
```

Текущая ветка будет отмечена символом `*`.

---

## Создание коммита

Проверить изменения:

```powershell
git status
```

Добавить все изменения:

```powershell
git add .
```

Создать коммит:

```powershell
git commit -m "feat: hello world"
```

Проверить историю коммитов:

```powershell
git log --oneline
```

---

## Отправка ветки в GitHub

Первый push новой ветки:

```powershell
git push -u origin feature/hello-world
```

Последующие push:

```powershell
git push
```

---

## Создание Pull Request

1. Открыть проект на GitHub
2. Перейти во вкладку `Pull requests`
3. Нажать `New pull request`
4. В `base` выбрать `main`
5. В `compare` выбрать свою ветку, например `feature/hello-world`
6. Нажать `Create pull request`
7. Добавить название и описание
8. Нажать `Create pull request`

---

## Обновление своей ветки после изменений в main

Нужно, если в `main` появились новые изменения, а вы продолжаете работать в своей ветке.

```powershell
git checkout main
git pull

git checkout feature/hello-world
git rebase main
```

Если после rebase появились конфликты:

1. Исправить конфликтующие файлы
2. Добавить исправленные файлы:

```powershell
git add .
```

3. Продолжить rebase:

```powershell
git rebase --continue
```

Если нужно отменить rebase:

```powershell
git rebase --abort
```

---

## Force Push после Rebase

После `git rebase` нужно обновить удалённую ветку:

```powershell
git push --force-with-lease
```

Использовать только после `rebase` или `squash commits`.

Не использовать обычный `--force`, лучше использовать:

```powershell
git push --force-with-lease
```

---

## Полезные команды

Посмотреть статус:

```powershell
git status
```

Посмотреть ветки:

```powershell
git branch
```

Посмотреть удалённый репозиторий:

```powershell
git remote -v
```

Посмотреть историю:

```powershell
git log --oneline
```

Переключиться на ветку:

```powershell
git checkout <название_ветки>
```