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

## Validation Setup — class-validator (Production Standard) + DTOs

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

## Auth Module + Register Logic (bcrypt + Prisma Error Handling)

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
import { Prisma } from 'generated/prisma/client';
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


#### `auth.controller.ts`
```bash
import { Body, Controller, Post, HttpCode, HttpStatus } from '@nestjs/common';
import { AuthService } from './auth.service';
import { RegisterDto } from './dto/register.dto';

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @Post('register')
  @HttpCode(HttpStatus.CREATED)
  register(@Body() dto: RegisterDto) {
    return this.authService.register(dto);
  }
}
```
---


#### `auth.module.ts`
```bash
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';

@Module({
  controllers: [AuthController],
  providers: [AuthService],
})
export class AuthModule {}
```
---


## Login Logic — Failed-Attempt Tracking + Account Lockout

#### Lockout policy define koro (constants)
#### `src/auth/auth.constants.ts`
```bash
export const MAX_FAILED_ATTEMPTS = 5;
export const LOCKOUT_DURATION_MS = 15 * 60 * 1000; // 15 minutes
```
---


#### `auth.service.ts`
```bash
import {
  Injectable,
  ConflictException,
  InternalServerErrorException,
  UnauthorizedException,
  ForbiddenException,
} from '@nestjs/common';
import { Prisma } from 'generated/prisma/client';
import * as bcrypt from 'bcrypt';
import { PrismaService } from '../prisma/prisma.service';
import { RegisterDto } from './dto/register.dto';
import { LoginDto } from './dto/login.dto';
import { MAX_FAILED_ATTEMPTS, LOCKOUT_DURATION_MS } from './auth.constants';

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

  async login(dto: LoginDto) {
    const user = await this.prisma.user.findUnique({
      where: { email: dto.email },
    });

    // user না পেলেও generic message — email enumeration prevent করার জন্য
    if (!user) {
      throw new UnauthorizedException('Email অথবা password ভুল');
    }

    // Lockout active কিনা check করো
    if (user.lockoutUntil && user.lockoutUntil > new Date()) {
      const minutesLeft = Math.ceil(
        (user.lockoutUntil.getTime() - Date.now()) / 60000,
      );
      throw new ForbiddenException(
        `অনেকবার ভুল চেষ্টার কারণে account সাময়িক লক আছে। ${minutesLeft} মিনিট পর আবার চেষ্টা করো`,
      );
    }

    const passwordMatches = await bcrypt.compare(dto.password, user.password);

    if (!passwordMatches) {
      await this.handleFailedLogin(user.id, user.failedLoginAttempts);
      throw new UnauthorizedException('Email অথবা password ভুল');
    }

    // Login successful — counter reset করো (lockout থাকলে সেটাও clear)
    await this.prisma.user.update({
      where: { id: user.id },
      data: {
        failedLoginAttempts: 0,
        lockoutUntil: null,
      },
    });

    return {
      id: user.id,
      email: user.email,
    };
    // Note: JWT token issue করা এখনো বাকি — এটা পরের একটা step-এ আলাদাভাবে করব
  }

  private async handleFailedLogin(userId: string, currentAttempts: number) {
    const newAttemptCount = currentAttempts + 1;
    const shouldLock = newAttemptCount >= MAX_FAILED_ATTEMPTS;

    await this.prisma.user.update({
      where: { id: userId },
      data: {
        failedLoginAttempts: newAttemptCount,
        lockoutUntil: shouldLock
          ? new Date(Date.now() + LOCKOUT_DURATION_MS)
          : undefined,
      },
    });
  }

}
```
---


#### `auth.controller.ts`
```bash
import { Body, Controller, Post, HttpCode, HttpStatus } from '@nestjs/common';
import { AuthService } from './auth.service';
import { RegisterDto } from './dto/register.dto';
import { LoginDto } from './dto/login.dto'

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @Post('register')
  @HttpCode(HttpStatus.CREATED)
  register(@Body() dto: RegisterDto) {
    return this.authService.register(dto);
  }

  @Post('login')
  @HttpCode(HttpStatus.OK)
  login(@Body() dto: LoginDto) {
    return this.authService.login(dto);
  }

}
```
---


## JWT Issuing — Access Token + Refresh Token Strategy

#### Packages instal koro
```bash
npm install @nestjs/jwt @nestjs/config
```
```bash
npm install cookie-parser
```
```bash
npm install -D @types/cookie-parser
```
---


#### `.env`
```bash
Project name: mydb
Database password: 2JFZsODxcbO2PW1L

DATABASE_URL="postgresql://postgres.rdjuzuwtfrpmvfsmswxy:2JFZsODxcbO2PW1L@aws-0-ap-northeast-2.pooler.supabase.com:5432/postgres"

JWT_ACCESS_SECRET="9515db6748b29c4cb034728f700d622d740e2543cf8a82aee8c2bb72ec89fff85117995897baa3225682c77b1cd1443efd4107b185c8df83b1f4a2aa79912377"
JWT_ACCESS_EXPIRY="15m"
JWT_REFRESH_SECRET="b9584e5920f08d64119b620a938ac75a2ee6899e99cf4353555a9b1318d0fd9e43bc7af0bb923306f23dc33dd793ad1b520c1fef8c47ff628f2f582505d60803"
JWT_REFRESH_EXPIRY="7d"
```
---

#### Random secret generate korar jonno
```bash
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
```
---

