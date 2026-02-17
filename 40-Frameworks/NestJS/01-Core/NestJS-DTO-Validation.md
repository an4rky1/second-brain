---
created: 2026-02-17
tags:
  - nestjs
  - dto
  - validation
  - pipes
  - class-validator
aliases:
  - NestJS DTO Validation
  - NestJS Class Validator
related:
  - NestJS-MOC
  - NestJS-Controllers
  - NestJS-Pipes
---

# NestJS — DTO & Validation

> [!SUMMARY] Обзор
> DTO (Data Transfer Objects) для валидации входящих данных. class-validator, class-transformer, custom validators.

---

## 📝 Basic DTO

```typescript
// users/dto/create-user.dto.ts
import {
  IsString,
  IsEmail,
  IsNotEmpty,
  IsOptional,
  IsInt,
  IsBoolean,
  IsArray,
  IsEnum,
  IsUrl,
  IsPhoneNumber,
  MinLength,
  MaxLength,
  Min,
  Max,
  Matches,
  ValidateNested,
  IsObject,
} from 'class-validator';
import { Type } from 'class-transformer';

// ───────────────────────────────────────────────────────
// ENUMS
// ───────────────────────────────────────────────────────
export enum UserRole {
  USER = 'user',
  ADMIN = 'admin',
  MODERATOR = 'moderator',
}

// ───────────────────────────────────────────────────────
// CREATE DTO
// ───────────────────────────────────────────────────────
export class CreateUserDto {
  @IsString()
  @IsNotEmpty()
  @MinLength(2)
  @MaxLength(50)
  name: string;

  @IsEmail()
  @IsNotEmpty()
  email: string;

  @IsString()
  @MinLength(8)
  @MaxLength(128)
  @Matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])/, {
    message: 'Password must contain uppercase, lowercase, number and special char',
  })
  password: string;

  @IsOptional()
  @IsInt()
  @Min(18)
  @Max(120)
  age?: number;

  @IsOptional()
  @IsEnum(UserRole)
  role?: UserRole;

  @IsOptional()
  @IsBoolean()
  isActive?: boolean;

  @IsOptional()
  @IsArray()
  @IsString({ each: true })
  tags?: string[];
}

// ───────────────────────────────────────────────────────
// NESTED DTO
// ───────────────────────────────────────────────────────
export class AddressDto {
  @IsString()
  @IsNotEmpty()
  street: string;

  @IsString()
  @IsNotEmpty()
  city: string;

  @IsString()
  @IsNotEmpty()
  zipCode: string;

  @IsOptional()
  @IsString()
  state?: string;

  @IsString()
  @IsNotEmpty()
  country: string;
}

export class CreateUserWithAddressDto {
  @IsString()
  @IsNotEmpty()
  name: string;

  @IsEmail()
  @IsNotEmpty()
  email: string;

  @ValidateNested()
  @Type(() => AddressDto)
  address: AddressDto;
}

// ───────────────────────────────────────────────────────
// UPDATE DTO (Partial)
// ───────────────────────────────────────────────────────
import { PartialType, OmitType, PickType } from '@nestjs/swagger';

export class UpdateUserDto extends PartialType(CreateUserDto) {}

// Omit sensitive fields
export class UserResponseDto extends OmitType(CreateUserDto, ['password'] as const) {
  id: number;
  createdAt: Date;
  updatedAt: Date;
}

// Pick only needed fields
export class UserPublicDto extends PickType(UserResponseDto, ['id', 'name', 'email'] as const) {}
```

---

## 🔧 Validation Pipe

### Global Setup

```typescript
// main.ts
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,              // Удалить лишние свойства
      forbidNonWhitelisted: true,   // Бросать ошибку на лишние
      transform: true,              // Преобразовать в DTO класс
      transformOptions: {
        enableImplicitConversion: true,
      },
      validationError: {
        target: false,              // Не включать target в ошибку
        value: false,               // Не включать значение в ошибку
      },
    }),
  );

  await app.listen(3000);
}
bootstrap();
```

### Route Level

```typescript
// users/users.controller.ts
import { ValidationPipe, UsePipes } from '@nestjs/common';

@Controller('users')
export class UsersController {
  @Post()
  @UsePipes(new ValidationPipe({ transform: true }))
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }

  @Post('bulk')
  @UsePipes(new ValidationPipe({ transform: true, whitelist: true }))
  createBulk(@Body() users: CreateUserDto[]) {
    return this.usersService.bulkCreate(users);
  }
}
```

### Parameter Decorator Level

```typescript
@Controller('users')
export class UsersController {
  @Post()
  create(
    @Body(new ValidationPipe({ transform: true }))
    createUserDto: CreateUserDto,
  ) {
    return this.usersService.create(createUserDto);
  }
}
```

---

## 🎯 Custom Validators

### Custom Validator Class

```typescript
// validators/is-password-match.validator.ts
import {
  registerDecorator,
  ValidationOptions,
  ValidatorConstraint,
  ValidatorConstraintInterface,
  ValidationArguments,
} from 'class-validator';

@ValidatorConstraint({ async: false })
export class IsPasswordMatchConstraint implements ValidatorConstraintInterface {
  validate(password: string, args: ValidationArguments) {
    const [relatedPropertyName] = args.constraints;
    const relatedValue = (args.object as any)[relatedPropertyName];
    return password === relatedValue;
  }

  defaultMessage(args: ValidationArguments) {
    const [relatedPropertyName] = args.constraints;
    return 'Password does not match $1';
  }
}

export function IsPasswordMatch(validationOptions?: ValidationOptions) {
  return function (object: any, propertyName: string) {
    registerDecorator({
      target: object.constructor,
      propertyName,
      options: validationOptions,
      constraints: ['confirmPassword'],
      validator: IsPasswordMatchConstraint,
    });
  };
}

// Usage
export class RegisterDto {
  @IsString()
  @MinLength(8)
  password: string;

  @IsPasswordMatch({ message: 'Passwords do not match' })
  confirmPassword: string;
}
```

### Async Validator (Database Check)

```typescript
// validators/is-email-exists.validator.ts
import {
  registerDecorator,
  ValidationOptions,
  ValidatorConstraint,
  ValidatorConstraintInterface,
  ValidationArguments,
} from 'class-validator';
import { Injectable } from '@nestjs/common';
import { ModuleRef } from '@nestjs/core';
import { PrismaService } from '../prisma/prisma.service';

@Injectable()
@ValidatorConstraint({ async: true })
export class IsEmailExistsConstraint implements ValidatorConstraintInterface {
  private prisma: PrismaService;

  constructor(private moduleRef: ModuleRef) {}

  async validate(email: string): Promise<boolean> {
    this.prisma = this.moduleRef.get(PrismaService, { strict: false });
    
    const existing = await this.prisma.user.findUnique({
      where: { email },
    });

    return !existing;  // Return true if email does NOT exist
  }

  defaultMessage(): string {
    return 'Email already exists';
  }
}

export function IsEmailExists(validationOptions?: ValidationOptions) {
  return function (object: any, propertyName: string) {
    registerDecorator({
      target: object.constructor,
      propertyName,
      options: validationOptions,
      validator: IsEmailExistsConstraint,
    });
  };
}

// Usage
export class RegisterDto {
  @IsEmail()
  @IsEmailExists({ message: 'Email already exists' })
  email: string;
}
```

### Conditional Validation

```typescript
// validators/is-field-equal.validator.ts
import {
  registerDecorator,
  ValidationOptions,
  ValidatorConstraint,
  ValidatorConstraintInterface,
} from 'class-validator';

@ValidatorConstraint({ async: false })
export class IsFieldEqualConstraint implements ValidatorConstraintInterface {
  validate(value: any, args: any) {
    const [field, expectedValue] = args.constraints;
    return (args.object as any)[field] === expectedValue;
  }

  defaultMessage(args: any) {
    const [field, expectedValue] = args.constraints;
    return `$property must be equal to ${expectedValue} when ${field} is ${expectedValue}`;
  }
}

export function IsFieldEqual(
  field: string,
  expectedValue: any,
  validationOptions?: ValidationOptions,
) {
  return function (object: any, propertyName: string) {
    registerDecorator({
      target: object.constructor,
      propertyName,
      options: validationOptions,
      constraints: [field, expectedValue],
      validator: IsFieldEqualConstraint,
    });
  };
}

// Usage
export class PaymentDto {
  @IsString()
  type: 'card' | 'paypal' | 'crypto';

  @IsFieldEqual('type', 'card', {
    message: 'Card number is required for card payments',
  })
  @IsOptional()
  cardNumber?: string;
}
```

---

## 🔄 Type Transformation

```typescript
import { Type, Transform } from 'class-transformer';

export class CreateUserDto {
  // Convert string to number
  @Type(() => Number)
  @IsInt()
  age: number;

  // Convert string to boolean
  @Type(() => Boolean)
  @IsBoolean()
  isActive: boolean;

  // Nested object transformation
  @Type(() => AddressDto)
  @ValidateNested()
  address: AddressDto;

  // Array of objects
  @Type(() => ProductDto)
  @ValidateNested({ each: true })
  products: ProductDto[];

  // Custom transform
  @Transform(({ value }) => value?.toLowerCase().trim())
  @IsString()
  email: string;

  // Trim whitespace
  @Transform(({ value }) => value?.trim())
  @IsString()
  name: string;

  // Convert comma-separated string to array
  @Transform(({ value }) => value?.split(',').map((s: string) => s.trim()))
  @IsArray()
  @IsString({ each: true })
  tags: string[];
}
```

---

## 🔗 Связанные заметки

- [[NestJS-MOC]] — NestJS индекс
- [[NestJS-Controllers]] — Контроллеры
- [[NestJS-Pipes]] — Pipes
- [[NestJS-Services]] — Сервисы
