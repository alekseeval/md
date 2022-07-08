
# Содержание

### Тестовый модуль _cf-test_

- Требования для запуска тестов:
  - [Опции конфига config.yaml](#cfg_option)
  - [Инициализация виртуального окружения venv](#venv)
- Способы запуска тестов:
  - [Использование cf-test.sh](#cf-test)
  - [Использование pytest. Запуск из виртуального окружения](#run-venv)
- [Группировка тестовых сценариев](#groups)
- Архитектура проекта:
  - [Тестовые сценарии](#test-scenarios)
    - [Добавление новых тестов](#new-test)
    - [Рекомендации по написанию новых тестов](#tests-recomenations)
  - [Фикстуры](#fixtures)
  - [Методы выполнения запросов к API](#requests)
  - [Вспомогательные скрипты](#scripts)

### GitLab CI/CD
- [Настройка GitLab-Runner](#ci-cd-settings)
- [Условия запуска CI/CD пайплайна](#ci-cd-conditions)
- [Этапы выполнения CI/CD пайплайна](#ci-cd-steps)

-----------------------------------------------------------------------


# Тестовый модуль *cf-test*

## <a name="cfg_option">Требования для запуска тестов</a>

Настройка необходимых параметров в config.yaml:

| Name              | Description                                                                                                                                  |
|-------------------|----------------------------------------------------------------------------------------------------------------------------------------------|
| `CYFOR_DUMP_PATH` | Полный путь до файла дампа **cyfor** (значение, установленное по умолчанию, соответствует расположению на сервере **_dev4_**)                |
| `PM_DUMP_PATH`    | Полный путь до файла дампа **pm** (значение, установленное по умолчанию, соответствует расположению на сервере **_dev4_**)                   |
| `request_timeout` | Timeout http-запросов к API во время исполнения тестов (в секундах, целое число)                                                             |
| `url`             | Адрес по которому выполняются запросы к API по http (значение по умолчанию -- _https://127.0.0.1:1500_)                                      |
| `op_date`         | Дата в формате yyyy-mm-dd. Все операции в системе, выполняемые во время тестирования, будут выполняться на указанную в этом параметре дату.  |

Все параметры из списка имеют установленные значения по умолчанию.

При использовании на локальной машине, в первую очередь следует убедиться в корректности параметра `url`.

При выполнении тестов используются фикстуры завязанные на данные из тестового бэкапа.
В связи с этим необходимо убедиться в корректности параметров `CYFOR_DUMP_PATH` и `PM_DUMP_PATH`.
Дампы с тестовыми данными можно установить используя скрипт clean_db.sh. <a name="clean-db-script">Скрипт описан ниже</a>.

### <a name="venv">Инициализация виртуального окружения venv</a>
Запуск тестов осуществляется из-под виртуального окружения python3-venv.
Окружение venv должно находиться в корне проекта с установленными необходимыми библиотеками, указанными в requirements.txt

Чтобы создать и настроить виртуальное окружение необходимо запустить скрипт `init_venv.sh`. <a name="clean-db-script">Скрипт описан ниже</a>.

## Запуск тестов
### <a name="cf-test">Использование cf-test.sh</a>
Предпочтительным способом запуска тестов является использование утилиты `cf-test.sh`, написанной для этой цели (лежит в корне проекта).

При запуске утилиты без использования дополнительных параметров будет выполнена следующая последовательность действий:
1) Выполнение команды git pull для git-репозитория cyfor в текущей активной ветке
2) Пересборка панелей cyfor и pm в случае, если при выполнении первого шага были получены изменения
3) Выполнение всех доступных тестовых сценариев

Кроме того, утилита может быть запущенна со следующими опциональными параметрами:

| Name                       | Description                                                                                                                                                                                                                                                                                        |
|:---------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `-b`                       | Пересобирает панели cyfor и pm перед запуском тестов даже если в текущей ветке проекта изменений не было.                                                                                                                                                                                          |
| `--no-build`               | Не пересобирает                                                                                                                                                                                                                                                                                    |
| `-p <path>`                | Запускает все тесты по переданному пути относительно проекта cf-test. Может быть использовано для запуска одиночных тестов с указанием полного пути до функции теста.<br/> Путь указывается в формате:<br/> `-p "package_tests/test_mod.py::TestClass::test_method"`                               |
| `-m <marker_expression>`   | Запускает все тесты, входящие в состав одной или нескольких маркированных групп. Может быть передано как имя одной единственной маркированной группы, так и регулярное выражение в формате:<br/> `-m "group1 or group2 and not group3"` <br/> Все доступные группы тестов описаны [ниже](#groups). |
| `-k <keyword_expression>`  | Запускает все тесты, полный путь до которых содержит в себе переданное ключевое слово или удовлетворяет переданному регулярному выражению из ключевых слов. Например:<br/> `-k "MyClass and not method"`                                                                                           |
| `--clean-db`               | Очищает БД _cyfor_ и _pm_, после чего восстанавливает из файлов бэкапа                                                                                                                                                                                                                             |
| ~~~~~~~~~~~~~~~~~~~~~~~~~~ |                                                                                                                                                                                                                                                                                                    |

Все параметры являются опциональными и могут быть переданы в любом порядке.

### <a name="run-venv">Запуск из своего виртуального окружения, через pytest</a>
Утилита `cf-test.sh` во время работы использует python-библиотеку `pytest`, поэтому существует возможность запуска тестов напрямую посредством `pytest`. Для этого необходимо:
1) Активировать виртуальное окружение командой: `source ".../cf-test/venv/bin/activate"`
2) Перейти в директорию `.../cf-test/tests`. Важно запускать тесты именно из этой директории, чтобы `pytest` мог распознать файл pytest.ini
3) Запускать тесты используя команду `pytest <options...>` ([подробнее можно найти в документации pytest](https://docs.pytest.org/en/latest/how-to/usage.html#invoke-other))
4) Деактивировать виртуальное окружение командой `deactivate`

## <a name="groups"> Группировка тестовых сценариев</a>

Все доступные группы тестов:


| Name         | Description                                                                                                                        |
|--------------|------------------------------------------------------------------------------------------------------------------------------------|
| `connection` | Тесты, проверяющие соединение со всем панелями (core/cyfor/pm)                                                                     |
| `validators` | Тесты, проверяющие корректность работы валидации у полей форм                                                                      |
| `toctou`     | Тесты, проверяющие наличие необходимых блокировок у сущностей ([wiki](https://en.wikipedia.org/wiki/Time-of-check_to_time-of-use)) |
| `inventory`  | Тесты, проверяющие корректность работы документов инвентаризации (все из раздела "инвентаризация")                                 |
| `stockin`    | Тесты, проверяющие корректность работы актов приходования (подгруппа `inventory`)                                                  |
| `move_task`  | Тесты, проверяющие корректность работы заданий на перемещения в смене                                                              |


## Архитектура проекта

### <a name="test-scenarios">Тестовые сценарии</a>
Все тестовые сценарии расположены в "tests/" и в соответствущим их назначению подкаталогам:
- **connection_tests/** - каталог с тестами, предназначенными для проверки соединения с сервером. Изначально
предполагалось, что при падении этих сценариев, остальные сценарии даже не будут запускаться, но от этой идеи
пришлось отказаться в связи с неочевидностью реализации.
- **documents_tests/** - каталог с тестами, предназначенными для проверки корректности работы основных
документов в системе. Например - актов приходования/списания, документов смены и заданий на перемещение
- **toctou_tests/** - каталог с тестами, предназначенными для выполнения проверок корректности одновременной обработки
некоторых запросов, которые потенциально могут что-либо сломать ([wiki](https://en.wikipedia.org/wiki/Time-of-check_to_time-of-use))
- **validators_tests/** - каталог с тестами, предназначенными для проверки корректности работы валидаторов на формах.

#### <a name="new-test"> Для добавления новых тестовых сценариев в систему необходимо:</a>
1. Следовать правилу наименования файлов-классов-функций, описанному в документации к [Pytest](https://docs.pytest.org/en/7.1.x/contents.html)
2. Добавлять новые тестовые сценарии в каталог **tests/***

В целом, это все ограничения, накладываемые на написание новых тестов. Доработав проект можно избавиться от
ограничения под пунктом **1**.

#### <a name="tests-recomenations">Рекомендации по написанию новых сценариев:</a>
1. Использовать базовые фикстуры пользовательских сессий, такие как _tester_session_, _user_admin_session_pm_, _requests_session_.
Более подробно о них написано в разделе о фикстурах.
2. Маркировать тесты с использованием аннотации _@pytest.mark.*_. Список собственных маркеров в системе находится по пути
**tests/pytest.ini**.

### <a name="fixtures">Фикстуры</a>

Фикстуры проекта расположены в нескольких местах, относительно их прямого назначения.

В файле "**cf-test/resources/fixtures.py**" расположены фикстуры, предназначенные для формирования новых сущностей
в системе. Этих сущностей еще не существует и назначение фикстур состоит в том, чтобы их более удобным образом создать.

В файле "**cf-test/resources/db_fixtures.py**" расположены фикстуры с данными, которые уже занесены в тестовую БД.

Файл "**cf-test/tests/conftest.py**" связан с библиотекой Pytest и про него стоит почитать в документации.
В проекте он используется для агрегации фикстур из файлов, описанных выше. Кроме того в этом файле
инициализируются некоторые другие фикстуры, которые используются практически везде в коде:

- Фикстуры _tester_session_, _user_admin_session_pm_ - это фикстуры содержащие в себе сессию соответствующих
пользователей в системе, которые создаются перед стартом всех тестовых сценариев и автоматически удаляются после.
Это позволяет не создавать пользовательскую сессию на каждый тест отдельно.
- Фикстура _requests_session_ - это фикстура содержащая объект Session из библиотеки Requests.
Рекомендуется при выполнении запросов использовать именно эту сессию, а не Requests.get/post() напрямую.
Это позволяет переиспользовать открытое соединение с сервером и выполнять запросы в разы быстрее.

### <a name="requests">Методы выполнения запросов к API</a>

Все вспомогательные методы для взаимодействия с киберлесом и выполнения API-запросов расположены по пути 
**/cf-test/src/***. Писать новые методы взаимодействия с API киберлеса рекомендуется в этом каталоге.

Большая часть методов для взаимодействия с API возвращают в качестве результата выполнения запроса объект 
класса ApiResponse. Исходный код класса расположен в **/cf-test/resources/model.py**.

### <a name="scripts">Вспомогательные скрипты</a>

Для более удобного взаимодействия с киберлесом в CI/CD пайплайне были написаны небольшие вспомогательные скрипты на bash.
Они расположены по пути "**/cf-test/scripts/***".

Список скриптов:
- **build_cyfor.sh** - скрипт, выполняющий сборку киберлеса. Принимает обязательный параметр "-p PATH",
указывающий на расположение репозитория киберлеса с исходниками.
- **clean_db.sh** - скрипт, выполняющий сброс базы данных с последующим развертыванием дампа тестовой БД. 
Требует установленных параметров `CYFOR_DUMP_PATH` и `PM_DUMP_PATH` в <a name="cfg_option">конфиге</a>.
- **init_venv.sh** - скрипт, который разворачивает виртуальное окружение venv с библиотеками из requirements.txt на директорию выше своего расположения. 
Требует установленного Python3 и pip.
- **start_replication.sh** и **stop_replication.sh**  - скрипты выключающие и включающие репликацию, соответственно.

--------

# GitLab CI/CD

## <a name="ci-cd-settings">Настройки для корректной работы CI/CD Pipeline</a>

### Установка gitlab-runner

Данный раздел полезен, если нужно создать нового раннера на другом сервере, для запуска пайплайнов.

[Guide по установке на сервер](https://sean-bradley.medium.com/installing-gitlab-runner-on-ubuntu-and-centos-80f3a2de0290)

Примечания:
- Токен инициализации лежит в настройках CI/CD gitlab
- Необходимо указать тэг 'tester' во время регистрации раннера

### Права доступа для gitlab-runner пользователя
На данный момент многие команды запускаются gitlab-runner'ом через SUDO, поэтому необходимо выдать права на использование SUDO без пароля.

### Другие настройки сервера
У пользователя postgres должны быть права на чтение файлов `CYFOR_DUMP_PATH` и `PM_DUMP_PATH`. (параметры [config.yaml](#config))

Установить глобальные настройки пильного модуля, если еще не установлены (делается через конфиг файл, расположенный в /usr/local/mgr5/etc/pm.conf)

## <a name="ci-cd-conditions">Условия запуска CI/CD пайплайна</a>

Пайплайн запускается при возникновении следующих событий, возникающих в репзитории:
- Создание Merge Request'а
- Push коммитов в существующий Merge Request
- Любой Push в master ветку

Все эти события вызывают один неизменный пайплайн, который отличается только этапом `Update backups`,
запускаемом в случае работы с master веткой 

## <a name="ci-cd-steps">Этапы выполнения CI/CD пайплайнов</a>

Далее представлены этапы выполнения пайплайна в хронологическом порядке:

### 1. Create Python venv
Этап, в ходе которого происходит подготовка виртуального окружения venv. Внутри этапа запускается скрипт <a name="init-venv-script">init_venv.sh</a>
Виртуальное окружение кэшируется.

### 2. Restore DB
Этап в ходе которого база данных восстанавливается из дампа, расположение которого берется из <a name="cfg_option">конфига</a>. Внутри этапа запускается скрипт <a name="clean-db-script">clean_db.sh</a>

### 3. Build cyfor
На данном этапе выполняется сборка проекта. Внутри этапа репозиторий полностью копируется в директорию
`/usr/local/mgr5/src/cyfor_test/` и оттуда собирается. Связано такое решение с условностями сборки проекта,
требующими фиксированного расположения в определенной директории и с тем, что _gitlab-runner_ может клонировать
репозиторий только в свое выделенное пространство, за пределы которого нельзя выйти. Кроме того, в конфиг киберлеса
добавляется опция пересоздания представлений (_CreateViews_) и запускается репликация по окончании сборки.
По итогу выполнения этапа имеем полностью работающий проект.

### 3.5 Update backups (выполняется не всегда)
Данный этап выполняется только при работе с веткой master. Его назначение -- обновить дампы БД под актуальную для
репозитория проекта структуру. Внутри выполняется скрипт <a name="update-backups-script">update_backups.sh</a>

### 4. Run Integration Tests
На данном этапе выполняется запуск тестовых сценариев посредством исполнения скрипта <a name="cf-test">cf-test.sh</a>. 
Результат исполнения тестов сохраняется в качестве артефакта в формате xml и автоматически обрабатывается GitLab-ом.
