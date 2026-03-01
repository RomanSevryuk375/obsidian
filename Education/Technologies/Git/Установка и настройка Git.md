---
aliases:
tags:
  - devops
  - git
  - tools
date: 2026-03-01 22:38
status:
---
Процесс установки иинструмента и обязательная конфигурация `user.name` и `user.email`, которые будут записываться в метаданные каждого [[git commit|коммита]] для идентификации автора.

---

## Уровни конфигурации
[[Git]] читает настройки в следующем порядке приоритета (локальный перекрывает глобальный):
1.  **Local** (`/path/to/repo/.git/config`) — настройки конкретного репозитория.
2.  **Global** (`~/.gitconfig`) — настройки текущего пользователя (основной уровень).
3.  **System** (`/etc/gitconfig`) — настройки для всех пользователей ОС.

## Шпаргалка по командам

```bash
# --- 1. Identity (Обязательно) ---
# Без этого Git будет ругаться при первой попытке коммита
git config --global user.name "Ivan Ivanov"
git config --global user.email "ivan@example.com"

# --- 2. Базовая гигиена ---
# Установка дефолтной ветки 'main' (вместо устаревшего 'master')
git config --global init.defaultBranch main

# Редактор для сообщений коммитов (vim, nano, code)
git config --global core.editor "code --wait"

# Обработка окончаний строк (Line Endings)
# Windows: конвертировать CRLF -> LF при коммите
git config --global core.autocrlf true
# Mac/Linux: не трогать окончания (input)
git config --global core.autocrlf input

# --- 3. Алиасы (Ускорители) ---
# Сокращения для частых команд
git config --global alias.co checkout
git config --global alias.st status
git config --global alias.ci commit

# --- 5. Проверка ---
# Список всех активных настроек
git config --list
# Узнать, в каком файле прописана конкретная настройка
git config --show-origin user.email
```

## Связанные темы
*   [[Что такое VCS и Git]]
*   [[.gitignore]] — можно настроить глобальный ignore-файл через `core.excludesFile`.
*   [[git init]] — инициализация репозитория после настройки.