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
JWT_ACCESS_SECRET="তোমার-নিজের-random-64-char-string"
JWT_ACCESS_EXPIRY="15m"
JWT_REFRESH_SECRET="আলাদা-আরেকটা-random-64-char-string"
JWT_REFRESH_EXPIRY="7d"
```
---

#### Random secret generate korar jonno
```bash
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
```
---

#### `auth.module.ts`
```bash
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { JwtModule } from '@nestjs/jwt';
import { ConfigModule, ConfigService } from '@nestjs/config';
import type { StringValue } from 'ms';

@Module({
  imports: [JwtModule.registerAsync({
    imports: [ConfigModule],
    inject: [ConfigService],
    useFactory: (config: ConfigService) => ({
      secret: config.getOrThrow<string>('JWT_ACCESS_SECRET'),
      signOptions: {
        expiresIn: config.get<StringValue>('JWT_ACCESS_EXPIRY'),
      },
    })
  }),],
  providers: [AuthService],
  controllers: [AuthController]
})
export class AuthModule {}
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
import { JwtService, JwtSignOptions } from '@nestjs/jwt';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class AuthService {
  private readonly SALT_ROUNDS = 12; // 10 default, 12 production-grade balance (security vs speed)

  constructor(
    private readonly prisma: PrismaService,
    private readonly jwtService: JwtService,
    private readonly configService: ConfigService,
  ) {}

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

    const tokens = this.generateTokens(user.id, user.email);
    return { user: { id: user.id, email: user.email }, ...tokens };
    }

  private generateTokens(userId: string, email: string) {
    const payload = { sub: userId, email };

    const accessToken = this.jwtService.sign(payload); // module-এ registered default (access) secret/expiry ব্যবহার করবে

    const refreshToken = this.jwtService.sign(payload, {
      secret: this.configService.get<string>('JWT_REFRESH_SECRET'),
      expiresIn: this.configService.get<JwtSignOptions['expiresIn']>(
        'JWT_REFRESH_EXPIRY',
      ),
    });

    return { accessToken, refreshToken };
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


#### Refresh token httpOnly cookie-te set koro
#### `main.ts`
```bash
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module';
import cookieParser from 'cookie-parser';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,          
      forbidNonWhitelisted: true, 
      transform: true,          
      transformOptions: { enableImplicitConversion: true },
    }),
  );

  app.use(cookieParser());

  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```
---


#### `auth.controller.ts`
```bash
import { Body, Controller, Post, HttpCode, HttpStatus, Res  } from '@nestjs/common';
import { AuthService } from './auth.service';
import { RegisterDto } from './dto/register.dto';
import { LoginDto } from './dto/login.dto'
import type { Response } from 'express';

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
  async login(
    @Body() dto: LoginDto,
    @Res({ passthrough: true }) res: Response,
  ) {
    const { user, accessToken, refreshToken } = await this.authService.login(dto);

    res.cookie('refresh_token', refreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production', // dev-এ HTTPS না থাকলে false লাগবে
      sameSite: 'strict',
      maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days, JWT_REFRESH_EXPIRY-র সাথে match রাখা
      path: '/auth', // শুধু auth routes-এ পাঠানো হবে
    });

    return { user, accessToken };
    // refreshToken response body-তে কখনো ফেরত যাবে না — শুধু cookie-তে
  }

}
```
---


## /auth/refresh Endpoint + JwtAuthGuard

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
import { JwtService, JwtSignOptions } from '@nestjs/jwt';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class AuthService {
  private readonly SALT_ROUNDS = 12; // 10 default, 12 production-grade balance (security vs speed)

  constructor(
    private readonly prisma: PrismaService,
    private readonly jwtService: JwtService,
    private readonly configService: ConfigService,
  ) {}

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

    const tokens = this.generateTokens(user.id, user.email);
    return { user: { id: user.id, email: user.email }, ...tokens };
    }

  private generateTokens(userId: string, email: string) {
    const payload = { sub: userId, email };

    const accessToken = this.jwtService.sign(payload); // module-এ registered default (access) secret/expiry ব্যবহার করবে

    const refreshToken = this.jwtService.sign(payload, {
      secret: this.configService.get<string>('JWT_REFRESH_SECRET'),
      expiresIn: this.configService.get<JwtSignOptions['expiresIn']>(
        'JWT_REFRESH_EXPIRY',
      ),
    });

    return { accessToken, refreshToken };
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

  async refreshTokens(refreshToken: string) {
  let payload: { sub: string; email: string };

  try {
    payload = await this.jwtService.verifyAsync(refreshToken, {
      secret: this.configService.get<string>('JWT_REFRESH_SECRET'),
    });
  } catch {
    throw new UnauthorizedException('Refresh token invalid অথবা expired');
  }

  // user এখনো exist করে কিনা re-check করা (deleted/deactivated account হলে block হবে)
  const user = await this.prisma.user.findUnique({
    where: { id: payload.sub },
    select: { id: true, email: true },
  });

  if (!user) {
    throw new UnauthorizedException('User খুঁজে পাওয়া যায়নি');
  }

  const tokens = this.generateTokens(user.id, user.email);
  return { user, ...tokens };
}

}
```
---


#### `auth.controller.ts`
```bash
import { Body, Controller, Post, HttpCode, HttpStatus, Res, Req, UnauthorizedException } from '@nestjs/common';
import { AuthService } from './auth.service';
import { RegisterDto } from './dto/register.dto';
import { LoginDto } from './dto/login.dto'
import type { Response } from 'express';
import type { Request } from 'express';

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
  async login(
    @Body() dto: LoginDto,
    @Res({ passthrough: true }) res: Response,
  ) {
    const { user, accessToken, refreshToken } = await this.authService.login(dto);

    res.cookie('refresh_token', refreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production', // dev-এ HTTPS না থাকলে false লাগবে
      sameSite: 'strict',
      maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days, JWT_REFRESH_EXPIRY-র সাথে match রাখা
      path: '/auth', // শুধু auth routes-এ পাঠানো হবে
    });

    return { user, accessToken };
    // refreshToken response body-তে কখনো ফেরত যাবে না — শুধু cookie-তে
  }

  @Post('refresh')
  @HttpCode(HttpStatus.OK)
  async refresh(
    @Req() req: Request,
    @Res({ passthrough: true }) res: Response,
  ) {
  const oldRefreshToken = (req as Request & { cookies?: Record<string, string> }).cookies?.['refresh_token'];

  if (!oldRefreshToken) {
    throw new UnauthorizedException('Refresh token পাওয়া যায়নি');
  }

  const { user, accessToken, refreshToken } =
    await this.authService.refreshTokens(oldRefreshToken);

  res.cookie('refresh_token', refreshToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    maxAge: 7 * 24 * 60 * 60 * 1000,
    path: '/auth',
  });

  return { user, accessToken };
  }

}
```
---

#### JwtAuthGuard banaw (protected routes-er jonno)
```bash
mkdir -p src/auth/guards
```
---

#### `src/auth/guards/jwt-auth.guard.ts`
```bash
import {
  CanActivate,
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { Request } from 'express';

@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(private readonly jwtService: JwtService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest<Request>();
    const token = this.extractToken(request);

    if (!token) {
      throw new UnauthorizedException('Access token পাওয়া যায়নি');
    }

    try {
      const payload = await this.jwtService.verifyAsync(token);
      // এখানে access token verify হবে module-এ registered default secret দিয়ে
      request['user'] = payload; // controller-এ @Req().user দিয়ে access করা যাবে
    } catch {
      throw new UnauthorizedException('Access token invalid অথবা expired');
    }

    return true;
  }

  private extractToken(request: Request): string | undefined {
    const authHeader = request.headers['authorization'];
    if (!authHeader) return undefined;

    const [type, token] = authHeader.split(' ');
    return type === 'Bearer' ? token : undefined;
  }
}
```
---


#### `auth.controller.ts`
```bash
import { UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from './guards/jwt-auth.guard';

@UseGuards(JwtAuthGuard)
@Get('me')
getProfile(@Req() req: Request) {
  return req['user'];
}
```
---


#### ``
```bash

```
---
