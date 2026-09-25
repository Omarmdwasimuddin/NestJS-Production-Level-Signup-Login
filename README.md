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
import { Injectable, OnModuleDestroy, OnModuleInit } from '@nestjs/common';
import { PrismaPg } from '@prisma/adapter-pg';
import { PrismaClient } from 'generated/prisma/client';

@Injectable()
export class PrismaService
  extends PrismaClient
  implements OnModuleInit, OnModuleDestroy
{
  constructor() {
    const adapter = new PrismaPg({
      connectionString: process.env.DATABASE_URL,
    });

    super({
      adapter,
    });
  }

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
  email!: string;

  @IsString()
  @MinLength(8, { message: 'Password কমপক্ষে 8 characters হতে হবে' })
  @MaxLength(72, { message: 'Password 72 characters-এর বেশি হতে পারবে না' }) // bcrypt 72-byte limit
  @Matches(/(?=.*[a-z])/, { message: 'Password-এ একটা lowercase letter থাকতে হবে' })
  @Matches(/(?=.*[A-Z])/, { message: 'Password-এ একটা uppercase letter থাকতে হবে' })
  @Matches(/(?=.*\d)/, { message: 'Password-এ একটা digit থাকতে হবে' })
  password!: string;
}
```
---


#### `src/auth/dto/login.dto.ts`
```bash
import { IsEmail, IsString } from 'class-validator';

export class LoginDto {
  @IsEmail()
  email!: string;

  @IsString()
  password!: string;
}
```
---


#### bcrypt install koro
```bash
npm install bcrypt
```
```bash
npm install -D @types/bcrypt
```
---


#### Auth module generate koro
```bash
nest g module auth
```
```bash
nest g service auth
```
```bash
nest g controller auth
```
---


#### `src/auth/auth.service.ts`
```bash
import {
  Injectable,
  ConflictException,
  InternalServerErrorException,
} from '@nestjs/common';
import { Prisma } from '@prisma/client';
import * as bcrypt from 'bcrypt';
import { PrismaService } from '../prisma/prisma.service';
import { RegisterDto } from './dto/register.dto';

@Injectable()
export class AuthService {
  private readonly SALT_ROUNDS = 12; // 10 default, 12 production-grade balance (security vs speed)

  constructor(private readonly prisma: PrismaService) {}

  async register(dto: RegisterDto) {
    const passwordHash = await bcrypt.hash(dto.password, this.SALT_ROUNDS);

    try {
      const user = await this.prisma.user.create({
        data: {
          email: dto.email,
          password: passwordHash,
        },
        select: {
          // explicit select — password hash কখনো response-এ যাবে না
          id: true,
          email: true,
          createdAt: true,
        },
      });

      return user;
    } catch (error) {
      if (
        error instanceof Prisma.PrismaClientKnownRequestError &&
        error.code === 'P2002' // unique constraint violation
      ) {
        throw new ConflictException('এই email দিয়ে already একটা account আছে');
      }

      // অজানা DB error — client-কে internal detail leak না করে generic error
      throw new InternalServerErrorException('Registration করতে সমস্যা হয়েছে');
    }
  }
}
```
---


