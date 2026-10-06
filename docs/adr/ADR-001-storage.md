# ADR-001: Каноническая модель хранения сущности Project на базе PostgreSQL и Git-репозитория

- **Статус:** Proposed
- **Дата:** 2026-10-06
- **Контекст:** Эпик 2.5 (пакет v3), требования заказчика по хранению артефактов в Git, интеграция с направлениями 2.1 и 2.3 (PPS-141)
- **Целевой артефакт:** docs/adr/ADR-001-storage.md
- **Зависимости:** Gateway, Runner, Auth/ownership

---

## 1. Контекст и проблематика

В рамках разработки веб-направления CoreTeam необходимо формализовать схему хранения сущности Project и связанных с ней подсущностей.

Система взаимодействует со следующими компонентами:
* Gateway: точка входа клиентских запросов, валидация прав, оркестрация сессий.
* Auth/ownership: сервис авторизации и разграничения прав доступа к сущностям и репозиториям.
* Runner: исполняющая среда пайплайнов, порождающая события и артефакты.

### Требования и ограничения:
1. Git-as-Storage для результатов: По требованию заказчика для хранения сгенерированных отчетов, результатов работы команды (Markdown-артефактов) и сопутствующих манифестов вместо объектного S3-хранилища используется Git-репозиторий проекта.
2. Версионирование коммитами: Каждая подтвержденная версия артефакта фиксируется уникальным commit_sha.
3. Разделение контуров: Внутренние рассуждения моделей, промежуточные токены и секреты не попадают в Git и не транслируются в публичные события.
4. Консистентность статусов: Статус completed выставляется только после успешного git commit / git push и фиксации события artifact_saved.
5. Метаданные и события в PostgreSQL: Метаданные проектов, права доступа, сессии пользователей и упорядоченный журнал публичных событий хранятся в реляционной БД.

---

## 2. Разделение ролей хранения (Storage Roles)

Для исключения конфликтов обязанностей фиксируются роли компонентов хранения:

* PostgreSQL: Источник истины (Source of Truth) для реляционных связей, метаданных сущностей, статусов запусков, индексации и упорядоченного журнала событий. Структурированные метаданные, транзакционные обновления, быстрые выборки по фильтрам.
* Git-репозиторий: Версионированное хранилище отчетов, декларативных материалов и служебных конфигураций CoreTeam. Markdown-документы (result.md), манифесты (.ctf/*.json), версионируемые входные спецификации.
* Object Storage (S3): Выведено из архитектуры v1 по требованию заказчика. Резервируется в бэклоге архитектуры как внешний слой для тяжелых бинарных данных (>50–100 МБ) через интеграцию с Git LFS или внешний bucket.

---

## 3. Архитектурное решение (Decision)

### 3.1. Структура каталогов в Git-репозитории

repo-root/
└── projects/
    └── {project_id}/
        ├── materials/              # Входные файлы пользователя
        │   └── dataset_spec.pdf
        ├── .ctf/                   # Служебные файлы фреймворка (манифесты, роли)
        │   ├── team_manifest.json
        │   └── framework_state.json
        └── runs/
            └── {run_id}/
                └── result.md       # Итоговый подтвержденный артефакт

### 3.2. Спецификация моделей данных (PostgreSQL)

1. Projects (projects)
* id (UUID, Primary Key)
* owner_id (UUID, Foreign Key)
* title (VARCHAR(255))
* description (TEXT)
* status (VARCHAR(32)): draft | team_proposed | active | archived
* git_repo_url (VARCHAR(512)) — ссылка на рабочий репозиторий
* git_branch (VARCHAR(128), default: 'main') — ветка проекта
* context_data (JSONB) — цели, входные требования, история ответов фасилитатору
* schema_version (INT, default: 1) — счетчик оптимистической блокировки
* created_at, updated_at, deleted_at (TIMESTAMPTZ)

2. Materials (materials)
* id (UUID, Primary Key)
* project_id (UUID, Foreign Key)
* filename (VARCHAR(255))
* mime_type (VARCHAR(128))
* size_bytes (BIGINT)
* git_path (VARCHAR(512)) — путь внутри репозитория (projects/{project_id}/materials/...)
* commit_sha (VARCHAR(40)) — коммит добавления материала
* created_at (TIMESTAMPTZ)

3. Runs (runs)
* id (UUID, Primary Key)
* project_id (UUID, Foreign Key)
* status (VARCHAR(32)): queued | running | waiting_for_input | partial | completed | failed | cancelled | limit_reached | unknown
* current_sequence (INT, default: 0)
* last_artifact_id (UUID, Foreign Key к artifacts, nullable)
* last_heartbeat_at (TIMESTAMPTZ, nullable) — маркер активности воркера Runner
* started_at, finished_at, created_at (TIMESTAMPTZ)

4. Events (events)
* id (UUID, Primary Key)
* run_id (UUID, Foreign Key)
* project_id (UUID, Foreign Key)
* sequence (INT, составной уникальный индекс UNIQUE(run_id, sequence))
* type (VARCHAR(64)) — публичный тип (role_progress, artifact_saved и др.)
* actor (JSONB) — роль CoreTeam (roleId, displayName)
* payload (JSONB) — нормализованные данные шага (прогресс, текст, ETA)
* created_at (TIMESTAMPTZ)

5. Artifacts (artifacts)
* id (UUID, Primary Key)
* project_id (UUID, Foreign Key)
* run_id (UUID, Foreign Key)
* title (VARCHAR(255))
* mime_type (VARCHAR(64), default: text/markdown)
* git_path (VARCHAR(512)) — путь к файлу (projects/{project_id}/runs/{run_id}/result.md)
* commit_sha (VARCHAR(40)) — хеш коммита Git с зафиксированным результатом
* git_tree_url (VARCHAR(512)) — URL для просмотра коммита/файла в веб-интерфейсе Git
* size_bytes (BIGINT)
* created_at (TIMESTAMPTZ)

6. CTF Files (ctf_files)
* id (UUID, Primary Key)
* project_id (UUID, Foreign Key)
* run_id (UUID, Foreign Key, nullable)
* kind (VARCHAR(64)): team_manifest | role_definition | framework_state
* git_path (VARCHAR(512)) — путь в скрытом каталоге (projects/{project_id}/.ctf/...)
* commit_sha (VARCHAR(40))
* is_internal (BOOLEAN, default: true) — изоляция от публичного API
* created_at (TIMESTAMPTZ)

---

## 4. Семантика записи (Write Semantics)

### 4.1. Сценарий Confirmed Write (Двухфазное подтверждение)
1. Runner формирует результирующий Markdown-файл в изолированном локальном каталоге.
2. Runner выполняет коммит и пуш файла по пути projects/{project_id}/runs/{run_id}/result.md в удаленный репозиторий. Фиксируется commit_sha.
3. Gateway открывает транзакцию в PostgreSQL:
   * Создается запись в таблице artifacts с привязкой к commit_sha.
   * Статус в таблице runs обновляется на completed.
   * В журнал events вставляется событие с типом artifact_saved.
4. Gateway возвращает клиенту код 200 OK.

### 4.2. Обработка Unknown Outcome (Сетевые сбои и таймауты)
* Путь артефакта детерминирован идентификаторами (projects/{project_id}/runs/{run_id}/result.md), что исключает появление дубликатов.
* При сбое Runner опрашивает Git: если коммит уже существует, повторная генерация пропускается, и система сразу повторяет фиксацию статуса в PostgreSQL.
* Коммиты без подтверждённой транзакции в базе данных по истечении 24 часов вычищаются фоновым аудитом.

---

## 5. Управление жизненным циклом и целостностью

### 5.1. Version Conflict (Разрешение конфликтов версий)
* На уровне Git конфликты исключены за счет изоляции каталогов под каждый run_id (projects/{project_id}/runs/{run_id}/).
* На уровне PostgreSQL применяется оптимистическая блокировка через поле schema_version (при параллельной записи возвращается ошибка 409 Conflict).

### 5.2. Recovery (Восстановление после сбоев)
* Во время работы Runner каждые 30 секунд обновляет поле last_heartbeat_at.
* Фоновый Reaper раз в минуту находит задачи со статусом running без обновлений более 5 минут и переводит их в failed с кодом TIMEOUT.

### 5.3. Delete & Retention (Удаление и политики хранения)
* Soft Delete: Заполняется projects.deleted_at, статус меняется на archived, данные скрываются из API на 30 дней.
* Hard Delete: Через 30 дней фоновый процесс выполняет git rm -rf projects/{project_id} и каскадно очищает строки в PostgreSQL.
* Retention: Логи events архивируются/удаляются через 90 дней, метаданные проектов и итоговые артефакты хранятся бессрочно.

---

## 6. Анализ альтернатив и архитектурный компромисс

1. Почему не только Git: Прямое сохранение тяжелых файлов (PDF, датасеты) в Git раздувает репозиторий, замедляя git clone воркеров, а полное удаление (Hard Delete) требует опасной пересборки всей истории коммитов.
2. Компромисс:
   * Этап 1 (v1 — Epic 2.5): Работа только через PostgreSQL + чистый Git для Markdown-отчетов и ограничение на размер входных файлов до 10 МБ.
   * Этап 2 (Целевой): Подключение Git LFS или Object Storage (S3) для тяжелых материалов, оставляя в Git только текст отчетов и манифесты.
