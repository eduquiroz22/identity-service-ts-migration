# Identity Service TS Migration

This project migrates the existing **Topcoder Identity Service** (originally in Java + MySQL) to a **TypeScript + Prisma + PostgreSQL** architecture.

The challenge required full migration and validation of the `RoleResource` endpoints, with setup for integration of `Group`, `Client`, and other resources from the `common_oltp` database.

---

## 📦 Project Structure

```
├── prisma/
│   ├── schema.mysql.prisma        # MySQL Authorization schema (legacy)
│   ├── schema.identity.prisma     # New PostgreSQL Identity schema
│   └── schema.postgres.prisma     # Existing PostgreSQL common_oltp schema (read-only)
├── generated/                     # Prisma Clients (mysql-client, identity-client, postgres-client)
├── src/
│   ├── controllers/               # RoleResource controller logic
│   ├── middlewares/              # authMiddleware with M2M/admin check
│   ├── migration/                # Migration scripts from MySQL → Postgres
│   ├── routes/                   # RoleResource Express routes
│   ├── services/                 # Member API fetcher
│   └── utils/                    # Helper: getNextId(), logger, etc.
├── test/                         # Jest tests for RoleResource
├── data/                         # 🔽 Place provided dump files here: `Authorization`, `common_oltp`
├── docker-compose.yml            # MySQL + Postgres setup
└── README.md
```

---

## 🚀 Setup Instructions

### 1. Clone and install dependencies
```bash
pnpm install
```

### 2. Prepare your data
Download and place the following dump files inside a `data/` folder at the project root:

```
./data/Authorization      # MySQL dump
./data/common_oltp        # PostgreSQL dump
```

> 📌 These are provided in the challenge forum.

---

### 3. Start services (MySQL + PostgreSQL)

Run:
```bash
docker-compose up -d
```
---

### 4. Import databases
```bash
# Load MySQL dump
docker cp data/Authorization identity-mysql:/tmp/
docker exec -it identity-mysql bash -c "mysql -u root -proot Authorization < /tmp/Authorization"

# Load PostgreSQL dump
docker cp data/common_oltp identity-postgres:/tmp/
docker exec -it identity-postgres psql -U postgres -d common_oltp -f /tmp/common_oltp
```

---

### 5. Generate Prisma clients
```bash
npx prisma generate --schema=prisma/schema.mysql.prisma
npx prisma generate --schema=prisma/schema.postgres.prisma
npx prisma db push --schema=prisma/schema.identity.prisma
```

> ⚠️ Be sure all DBs are accessible from `.env`

---

### 6. Migrate data from MySQL → PostgreSQL
```bash
npx ts-node src/migration/migrateRole.ts
npx ts-node src/migration/migrateRoleAssignment.ts
```

#### ⚙️ Parallel Migration
Migration scripts use parallel processing for RoleAssigment. You can control the level via:
```env
MIGRATION_PARALLELISM=6
```
Defaults to 6 if not set.

#### 🔍 DataMigrated vs. DataMySQL
Snapshot showing a side-by-side verification of a few sample records from both Role and RoleAssignment tables, confirming that the data was correctly migrated from MySQL to PostgreSQL.

![Screenshot from 2025-04-25 04-54-44](https://github.com/user-attachments/assets/df1b17b2-d5d6-4bee-9374-e07ddb96b5d8)


---

### 7. Run the API Server
```bash
npx ts-node src/index.ts
```

---

### 8. Run Tests
```bash
npx jest
```

---

## ✅ RoleResource API Endpoints

- `GET /roles`
- `GET /roles/:id`
- `POST /roles`
- `PUT /roles/:id`
- `DELETE /roles/:id`
- `POST /roles/:id/assign?subjectId=X`
- `DELETE /roles/:id/deassign?subjectId=X`
- `GET /roles/:id/hasrole?subjectId=X`

---

## 🔎 Postman Collection
Import the provided `postman/role-resource.postman_collection.json` and run tests via the UI or CLI.

![Screenshot from 2025-04-25 05-33-20](https://github.com/user-attachments/assets/a7de788b-4837-48d6-969f-f84a27c2ccf2)


---

## ⚠️ Prisma Limitation: Tables Without Unique Identifiers

During introspection of the `common_oltp` PostgreSQL schema, Prisma **ignored several tables** due to the absence of primary keys or unique identifiers.

These include:
- `audit_user`, `contact`, `request`, `user_achievement`, etc.

They have been excluded from the schema and cannot be queried via Prisma. Raw SQL may be required in the future.

Reference from the forum:
> _"If some of the tables are not used in the 4 resources, otherwise, we should make sure they're handled properly."_

---

## 💡 Prisma Clients Used

This project uses **three** separate Prisma Clients:

| Client           | Schema File                | Purpose                                 |
|------------------|----------------------------|------------------------------------------|
| `mysql-client`   | `schema.mysql.prisma`      | Read legacy MySQL `Authorization` data  |
| `postgres-client`| `schema.postgres.prisma`   | Read-only `common_oltp` PostgreSQL dump |
| `identity-client`| `schema.identity.prisma`   | Main service logic & API storage        |

> Prisma does not support multiple datasources per schema — hence the split.

---

## 📊 Optimizations

- **Parallel migration** of `RoleAssignment` records using batch + Promise.all for speedup
- **Custom ID handling** for migrated roles to maintain legacy compatibility
- **CLI progress logs** for tracking large migrations

![Screenshot from 2025-04-25 05-38-01](https://github.com/user-attachments/assets/8d036271-f5b7-4b16-9d66-d4b929128406)

---

## 🔌 Infrastructure-Ready Components

This project includes placeholder implementations to support key infrastructure features expected in the final Identity Service:

- **Redis Cache (`src/lib/infra/redis.ts`)**  
  Prepared to store session or permission data. Replace with `ioredis` for production.

- **Kafka Publisher (`src/lib/infra/kafka.ts`)**  
  Stubbed class to simulate event publishing. Integrate with `kafkajs` or similar client.

- **Shiro-like Permissions (`src/lib/infra/permissions.ts`)**  
  Basic middleware structure for M2M and admin role checks. Ready to extend with fine-grained RBAC logic.

> These modules are not required by the challenge but were scaffolded for architecture completeness.


## 📝 Notes

- This challenge **only** requires RoleResource logic.
- `Group`, `Client`, and other models were included in the Postgres schema (`common_oltp`) but not implemented.
- JWT logic is mocked to recognize `admin` and `machine` roles for test purposes.
