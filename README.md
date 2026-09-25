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


#### `main.ts`
```bash
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,          // DTO-তে না থাকা extra field silently drop করবে
      forbidNonWhitelisted: true, // extra field থাকলে error throw করবে (security: mass-assignment prevent)
      transform: true,          // payload কে DTO class instance-এ auto-transform করবে
      transformOptions: { enableImplicitConversion: true },
    }),
  );

  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```
---


#### dto folder create koro
```bash
mkdir -p src/auth/dto
```
---


#### `src/auth/dto/register.dto.ts`
```bash
import { IsEmail, IsString, MinLength, MaxLength, Matches } from 'class-validator';

export class RegisterDto {
  @IsEmail({}, { message: 'Valid email address দিতে হবে' })
  email: string;

  @IsString()
  @MinLength(8, { message: 'Password কমপক্ষে 8 characters হতে হবে' })
  @MaxLength(72, { message: 'Password 72 characters-এর বেশি হতে পারবে না' }) // bcrypt 72-byte limit
  @Matches(/(?=.*[a-z])/, { message: 'Password-এ একটা lowercase letter থাকতে হবে' })
  @Matches(/(?=.*[A-Z])/, { message: 'Password-এ একটা uppercase letter থাকতে হবে' })
  @Matches(/(?=.*\d)/, { message: 'Password-এ একটা digit থাকতে হবে' })
  password: string;
}
```
---


