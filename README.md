# NestJS Production Level Signup Login

#### Project Create koro.
```bash
nest new secure-auth-api
```
---

[Connect NestJ with Prisma and Supabase (Prisma v7.10.0)](https://github.com/Omarmdwasimuddin/Connect-NestJ-with-Prisma-and-Supabase-Prisma-v7.10.0-)

#### Generate Prisma Service & Module
```bash
nest g service prisma
```
```bash
nest g module prisma
```
---

#### `src/prisma/prisma.service.ts`
```bash
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from 'generated/prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}
```
---


#### `src/prisma/prisma.module.ts`
```bash
import { Global, Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```
---


#### `app.module.ts`
```bash
import { Module } from '@nestjs/common';
import { PrismaModule } from './prisma/prisma.module';

@Module({
  imports: [PrismaModule],
})
export class AppModule {}
```
---


#### Packages instal
```bash
npm install class-validator class-transformer
```
---

