# Connect NestJS with Prisma and Neon (Prisma v8)

## 1. Create NestJS Project

```bash
nest new my-nest
```

```bash
cd my-nest
```

---

## 2. Install `@nestjs/config`

Environment variables ব্যবহার করার জন্য `@nestjs/config` install করো।

```bash
npm i @nestjs/config
```

### `app.module.ts`

```ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

### কেন `isGlobal: true`?

`isGlobal: true` দিলে অন্য module-এ আলাদাভাবে `ConfigModule` import না করেও `ConfigService` ব্যবহার করা যাবে।

---

## 3. Create Neon PostgreSQL Database

[Neon](https://neon.tech/) এ একটি PostgreSQL project তৈরি করো।

তারপর Neon থেকে database connection string copy করো।

Example:

```env
DATABASE_URL="postgresql://username:password@host/neondb?sslmode=require"
```

---

## 4. Create `example.env`

Project root-এ `example.env` তৈরি করো:

```env
DATABASE_URL=""
```

> `example.env`-এ real password বা secret রাখবে না। এটি শুধু environment variable-এর structure দেখানোর জন্য।

---

# 5. Install Prisma 8

Prisma 8 install করো:

```bash
npm install -D prisma@latest
```

PostgreSQL ORM package:

```bash
npm install @prisma/orm-postgres
```

> `@prisma/orm-postgres` একবারই install করতে হবে।

---

# 6. Initialize Prisma 8

### Interactive setup

```bash
npx prisma@latest orm init --target postgres
```

Prisma প্রশ্ন করলে:

* PostgreSQL নির্বাচন করবে
* Authoring হিসেবে `PSL` নির্বাচন করবে
* Default contract path রাখতে পারো

### Recommended: Non-interactive setup

```bash
npx prisma@latest orm init --yes --target postgres --authoring psl
```

এই command Prisma 8-এর প্রয়োজনীয় configuration এবং starter Prisma files তৈরি করবে।

সাধারণত structure হবে:

```text
my-nest/
├── src/
│   └── prisma/
│       ├── contract.prisma
│       └── ...
├── prisma.config.ts
├── .env
└── package.json
```

Prisma 8-এর `orm init` existing project-এর মধ্যে Prisma configuration এবং প্রয়োজনীয় files scaffold করার জন্য ব্যবহৃত হয়।

---

# 7. Configure Environment Variable

`example.env` থেকে `DATABASE_URL` copy করে `.env` file-এ রাখো।

### `.env`

```env
DATABASE_URL="YOUR_NEON_DATABASE_URL"
```

Example:

```env
DATABASE_URL="postgresql://username:password@host/neondb?sslmode=require"
```

> `.env` কখনো GitHub-এ commit করবে না।

`.gitignore`-এ `.env` থাকা উচিত।

---

# 8. Configure Prisma 8

`orm init` সাধারণত `prisma.config.ts` তৈরি করে।

### `prisma.config.ts`

```ts
import 'dotenv/config';
import { defineConfig } from '@prisma/cli-engine';
import { defineConfig as ormConfig } from '@prisma/orm-postgres/config';

export default defineConfig({
  orm: ormConfig({
    contract: './src/prisma/contract.prisma',
    db: {
      connection: process.env['DATABASE_URL']!,
    },
  }),
});
```

এখানে:

* `contract` → Prisma contract-এর location
* `DATABASE_URL` → Neon PostgreSQL connection string
* `@prisma/orm-postgres/config` → PostgreSQL-এর Prisma 8 ORM configuration

Prisma 8 CLI-এর database-related commands `prisma.config.ts` থেকে ORM configuration পড়ে।

---

# 9. Create Prisma Contract

### `src/prisma/contract.prisma`

```prisma
model Book {
  id        String   @id @default(uuid())
  title     String
  author    String
  createdAt DateTime @default(now())
}
```

এখানে `Book` আমাদের database model।

### Fields

```text
id
title
author
createdAt
```

---

# 10. Emit Prisma Contract

`contract.prisma` তৈরি বা পরিবর্তন করার পর contract artifacts generate করতে হবে।

```bash
npx prisma@latest contract emit
```

এতে সাধারণত তৈরি হবে:

```text
src/prisma/
├── contract.prisma
├── contract.json
└── contract.d.ts
```

### গুরুত্বপূর্ণ

`contract.json` এবং `contract.d.ts` manually edit করবে না।

`contract.prisma` পরিবর্তন করলে আবার:

```bash
npx prisma@latest contract emit
```

চালাবে।

Prisma 8-এর `contract emit` database-এ connect করে না; এটি contract থেকে runtime ও tooling-এর জন্য generated artifacts তৈরি করে।

---

# 11. Initialize Database

এখানে **দুটি আলাদা situation** বুঝতে হবে।

## Case A — একদম নতুন/Empty Database

যদি Neon database নতুন এবং empty হয়:

```bash
npx prisma@latest contract emit
```

তারপর:

```bash
npx prisma@latest db init
```

তারপর verify করতে পারো:

```bash
npx prisma@latest db verify
```

### Flow

```text
contract.prisma
      ↓
contract emit
      ↓
contract.json
      ↓
db init
      ↓
Neon Database
```

`db init` বর্তমান emitted contract অনুযায়ী নতুন database bootstrap করে।

> **এখানে `db init`-এর পরে আবার `db update` চালানোর দরকার নেই।**

---

# 12. Existing Database হলে

যদি database-এ আগে থেকেই table থাকে এবং তুমি `contract.prisma` পরিবর্তন করো, তাহলে:

### Step 1 — Contract emit

```bash
npx prisma@latest contract emit
```

### Step 2 — আগে dry-run

```bash
npx prisma@latest db update --dry-run
```

এতে Prisma দেখাবে কী কী পরিবর্তন করতে যাচ্ছে।

Example:

```text
Drop table "post"
Drop table "user"
Create table "book"
```

### Step 3 — Apply changes

যদি changes ঠিক থাকে:

```bash
npx prisma@latest db update
```

যদি destructive operation থাকে, Prisma confirmation চাইবে।

### Step 4 — Verify

```bash
npx prisma@latest db verify
```

Prisma-এর official recommended direct-reconciliation flow হলো:

```bash
npx prisma@latest contract emit
npx prisma@latest db update --dry-run
npx prisma@latest db update
npx prisma@latest db verify
```

---

# 13. `db init` vs `db update`

এটা খুব গুরুত্বপূর্ণ।

| Command               | কখন ব্যবহার করবে                                             |
| --------------------- | ------------------------------------------------------------ |
| `contract emit`       | Contract তৈরি/পরিবর্তনের পর                                  |
| `db init`             | নতুন/empty database bootstrap করতে                           |
| `db update`           | Existing database-কে current contract-এর সাথে reconcile করতে |
| `db update --dry-run` | Change apply করার আগে preview করতে                           |
| `db verify`           | Database contract-এর সাথে match করছে কিনা যাচাই করতে         |

### সহজভাবে

```text
New Database
     ↓
contract emit
     ↓
db init
```

আর:

```text
Existing Database
     ↓
contract change
     ↓
contract emit
     ↓
db update --dry-run
     ↓
db update
     ↓
db verify
```

Prisma 8 documentation-এও `db init` এবং `db update` এইভাবেই আলাদা করা হয়েছে।

---

# 14. PostgreSQL Driver

Prisma 8 PostgreSQL setup-এর জন্য:

```bash
npm install @prisma/orm-postgres
```

> এই package আগে install করা হয়ে থাকলে আবার install করার দরকার নেই।

---

# 15. Final Project Structure

এই পর্যায়ে project structure মোটামুটি এমন হবে:

```text
my-nest/
│
├── src/
│   ├── prisma/
│   │   ├── contract.prisma
│   │   ├── contract.json
│   │   ├── contract.d.ts
│   │   └── db.ts
│   │
│   ├── app.controller.ts
│   ├── app.service.ts
│   ├── app.module.ts
│   └── main.ts
│
├── .env
├── example.env
├── prisma.config.ts
├── package.json
└── tsconfig.json
```

> `orm init` version/configuration অনুযায়ী কিছু generated file-এর exact location বা additional files ভিন্ন হতে পারে।

---

# 16. Useful Prisma 8 Commands

### Emit contract

```bash
npx prisma@latest contract emit
```

### Initialize new database

```bash
npx prisma@latest db init
```

### Preview database changes

```bash
npx prisma@latest db update --dry-run
```

### Apply changes

```bash
npx prisma@latest db update
```

### Verify database

```bash
npx prisma@latest db verify
```

### Inspect database schema

```bash
npx prisma@latest db schema
```

---

# Important Rule

Prisma 8-এ এই workflow মনে রাখবে:

```text
                contract.prisma
                       │
                       ▼
                contract emit
                       │
                       ▼
                contract.json
                       │
             ┌─────────┴─────────┐
             │                   │
       New Database       Existing Database
             │                   │
             ▼                   ▼
          db init          db update --dry-run
                                 │
                                 ▼
                             db update
                                 │
                                 ▼
                             db verify
```

এই workflow অনুসরণ করলে Prisma 8-এ `User`/`Post` starter table-এর মতো unexpected schema issue সহজে বুঝতে পারবে।
