## Connect NestJ with Prisma and Neon (Prisma v8)

#### Create Project
```bash
nest new my-nest
```
```bash
cd my-nest
```
---

### install @nestjs/config
```bash
npm i @nestjs/config
```

### `app.module.ts`
```bash
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [ConfigModule.forRoot({ isGlobal: true })],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

>#### Neon e project create koro and then database connect koro and example.env te paste koro.
<img width="1597" height="762" alt="image" src="https://github.com/user-attachments/assets/bf2fd0be-b6c5-4f60-9b5b-15ef35768385" />


#### `example.env`
```bash
DATABASE_URL=''
```
---


#### Prisma v8 install
```bash
npm install -D prisma@latest
```
```bash
npm install @prisma/orm-postgres
```
> interactive setup
```bash
npx prisma@latest orm init --target postgres
```
> Or, non-interactive setup  [recommended]
```bash
npx prisma@latest orm init --yes --target postgres --authoring psl
```
---
