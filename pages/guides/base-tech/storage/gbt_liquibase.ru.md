---
title: Liquibase
keywords: liquibase, cli, docker, sql, postgres, postgresql, mssql, flexberry enterprise
sidebar: guide-base-tech_sidebar
toc: true
permalink: ru/gbt_liquibase.html
lang: ru
---

## Описание

Liquibase - это библиотека с открытым исходным кодом для отслеживания, управления и применения изменений схемы базы данных. Является аналогом Git для баз данных. Вместо "коммитов" - changesets, представляющие наборы SQL команд на изменение БД.

Использование Liquibase в проекте позволяет отслеживать изменения схем БД, откатывать (rollback) базы до нужного состояния, облегчает развертывание скриптов на разных серверах.

> [Flexberry Designer](fd_flexberry-designer.html) поддерживает генерацию скриптов Liquibase для СУБД `PostgreSQL` и `Microsoft SQL Server`.

### Установка Liquibase

Liquibase позволяет запускать команды через собственный CLI. В качестве альтернативы, можно запускать CLI в виде Docker контейнера.
- [Скачать Liquibase CLI](https://www.liquibase.com/download).
- [Использовать Liquibase в Docker](https://docs.liquibase.com/workflows/liquibase-community/using-liquibase-and-docker.html)

### Конфигурация Liquibase на проекте

1. Инициализировать Liquibase в папке `SQL` проекта: `liquibase init project` (следовать инструкциям на экране). При создании проекта требуется указать адрес БД в формате JDBC URL (например, `jdbc:postgresql://localhost:5432/my-app-db`) и данные для подключения (логин и пароль).
> Настройки сохраняются в файл `liquibase.properties` - настройки в этом файле применяются по умолчанию. Этот файл может быть использован, чтобы задать подключение или схему, на которой будут запускаться скрипты. Более детально см. [официальную документацию.](https://docs.liquibase.com/concepts/connections/creating-config-properties.html)
2. Добавить ссылки на папки со скриптами, которые необходимо отслеживать Liquibase:
    1. В файле `liquibase.properties` указать название корневого changelog: `changeLogFile=liquibase.json`
    2. В корневой changelog `liquibase.json` добавить ссылки на отслеживаемые папки:

    ```json
    {
        "databaseChangeLog": [
            {
                "includeAll": {
                    "path": "относительный/путь/к/папкам/со/скриптами"
                }
            },
            {
                "includeAll": //...
            }
        ]
    }
    ```

3. Если Liquibase устанавливается в существующее приложение и имеет скрипты, примененные вручную, необходимо отметить эти скрипты как выполненные:
   - `liquibase changelog-sync` отметит существующие скрипты в папках как "применённые", сохранив информацию о них в таблицу `databasechangelog`; после этого командой `update` будут применяться только новые скрипты

#### Рекомендации

Рекомендуется для liquibase указывать отдельную схему для хранения таблиц `databasechangelog` и `databasechangeloglock`:
1. Указать название схемы `liquibase` для хранения внутренних таблиц Liquibase. В файл `liquibase.properties` добавить строчку:
`liquibase.liquibaseSchemaName=liquibase`
2. Создать схему `liquibase` в БД (если скрипты будут применяться на разных БД, необходимо создать схему в каждой БД):

```sql
CREATE SCHEMA liquibase AUTHORIZATION postgres;
```

### Создание скриптов

Для описания изменений в Liquibase используются текстовые файлы [changelog](https://docs.liquibase.com/concepts/changelogs/home.html). Файл содержит SQL-изменения базы данных в специальном liquibase-формате (xml, yaml, json, sql - на выбор). Рекомендуется использовать [liquibase formatted sql](https://docs.liquibase.com/concepts/changelogs/sql-format.html), т.к. именно его генерирует Flexberry Designer.

Скрипты для Liquibase можно генерировать, используя `Flexberry Designer`:
1. [Desktop версия](https://flexberry.github.io/ru/fd_flexberry-designer.html)
  - Выбрать стадию -> Кликнуть правой кнопкой мыши -> `ORM` -> `SQL` -> `PostgreSQL` -> `Сгенерировать SQL для liquibase`

2. [Flexberry Designer Enterprise](https://flexberry.github.io/ru/gpg_practical-guide.html)
  - Перейти в раздел `Генерация` -> для нужного типа хранилища отметить `"Сгенерировать SQL для liquibase"`:
![Настройка через интерфейс](/images/pages/guides/base-technologies/storage/liquibaseExample.jpg)
  - Нажать `"Скачать fdg файл"`. Данный файл будет иметь название `GenConfig.fdg`, в нём появится опция `"LiquibaseSQL": true` в разделе, который соответствует выбранной СУБД (в данном примере - `PostgreSQL`). Если эта настройка в файле отсутствует, необходимо добавить её вручную:

  ```json
  "Storage": {
      "PostgreSql": {
          "LiquibaseSql": true,
          ...
      }
  }
  ```

  > Подробнее в [практическом руководстве](../../practical-guides/flexberry-designer-enterprise/gpg_practical-guide.md)

Есть возможность написать скрипт самостоятельно. Для этого необходимо в начало обычного `.sql` скрипта добавить следующие строчки:
- `--liquibase formatted sql`
- `--changeset author:id` - где `author` - автор изменений, `id` - уникальный идентификатор набора изменений _(рекомендуется использовать текущую дату и время)_.

Пример:
```sql
--liquibase formatted sql
--changeset aivanov:2023-01-24-11:30
CREATE TABLE example...

--changeset aivanov:2023-01-24-11:45
INSERT INTO example VALUES ...
```

### Запуск команд Liquibase

После того, как конфигурация проекта Liquibase завершена, можно запускать команды. Команды запускаются с помощью CLI, который устанавливается вместе с Liquibase.

Команды можно запускать следующим образом:

```sh
liquibase update --log-level INFO
```

> Команды необходимо запускать из папки, в которой был инициализирован liquibase (папка `SQL`).

Команда `liquibase update` применит изменения, на которые есть ссылка в корневом changelog. Список полезных команд:
- `update` - применить все скрипты
- `update-sql` - предпросмотр SQL для команды `update` _(не применяет скрипты)_
- `status` - отобразить количество ещё не запущенных скриптов
- `validate` - проверить корректность скриптов
> [Полный список команд](https://docs.liquibase.com/commands/home.html)

### Запуск команд Liquibase через Docker

Есть возможность запускать команды Liquibase через Docker - т.е. без установки Liquibase CLI. Для этого необходимо использовать следующую команду:

```sh
docker run --rm -v ${PWD}/:/liquibase/changelog/ liquibase/liquibase --defaultsFile=/liquibase/changelog/liquibase.properties --changelog-file=liquibase.json --search-path=/liquibase/changelog/
```

Описание команды:
- `--rm` - удалить контейнер при завершении
- `-v ${PWD}/:/liquibase/changelog/` - монтировать текущую директорию в папку `/liquibase/changelog/` внутри образа
- `liquibase/liquibase` - использовать официальный образ Liquibase CLI
- `--defaultsFile=/liquibase/changelog/liquibase.properties` - путь до файла конфигурации `liquibase.properties`
- `--changelog-file=liquibase.json` - путь до корневого changelog
- `--search-path=/liquibase/changelog/` - путь, в котором будет производиться поиск changelogs

> Если в контейнере папка `/liquibase/changelog/` становится пустой, возможно команда запущена из Windows через Git Bash. В таком случае, перед запуском необходимо задать ENV переменную **MSYS_NO_PATHCONV=1**. Переменную можно записать в `system path` чтобы она не сбрасывалась.

Подробнее см. [Использование Liquibase в Docker.](https://docs.liquibase.com/workflows/liquibase-community/using-liquibase-and-docker.html)

### Откат изменений

Liquibase позволяет откатывать изменения. Это можно сделать с помощью команд:
- `rollback`
- `rollback-to-date`
- `rollback-count`
> (см. [rollback](https://docs.liquibase.com/commands/home.html#database-rollback-commands)).

{% include important.html content="'Из коробки' откат изменений работать не будет. Для каждого скрипта необходимо указывать команды для отката (см. [rollback actions](https://docs.liquibase.com/concepts/changelogs/sql-format.html#rollback-actions)). Flexberry Designer не генерирует таких команд." %}

### Использование скриптов в нескольких БД

В приложении может потребоваться использовать несколько баз данных или развёртывать скрипты на разных БД (или разных схемах одной БД). Для этого можно воспользоваться контекстами в Liquibase.

*Контекст* - это группа, в которую помещаются changelogs для того, чтобы фильтровать скрипты по контексту и применять только нужные изменения ([подробнее](https://docs.liquibase.com/concepts/changelogs/attributes/contexts.html)). Можно применить скрипты из контекста на определённой БД.

- Каждый скрипт из проекта может быть отнесен к своему контексту.

Чтобы отнести файл/папку к контексту, необходимо добавить параметр `contextFilter` в changelog:

```json
"includeAll": {
    "path": "путь/к/папке/со/скриптами",
    "contextFilter": "название_контекста"
}
```

Параметр задаёт, к какому контексту отнести нужный файл/папку.

- Скрипты из контекста можно применить на другой БД и другой схеме.

Следующие параметры могут быть использованы, чтобы применить скрипты на нужной БД:
  - `--contexts контекст` - запустить команду `liquibase` только по скриптам из указанного контекста;
  - `--url=jdbc:postgresql://адрес:порт/название_бд` - применить скрипты на указанной БД;
  - `--default-schema-name схема` - применить скрипты на указанной схеме БД.

> **Пример.** Приложение имеет 2 БД - `core` (основную) и `storm` (для хранения полномочий и настроек). Чтобы применять скрипты на двух базах, необходимо:
> 1. создать 2 папки:
>  - `/SQL/app` - для скриптов основного приложения;
>  - `/SQL/storm` - для скриптов полномочий.
>2. отнести папки к своему контексту (см. раздел *использование нескольких БД*):
>  - `/SQL/app` - к контексту `core`,
>  - `/SQL/storm` - к контексту `storm`.
>3. запустить команду `liquibase update --contexts storm --url=jdbc:postgresql://localhost:5432/storm` - это применит скрипты полномочий на БД `storm`
>4. запустить команду `liquibase update --contexts >core --url=jdbc:postgresql://localhost:5432/app` - это применит скрипты приложения на БД `core`.

Чтобы облегчить процесс обновления, можно написать `cmd` или `sh` скрипт, который будет подставлять нужный url БД в зависимости от переданного контекста. Вызывать его, например, так: `./liquibase.sh update storm` - без необходимости прописывать url вручную. Пример такого скрипта:

```sh
#!/bin/bash

# Вспомогательный скрипт для запуска команд liquibase.
# Автоматически подставляет url нужной базы в зависимости от контекста.
#
# Использование:
# ./liquibase.sh <команда_liquibase> <название контекста (базы)>, например: ./liquibase.sh status storm
# Запуск в режиме Docker: добавьте опцию --docker
# P.S. Если <название контекста> не передано - скрипт будет применён на всех контекстах (базах) по очереди.

script_dir="$(cd "$(dirname "$0")" && pwd)"
env_file="$script_dir/liquibase.env"

# Загрузим ENV переменные из файла, которые ещё не были установлены:
source liquibase.env

contexts=(core storm) # доступные контексты
contextArgs=()

# Проверка, является ли переданное значение одним из доступных контекстов
isContext() {
  if [[ " ${contexts[@]} " =~ " ${1} " ]]; then
    return 1
  else
    return 0
  fi;
}

# Проверка наличия опции в списке аргументов
check_option() {
  local option=$1
  shift

  for arg in "$@"; do
    if [[ "$arg" == "$option" ]]; then
      return 1  # Опция найдена
    fi
  done

  return 0
}

# Проверяем наличие опции --docker
check_option "--docker" "$@"
dockerMode=$?

filtered_arguments=()

# Обрабатываем переданные контексты
for arg in "$@"; do
  isContext "$arg"
  if [ $? = 1 ]; then
    contextArgs+=("$arg")
    elif [ $arg != '--docker' ]; then
    filtered_arguments+=("$arg")
  fi
done

# Если контексты не переданы, используем все доступные
if [ ${#contextArgs[@]} -eq 0 ]; then
  contextArgs=("${contexts[@]}")
fi

for context in "${contextArgs[@]}"; do

  # Переключаем в режим docker если передан параметр --docker:
  if [ "$dockerMode" = 1 ]; then
    export MSYS_NO_PATHCONV=1 # нужен чтобы пофиксить запуск через Git Bash на Windows
    liquibaseCmd="docker run --rm -v ${PWD}/:/liquibase/changelog/ liquibase/liquibase --defaultsFile=/liquibase/changelog/liquibase.properties --changelog-file=liquibase.json --search-path=/liquibase/changelog/"
  else
    liquibaseCmd="liquibase"
  fi

  # Задаём БД в зависимости от контекста:
  if [ "$context" = "storm" ]; then
    database=$STORM_JDBC_URL
    elif [ "$context" = "core" ]; then
    database=$CORE_JDBC_URL
  fi;

  echo -e "Запускаем $1 на БД $context... \n"

  # Запускаем команду:
  echo "$liquibaseCmd $1 --contexts $context --url=$database ${filtered_arguments[@]:1}"
  $liquibaseCmd $1 --contexts $context --url=$database ${filtered_arguments[@]:1}

  # Завершаем выполнение на первом исключении:
  if [ $? = 1 ]; then
    exit 1
  fi

  echo ""
done

exit 0
```

Для работы скрипта необходим файл liquibase.env:

```env
CORE_JDBC_URL=jdbc:postgresql://localhost:5432/app
STORM_JDBC_URL=jdbc:postgresql://localhost:25432/storm
```

## Дополнительно

* [Использование Liquibase в Azure Pipelines](./gbt_liquibase_azure.ru) <i class="fa fa-arrow-up" aria-hidden="true"></i>

## Ресурсы

* [Официальная документация Liquibase](https://docs.liquibase.com/start/home.html)
