---
title: RSync для локального бэкапа кода
description: Разбираюсь как бэкапить код на флешку с учетом gitignore
pubDate: 2026-1-17
---

Иногда хочется сделать бэкап проекта на сменный носитель (в случае, если нет доступа к интернету или не хоется размещать туда код)

## Есть решение с git

```bash
cd $BACKUP_DIR # Переходим в дирректорию с бэкапом
git init --bare # Инициализируем репозиторий в дирректории
cd $PROJECT_DIR # И обратно в дирректорию с проектом
git remote add backup $PROJECT_DIR # Добавляем директорию бэкапа как удаленный репозиторий c именем "backup"
git push backup # Пушим туда изменения
```

В случае восстановления из бэкапа работаем как с обычным удаленным репозиторием.

```bash
git clone $BACKUP_DIR
```

Плюсы такого решения:

+ Это git, он привычный
+ Точно учитываются `.gitignore` файлы и т.д.

Минусы такого решения:

+ Нужно помнить, в какой `remote` пушить
+ Сработают push хуки git
+ Нужно пушить временные ветки
+ При изменении дирректории (например, точки монтирования) будут веселые игрища вида "удали старый, добавь новый"
+ Для каждого проекта нужно заводить отдельный репозиторий

И, главное, по сути это не совсем бэкап, а именно удаленный репозиторий. Файлы не добавленные в git не попадут туда, можно забыть запушить ветку и т.д.

С другой стороны, не хотелось бы бэкапить лишнее (бэкап `node_modules` даже звучит кошмарно).

## Попробуем научить rsync делать то, что нам нужно

Самый простой путь

```bash
rsync -a $PROJECT_DIR $BACKUP_DIR/
```

Просто и незатейливо воссоздаст дирректорию проекта в папке с бэкапами.

Проблемы:

+ Лишние файлы (из списка gitignore)
+ Удаленные файлы из каталога проекта останутся на месте, что при восстановлении из бэкапа может удивить

Вторая проблема решается опцией `--delete`:

```bash
rsync -a --delete $PROJECT_DIR $BACKUP_DIR/
```

### Список игнорируемых файлов

Описание правил игнорирования:

[git-scm.com/docs/gitignore](https://git-scm.com/docs/gitignore)

И что то не хочется такое парсить. Надо посмотреть, может ли git сформировать список. Оказывается может:

```bash
git ls-files -i \ # Игнорируемы git файлы
-o \ # Не учитывать файлы в индексе. Если вдруг почему то надо - тут должно быть -i
--directory \ # Не раскрывать содержимое дирректорий, если они игнорируются целиком
--exclude-standard # Просто нужно. Я так понял, что есть разные способы передать файлы для исключения, но не углублялся
```

Тут список всех существующих игнорируемых файлов. Без `--directory` тоже можно, но там получается сильно страшней.

### Запихиваем его в rsync

#### Учимся доставать список игнорируемых файлов из дирректории проекта

```bash
git --git-dir=$PROJECT_DIR/.git --work-tree=$PROJECT_DIR ls-files -i -o --directory --exclude-standard
```

#### Передаем в rsync

Мы можем передать список для игнорирования через стандартный ввод (`-` вместо имени файла для опции `exclude-from`).

```bash
git --git-dir=$PROJECT_DIR/.git --work-tree=$PROJECT_DIR ls-files -i -o --directory --exclude-standard | rsync -a --delete --exclude-from - $PROJECT_DIR $BACKUP_DIR
```

## Итого

Получаем скрипт

```bash
#!/usr/bin/env bash

export BACKUP_DIR=/backups/
export RSYNC_OPTIONS=-a # for debuging and verbose

# NOTICE: directories paths MUST NOT be ended by SLASH (/)
read -r -d '' PROJECT_LIST <<EOL
/home/user/project1
/home/user/project2
EOL

mkdir -p $BACKUP_DIR

while IFS= read -r PROJECT_DIR; do
    echo "Backup of $PROJECT_DIR started"
    if [ -e "$PROJECT_DIR"/.git ]; then
        git --git-dir=$PROJECT_DIR/.git --work-tree=$PROJECT_DIR ls-files -i -o --directory --exclude-standard | rsync $RSYNC_OPTIONS --delete --exclude-from - $PROJECT_DIR $BACKUP_DIR
    else
        rsync $RSYNC_OPTIONS --delete $PROJECT_DIR $BACKUP_DIR
    fi
    echo "Backup of $PROJECT_DIR started ended with code $?"
done <<< "$PROJECT_LIST"
```

Проверяем, всё работает.