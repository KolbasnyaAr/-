# ADR-001: Каноническая модель хранения сущности Project на базе PostgreSQL и Git-репозитория

- **Статус:** Proposed
- **Дата:** 2026-10-01
- **Контекст:** Эпик 2.5 (пакет v3), требования заказчика по хранению артефактов в Git, интеграция с направлениями 2.1 и 2.3
- **Целевой артефакт:** `docs/adr/ADR-001-storage.md`

---

## 1. Контекст и проблематика

В рамках разработки веб-направления CoreTeam необходимо формализовать схему хранения сущности `Project` и связанных с ней подсущностей. 

По требованию заказчика для хранения сгенерированных отчетов, результатов работы команды (Markdown-артефактов) и сопутствующих манифестов вместо объектного S3-хранилища должен использоваться **Git-репозиторий**.

### Требования и ограничения:
1. **Git-as-Storage для результатов:** Итоговые подтвержденные артефакты (`text/markdown`) и внутренние конфигурации фреймворка коммитятся в Git-репозиторий проекта.
2. **Версионирование коммитами:** Каждая подтвержденная версия артефакта фиксируется уникальным `commit_sha`.
3. **Разделение контуров:** Внутренние рассуждения моделей, промежуточные токены и секреты не попадают в Git и не транслируются в публичные события.
4. **Консистентность статусов:** Статус `completed` выставляется только после успешного `git commit` / `git push` и фиксации события `artifact_saved`.
5. **Метаданные и события в PostgreSQL:** Метаданные проектов, права доступа, сессии пользователей и упорядоченный журнал публичных событий хранятся в реляционной БД.

---

## 2. Архитектурное решение (Decision)

Принята гибридная схема **PostgreSQL + Git**:
- **PostgreSQL:** хранит профили пользователей, метаданные проектов, состояние текущего запуска (`runs`) и строгий упорядоченный журнал публичных событий (`events`).
- **Git-репозиторий:** выступает версионированным хранилищем входных материалов, сгенерированных отчетов (`artifacts`) и служебных манифестов (`ctf_files`).

### 2.1. Структура каталогов в Git-репозитории

Каждый проект и запуск изолированы в структуре директорий репозитория:

```text
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
```

### 2.2. ER-диаграмма сущностей

```mermaid
erDiagram
    USERS ||--o{ PROJECTS : "owns"
    PROJECTS ||--o{ MATERIALS : "tracks files in git"
    PROJECTS ||--o{ RUNS : "executes"
    PROJECTS ||--o{ CTF_FILES : "contains manifests"

    RUNS ||--o{ EVENTS : "emits"
    RUNS ||--o{ ARTIFACTS : "commits result to git"

    USERS {
        uuid id PK
        string email
        string role
        timestamp created_at
    }

    PROJECTS {
        uuid id PK
        uuid owner_id FK
        string name
        string git_repo_url
        timestamp created_at
    }

    MATERIALS {
        uuid id PK
        uuid project_id FK
        string file_name
        string file_path
        timestamp uploaded_at
    }

    CTF_FILES {
        uuid id PK
        uuid project_id FK
        uuid run_id FK
        string kind
        string git_path
        string commit_sha
        boolean is_internal
        timestamp created_at
    }

    RUNS {
        uuid id PK
        uuid project_id FK
        string status
        timestamp started_at
        timestamp finished_at
    }

    EVENTS {
        uuid id PK
        uuid run_id FK
        string event_type
        string payload
        timestamp created_at
    }

    ARTIFACTS {
        uuid id PK
        uuid run_id FK
        string file_path
        string commit_sha
        timestamp created_at
    }
```
### 2.3. Спецификация подсущностей
1. Projects (projects)
id (UUID, Primary Key)

owner_id (UUID, Foreign Key)

title (VARCHAR(255))

description (TEXT)

status (VARCHAR(32)): draft | team_proposed | active | archived

git_repo_url (VARCHAR(512)) — ссылка на рабочий репозиторий

git_branch (VARCHAR(128), default: 'main') — ветка проекта

context_data (JSONB) — цели, входные требования, история ответов фасилитатору

schema_version (INT, default: 1)

created_at, updated_at (TIMESTAMPTZ)

2. Materials (materials)
id (UUID, Primary Key)

project_id (UUID, Foreign Key)

filename (VARCHAR(255))

mime_type (VARCHAR(128))

size_bytes (BIGINT)

git_path (VARCHAR(512)) — путь внутри репозитория (projects/{project_id}/materials/...)

commit_sha (VARCHAR(40)) — коммит добавления материала

created_at (TIMESTAMPTZ)

3. Runs (runs)
id (UUID, Primary Key)

project_id (UUID, Foreign Key)

status (VARCHAR(32)): queued | running | waiting_for_input | partial | completed | failed | cancelled | limit_reached | unknown

current_sequence (INT, default: 0)

last_artifact_id (UUID, Foreign Key к artifacts, nullable)

started_at, finished_at, created_at (TIMESTAMPTZ)

4. Events (events)
id (UUID, Primary Key)

run_id (UUID, Foreign Key)

project_id (UUID, Foreign Key)

sequence (INT, составной уникальный индекс UNIQUE(run_id, sequence))

type (VARCHAR(64)) — публичный тип (role_progress, artifact_saved и др.)

actor (JSONB) — роль CoreTeam (roleId, displayName)

payload (JSONB) — нормализованные данные шага (прогресс, текст, ETA)

created_at (TIMESTAMPTZ)

5. Artifacts (artifacts)
id (UUID, Primary Key)

project_id (UUID, Foreign Key)

run_id (UUID, Foreign Key)

title (VARCHAR(255))

mime_type (VARCHAR(64), default: text/markdown)

git_path (VARCHAR(512)) — путь к файлу (projects/{project_id}/runs/{run_id}/result.md)

commit_sha (VARCHAR(40)) — хеш коммита Git с зафиксированным результатом

git_tree_url (VARCHAR(512)) — URL для просмотра коммита/файла в веб-интерфейсе Git

size_bytes (BIGINT)

created_at (TIMESTAMPTZ)

6. CTF Files (ctf_files)
id (UUID, Primary Key)

project_id (UUID, Foreign Key)

run_id (UUID, Foreign Key, nullable)

kind (VARCHAR(64)): team_manifest | role_definition | framework_state

git_path (VARCHAR(512)) — путь в скрытом каталоге (projects/{project_id}/.ctf/...)

commit_sha (VARCHAR(40))

is_internal (BOOLEAN, default: true) — изоляция от публичного API интерфейса

created_at (TIMESTAMPTZ)

### 2.4. Типы валидации (TypeScript / Pydantic)
```TypeScript
export interface ArtifactGit {
  artifactId: string;
  projectId: string;
  runId: string;
  title: string;
  mimeType: 'text/markdown';
  gitPath: string;            // projects/{projectId}/runs/{runId}/result.md
  commitSha: string;          // 40-символьный SHA фиксации результата
  gitTreeUrl?: string;        // Ссылка на просмотр в Git
  sizeBytes: number;
  createdAt: string;
}

export interface MaterialGit {
  materialId: string;
  projectId: string;
  filename: string;
  mimeType: string;
  gitPath: string;
  commitSha: string;
  sizeBytes: number;
  createdAt: string;
}
```
---

## 2.5. Матрица ответственности хранилищ (PostgreSQL vs Git vs Object Storage)

| Критерий / Хранилище | PostgreSQL | Git (GitHub / GitLab) | Object Storage (S3 / MinIO) |
| :--- | :--- | :--- | :--- |
| Основная роль | Системный Source of Truth: транзакции, состояние сущностей, блокировки, строгая очередность events. | Версионированное хранение Markdown-артефактов (Artifact), входных файлов (Material) и манифестов фреймворка (CTF). | Secondary / Fallback: хранение тяжелых бинарных материалов (>50–100 МБ), дампов и временных снапшотов. |
| Характер данных | Структурированные метаданные, JSONB-контекст, RBAC. | Текстовые файлы, markdown-отчеты, конфигурации (до 50 МБ). | Крупные датасеты, медиа-файлы, архивы. |
| Гарантии консистентности | Строгая транзакционность (ACID), уникальность составных ключей (run_id, sequence). | Линеаризация истории коммитов на ветку, SHA-хеширование. | Eventual consistency / Read-after-write, ETag-верификация. |
| Статус в архитектуре v1 | Primary (Обязательно) | Primary (Обязательно) | Deferred (Опционально): для v1 не задействован; при превышении лимита файлов Material делегируется через Git LFS / S3-совместимый бэкенд. |

---

## 2.6. Гарантии записи: Confirmed Write и Unknown Outcome

### Модель Confirmed Write
Запись считается подтвержденной (Confirmed Write) и статус Run переходит в completed только при строгом соблюдении цепочки:
1. Runner / Worker генерирует итоговый артефакт.
2. Файл отправлен в Git: выполнен успешный git push и получен commit_sha от апстрима.
3. В PostgreSQL в рамках одной транзакции:
   - Создана запись в artifacts с полученным commit_sha и git_path.
   - Добавлено терминальное событие в events с типом artifact_saved и следующим порядковым sequence.
   - runs.status обновлен на completed, проставлен runs.last_artifact_id.

### Обработка Unknown Outcome (Таймауты и сетевые сбои)
Если сетевой запрос завершился по таймауту (например, git push завис или оборвалось соединение с базой данных):
- Сбой на этапе Git Push: Статус Run переводится в unknown (или running с повторным polling). Воркер выполняет идемпотентную проверку git ls-remote / вызов API GitHub по тегу ветки/коммита.
- Сбой записи в PostgreSQL при успешном коммите в Git: При перезапуске воркер (Reconciliation Worker) проверяет наличие ожидаемого файла по шаблону projects/{project_id}/runs/{run_id}/result.md в Git. Если файл существует, метаданные дозаписываются в PostgreSQL без повторного создания коммита (Deduplication).
- Смерть воркера: По истечении heartbeat_timeout оркестратор переводит запуск в статус unknown, после чего запускается процедура восстановления.

---

## 2.7. Жизненный цикл: Конфликты, Восстановление, Удаление и Retention

### 1. Version Conflict (Конфликты версий)
- Изоляция веток/путей: Каждый Run пишет результат по уникальному пути projects/{project_id}/runs/{run_id}/result.md. Прямые перезаписи одного и того же пути параллельными запусками исключены на уровне файловой структуры.
- Конкурентный push в репозиторий: Для предотвращения гонок при одновременном завершении запусков используется распределенная блокировка (Distributed Lock на уровне PostgreSQL: pg_advisory_xact_lock(project_id)) перед выполнением pull --rebase и push.

### 2. Recovery (Восстановление)
- Point-in-Time Recovery событий: События в таблице events защищены составным индексом UNIQUE(run_id, sequence). При падении клиента воспроизведение состояния выполняется чтением журнала events с последнего подтвержденного sequence.
- Восстановление Git-консистентности: В случае расхождения данных таблица artifacts сверяется с Git-деревом фоновым процессом GitConsistencyJob. Если коммит не найден в апстриме, запуск помечается как failed с кодом ошибки E_GIT_COMMIT_NOT_FOUND.

### 3. Delete (Удаление)
- Soft Delete по умолчанию: При удалении проекта или запуска в PostgreSQL проставляется статус archived / deleted_at. Данные в Git остаются неизменными для сохранения истории аудита.
- Hard Delete (GDPR / Практика очистки):
  - Записи в PostgreSQL удаляются каскадно (ON DELETE CASCADE).
  - В Git коммитится специальный tombstone-файл либо инициируется git rm -r projects/{project_id} с созданием коммита удаления. Физическая перезапись истории Git (git filter-repo) запрещена в штатном режиме во избежание повреждения общих веток.

### 4. Retention Policy (Сроки хранения)
- PostgreSQL Events: Детализированные шаги events хранятся 90 дней, после чего архивируются в холодное хранилище или сжимаются до снапшота результата.
- Git Repositories: Входные materials и итоговые artifacts имеют бессрочный срок хранения (Permanent Retention), так как представляют собой каноническую историю проекта.
- **Временные манифесты (ctf_files):** Манифесты с признаком is_internal = true подлежат автоматической очистке/архивации че180 днейей** после перевода проекта в статус archived.
  
## 3. Последствия (Consequences)
Положительные:
Соответствие требованиям заказчика: Полное исключение стороннего S3-хранилища для артефактов первой версии.

Встроенное версионирование и аудит: Каждая версия отчета привязана к неизменяемому коммиту (commit_sha). Заказчик может использовать git diff для сравнения результатов разных запусков.

Удобный просмотр: Пользователь или эксперт может просматривать сгенерированные Markdown-отчеты напрямую в веб-интерфейсе репозитория (GitHub/GitLab).

Ограничения и риски:
Конфликты параллельной записи: Необходима сериализация коммитов (блокировка или раздельные ветки под каждый runId), чтобы избежать merge-конфликтов.

Ограничение на размер файлов: Git не предназначен для хранения тяжелых бинарников (более 50–100 МБ). В v1 это приемлемо, так как артефакт — Markdown-документ, но для больших файлов в будущем потребуется подключение Git LFS.

