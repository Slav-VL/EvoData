# Файловая структура проекта для табличного приложения (Node.js + React + PostgreSQL + ClickHouse + Docker Compose)

Ниже — практичная структура для продукта с возможностями:
- таблицы и «книги» (workbooks),
- совместная работа,
- аналитика и кубы,
- SCD (медленно меняющиеся измерения),
- хранение табличных данных.

```text
EvoData/
├─ apps/
│  ├─ web/                              # React-приложение (UI)
│  │  ├─ public/
│  │  └─ src/
│  │     ├─ app/                        # Router/layout/providers
│  │     ├─ features/
│  │     │  ├─ auth/
│  │     │  ├─ workbooks/               # Книги/листы/права
│  │     │  ├─ tables/                  # Просмотр/редактирование таблиц
│  │     │  ├─ collaboration/           # Комментарии, presence, shared links
│  │     │  ├─ analytics/               # Дашборды, витрины, отчёты
│  │     │  └─ cubes/                   # UI конструктора кубов
│  │     ├─ entities/                   # Базовые сущности (user, workbook, table)
│  │     ├─ shared/                     # UI-kit, hooks, utils, api-client
│  │     └─ pages/
│  └─ api/                              # Node.js backend (REST/GraphQL + jobs)
│     ├─ src/
│     │  ├─ main.ts
│     │  ├─ config/
│     │  ├─ modules/
│     │  │  ├─ auth/
│     │  │  ├─ users/
│     │  │  ├─ organizations/
│     │  │  ├─ workbooks/
│     │  │  ├─ sheets/
│     │  │  ├─ tables/
│     │  │  ├─ data-ingestion/          # Импорт CSV/XLSX/JSON
│     │  │  ├─ collaboration/           # ACL/RBAC, комментарии, realtime events
│     │  │  ├─ analytics/               # Запросы к ClickHouse, метрики
│     │  │  ├─ cubes/                   # Определения кубов, измерения, меры
│     │  │  ├─ dimensions/              # SCD типы, версионирование измерений
│     │  │  ├─ lineage/                 # Происхождение данных
│     │  │  └─ audit/                   # Журнал изменений
│     │  ├─ db/
│     │  │  ├─ postgres/                # Prisma/TypeORM schemas + migrations
│     │  │  └─ clickhouse/              # SQL миграции для CH
│     │  ├─ workers/                    # Очереди: расчёт кубов, пересчёт витрин
│     │  └─ integrations/               # S3/MinIO, email, webhook
│     └─ test/
├─ packages/                            # Общие библиотеки monorepo
│  ├─ ui/                               # Общие React-компоненты
│  ├─ types/                            # DTO, контракты, схемы (zod/io-ts)
│  ├─ eslint-config/
│  └─ tsconfig/
├─ infra/
│  ├─ docker/
│  │  ├─ api.Dockerfile
│  │  ├─ web.Dockerfile
│  │  └─ nginx.conf
│  ├─ compose/
│  │  ├─ docker-compose.yml
│  │  ├─ docker-compose.dev.yml
│  │  └─ docker-compose.prod.yml
│  ├─ postgres/
│  │  ├─ init/
│  │  └─ conf/
│  ├─ clickhouse/
│  │  ├─ init/
│  │  ├─ users.d/
│  │  └─ config.d/
│  └─ monitoring/
│     ├─ prometheus.yml
│     └─ grafana/
├─ warehouse/                           # DWH-логика и модели
│  ├─ models/
│  │  ├─ staging/
│  │  ├─ core/
│  │  ├─ marts/
│  │  └─ cubes/
│  ├─ dimensions/
│  │  ├─ scd_type1/
│  │  ├─ scd_type2/
│  │  └─ helpers/
│  ├─ snapshots/                        # Историзация (если используете dbt)
│  └─ seeds/
├─ storage/
│  ├─ raw/                              # Сырые файлы
│  ├─ processed/                        # Нормализованные датасеты
│  └─ exports/
├─ scripts/
│  ├─ bootstrap.sh
│  ├─ migrate-postgres.sh
│  ├─ migrate-clickhouse.sh
│  └─ backfill-scd.sh
├─ .env.example
├─ package.json
├─ pnpm-workspace.yaml
└─ README.md
```

## Что хранить в PostgreSQL, а что в ClickHouse

### PostgreSQL (OLTP, транзакционка)
- Пользователи, роли, организации, доступы (RBAC).
- Метаданные книг/листов/таблиц.
- Схемы наборов данных, правила валидации.
- Операционные журналы и настройки.
- SCD-измерения (если нужны точные upsert/update/delete транзакции).

### ClickHouse (OLAP, аналитика)
- Факты (события, транзакции, агрегируемые данные).
- Денормализованные витрины и предрасчётные агрегаты.
- Данные для построения кубов и тяжёлые аналитические запросы.

## Минимальные сервисы в `docker-compose`
- `web` (React)
- `api` (Node.js)
- `postgres`
- `clickhouse`
- `redis` (очереди/background jobs)
- `minio` (файловое объектное хранилище)
- `nginx` (reverse proxy)

## Рекомендации по SCD
- **SCD Type 1**: для исправления текущих значений без истории.
- **SCD Type 2**: для полной истории изменений (поля `valid_from`, `valid_to`, `is_current`, `surrogate_key`).
- Храните бизнес-ключ отдельно от surrogate key.
- Для аналитики отправляйте в ClickHouse уже «расплющенные» версии измерений/фактов.

## С чего стартовать (MVP)
1. Поднять `web + api + postgres + clickhouse + redis` через Compose.
2. Реализовать модули `auth`, `workbooks`, `tables`, `data-ingestion`.
3. Добавить базовую ACL-модель совместного доступа.
4. Сделать 1-2 аналитические витрины в ClickHouse.
5. Включить SCD Type 2 хотя бы для одной ключевой размерности (например, `customer`).
