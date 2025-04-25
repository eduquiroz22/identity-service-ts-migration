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
docker exec -i identity-mysql mysql -u root -proot Authorization < /tmp/Authorization

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

---

## 📝 Notes

- This challenge **only** requires RoleResource logic.
- `Group`, `Client`, and other models were included in the Postgres schema (`common_oltp`) but not implemented.
- JWT logic is mocked to recognize `admin` and `machine` roles for test purposes.

---

## ✅ Submission Checklist

- [x] All RoleResource endpoints implemented
- [x] Prisma schemas (MySQL, PostgreSQL, Identity) separated
- [x] Migrations from MySQL → PostgreSQL completed
- [x] Jest test suite covers all endpoints
- [x] Postman test collection included
- [x] README with complete setup and documentation

---

Feel free to reach out in the Topcoder forum if anything is unclear!

