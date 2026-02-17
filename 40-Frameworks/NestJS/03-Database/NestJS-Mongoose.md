---
created: 2026-02-17
tags:
  - nestjs
  - mongoose
  - mongodb
  - database
aliases:
  - NestJS Mongoose
  - NestJS MongoDB
related:
  - NestJS-MOC
  - NestJS-Prisma
  - NestJS-TypeORM
---

# NestJS — Mongoose (MongoDB)

> [!SUMMARY] Обзор
> Mongoose ORM для MongoDB в NestJS: схемы, модели, индексы, агрегации.

---

## 📦 Установка

```bash
npm install @nestjs/mongoose mongoose
npm install -D @types/mongoose
```

---

## 🔧 Module Setup

```typescript
// database/database.module.ts
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { ConfigModule, ConfigService } from '@nestjs/config';

@Module({
  imports: [
    MongooseModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: (config: ConfigService) => ({
        uri: config.get('MONGODB_URI'),
        useNewUrlParser: true,
        useUnifiedTopology: true,
        // Connection pool
        maxPoolSize: 10,
        minPoolSize: 5,
        serverSelectionTimeoutMS: 5000,
        socketTimeoutMS: 45000,
      }),
      inject: [ConfigService],
    }),
  ],
})
export class DatabaseModule {}
```

---

## 📝 Schema Definition

```typescript
// users/schemas/user.schema.ts
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document, Types } from 'mongoose';
import * as bcrypt from 'bcryptjs';

export type UserDocument = User & Document;

@Schema({
  timestamps: true,  // createdAt, updatedAt
  toJSON: {
    virtuals: true,
    transform: (doc, ret) => {
      delete ret.password;
      delete ret.__v;
    },
  },
})
export class User {
  @Prop({ required: true, trim: true })
  name: string;

  @Prop({
    required: true,
    unique: true,
    lowercase: true,
    trim: true,
    match: [/^\w+([.-]?\w+)*@\w+([.-]?\w+)*(\.\w{2,3})+$/, 'Invalid email'],
  })
  email: string;

  @Prop({
    required: true,
    minlength: 8,
    select: false,  // Don't include in queries by default
  })
  password: string;

  @Prop({
    type: String,
    enum: ['user', 'admin', 'moderator'],
    default: 'user',
  })
  role: string;

  @Prop({ default: true })
  isActive: boolean;

  @Prop({ type: [{ type: Types.ObjectId, ref: 'Post' }] })
  posts: Types.ObjectId[];

  // Virtual
  get fullName(): string {
    return this.name.toUpperCase();
  }
}

export const UserSchema = SchemaFactory.createForClass(User);

// Indexes
UserSchema.index({ email: 1 });
UserSchema.index({ role: 1, isActive: 1 });

// Pre-save hook (hash password)
UserSchema.pre('save', async function (next) {
  if (!this.isModified('password')) return next();
  
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

// Instance method
UserSchema.methods.comparePassword = async function (
  candidatePassword: string,
): Promise<boolean> {
  return bcrypt.compare(candidatePassword, this.password);
};

// Static method
UserSchema.statics.findByEmail = function (email: string) {
  return this.findOne({ email });
};
```

---

## 🔧 Repository Pattern

```typescript
// users/users.repository.ts
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model, FilterQuery, UpdateQuery } from 'mongoose';
import { User, UserDocument } from './schemas/user.schema';

@Injectable()
export class UsersRepository {
  constructor(
    @InjectModel(User.name) private userModel: Model<UserDocument>,
  ) {}

  async findAll(
    filter: FilterQuery<UserDocument> = {},
    skip = 0,
    limit = 10,
  ): Promise<{ data: UserDocument[]; total: number }> {
    const [data, total] = await Promise.all([
      this.userModel.find(filter).skip(skip).limit(limit).exec(),
      this.userModel.countDocuments(filter),
    ]);

    return { data, total };
  }

  async findById(id: string): Promise<UserDocument | null> {
    return this.userModel.findById(id).exec();
  }

  async findByEmail(email: string): Promise<UserDocument | null> {
    return this.userModel.findOne({ email }).exec();
  }

  async create(data: Partial<UserDocument>): Promise<UserDocument> {
    const created = new this.userModel(data);
    return created.save();
  }

  async update(
    id: string,
    data: UpdateQuery<UserDocument>,
  ): Promise<UserDocument | null> {
    return this.userModel
      .findByIdAndUpdate(id, data, { new: true, runValidators: true })
      .exec();
  }

  async delete(id: string): Promise<UserDocument | null> {
    return this.userModel.findByIdAndDelete(id).exec();
  }

  // Aggregation
  async getStats() {
    return this.userModel.aggregate([
      {
        $group: {
          _id: '$role',
          count: { $sum: 1 },
          avgAge: { $avg: '$age' },
        },
      },
      {
        $sort: { count: -1 },
      },
    ]);
  }
}
```

---

## 🛠️ Service

```typescript
// users/users.service.ts
import { Injectable, NotFoundException, ConflictException } from '@nestjs/common';
import { UsersRepository } from './users.repository';

@Injectable()
export class UsersService {
  constructor(private usersRepository: UsersRepository) {}

  async findAll(page = 1, limit = 10, search?: string) {
    const filter: any = {};
    
    if (search) {
      filter.$or = [
        { name: { $regex: search, $options: 'i' } },
        { email: { $regex: search, $options: 'i' } },
      ];
    }

    const skip = (page - 1) * limit;
    const { data, total } = await this.usersRepository.findAll(filter, skip, limit);

    return {
      data,
      meta: {
        total,
        page,
        limit,
        totalPages: Math.ceil(total / limit),
      },
    };
  }

  async findOne(id: string) {
    const user = await this.usersRepository.findById(id);
    
    if (!user) {
      throw new NotFoundException('User not found');
    }
    
    return user;
  }

  async create(createUserDto: CreateUserDto) {
    const existing = await this.usersRepository.findByEmail(createUserDto.email);
    
    if (existing) {
      throw new ConflictException('Email already exists');
    }

    return this.usersRepository.create(createUserDto);
  }

  async update(id: string, updateUserDto: UpdateUserDto) {
    await this.findOne(id);

    if (updateUserDto.email) {
      const existing = await this.usersRepository.findByEmail(updateUserDto.email);
      if (existing && existing.id !== id) {
        throw new ConflictException('Email already exists');
      }
    }

    return this.usersRepository.update(id, updateUserDto);
  }

  async remove(id: string) {
    await this.findOne(id);
    return this.usersRepository.delete(id);
  }
}
```

---

## 🎮 Controller

```typescript
// users/users.controller.ts
import {
  Controller,
  Get,
  Post,
  Put,
  Delete,
  Body,
  Param,
  Query,
} from '@nestjs/common';
import { UsersService } from './users.service';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get()
  findAll(
    @Query('page') page = 1,
    @Query('limit') limit = 10,
    @Query('search') search?: string,
  ) {
    return this.usersService.findAll(page, limit, search);
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(id);
  }

  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }

  @Put(':id')
  update(@Param('id') id: string, @Body() updateUserDto: UpdateUserDto) {
    return this.usersService.update(id, updateUserDto);
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return this.usersService.remove(id);
  }
}
```

---

## 📋 Environment

```bash
# .env
MONGODB_URI=mongodb://localhost:27017/mydb
MONGODB_POOL_MAX=10
MONGODB_POOL_MIN=5
```

---

## 🔗 Связанные заметки

- [[NestJS-MOC]] — NestJS индекс
- [[NestJS-Prisma]] — Prisma ORM
- [[NestJS-TypeORM]] — TypeORM
- [[NoSQL-Cheatsheet]] — NoSQL databases
