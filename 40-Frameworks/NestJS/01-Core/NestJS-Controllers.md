---
created: 2026-02-17
tags:
  - nestjs
  - controllers
  - routing
  - decorators
aliases:
  - NestJS Controllers
  - NestJS Routing
related:
  - NestJS-MOC
  - NestJS-Modules
  - NestJS-Guards
---

# NestJS — Controllers

> [!SUMMARY] Обзор
> Контроллеры обрабатывают входящие запросы, возвращают ответы клиентам. Декораторы маршрутов, request/response.

---

## 🎮 Basic Controller

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
  ParseIntPipe,
  UseGuards,
  HttpCode,
  HttpStatus,
  NotFoundException,
} from '@nestjs/common';
import { UsersService } from './users.service';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';
import { AuthGuard } from '../auth/auth.guard';

@Controller('users')  // Base path
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  // ─────────────────────────────────────────────────────
  // GET /users
  // ─────────────────────────────────────────────────────
  @Get()
  @UseGuards(AuthGuard)
  findAll(
    @Query('page', new ParseIntPipe({ optional: true })) page = 1,
    @Query('limit', new ParseIntPipe({ optional: true })) limit = 10,
    @Query('search') search?: string,
  ) {
    return this.usersService.findAll({ page, limit, search });
  }

  // ─────────────────────────────────────────────────────
  // GET /users/:id
  // ─────────────────────────────────────────────────────
  @Get(':id')
  async findOne(@Param('id', ParseIntPipe) id: number) {
    const user = await this.usersService.findOne(id);
    
    if (!user) {
      throw new NotFoundException('User not found');
    }
    
    return user;
  }

  // ─────────────────────────────────────────────────────
  // POST /users
  // ─────────────────────────────────────────────────────
  @Post()
  @HttpCode(HttpStatus.CREATED)
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }

  // ─────────────────────────────────────────────────────
  // PUT /users/:id
  // ─────────────────────────────────────────────────────
  @Put(':id')
  update(
    @Param('id', ParseIntPipe) id: number,
    @Body() updateUserDto: UpdateUserDto,
  ) {
    return this.usersService.update(id, updateUserDto);
  }

  // ─────────────────────────────────────────────────────
  // DELETE /users/:id
  // ─────────────────────────────────────────────────────
  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  remove(@Param('id', ParseIntPipe) id: number) {
    return this.usersService.remove(id);
  }
}
```

---

## 📍 HTTP Decorators

### Methods

```typescript
@Controller('users')
export class UsersController {
  @Get()              // GET /users
  findAll() {}

  @Get(':id')         // GET /users/:id
  findOne() {}

  @Post()             // POST /users
  create() {}

  @Put(':id')         // PUT /users/:id
  update() {}

  @Patch(':id')       // PATCH /users/:id
  partialUpdate() {}

  @Delete(':id')      // DELETE /users/:id
  remove() {}

  @Options()          // OPTIONS /users
  options() {}

  @Head()             // HEAD /users
  head() {}

  @All('all')         // All methods /users/all
  all() {}
}
```

### Request Decorators

```typescript
@Controller('users')
export class UsersController {
  @Get(':id')
  findOne(
    @Param('id') id: string,           // Route param
    @Query('search') search: string,   // Query param
    @Body() body: any,                 // Request body
    @Headers() headers: any,           // Headers
    @Headers('authorization') auth: string,  // Specific header
    @Ip() ip: string,                  // IP address
    @Request() req: any,               // Request object
    @Response() res: any,              // Response object
  ) {
    return { id, search, auth, ip };
  }
}
```

### Custom Route Paths

```typescript
@Controller('users')
export class UsersController {
  @Get('active')              // GET /users/active
  findActive() {}

  @Get('search/:term')        // GET /users/search/:term
  search(@Param('term') term: string) {}

  @Get('*')                   // Wildcard
  catchAll() {}

  @Get(['list', 'all'])       // Multiple paths
  list() {}
}
```

---

## 🔄 Response Handling

### Status Codes

```typescript
@Controller('users')
export class UsersController {
  @Post()
  @HttpCode(HttpStatus.CREATED)  // 201
  create() {
    return { message: 'Created' };
  }

  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)  // 204
  remove() {
    return;  // No content
  }

  @Get(':id')
  findOne(@Res() res: Response) {
    return res.status(200).json({ data: {} });
  }
}
```

### Headers

```typescript
@Controller('users')
export class UsersController {
  @Get(':id')
  @Header('Cache-Control', 'max-age=3600')
  @Header('X-Custom-Header', 'value')
  findOne() {
    return { data: {} };
  }

  @Get('download')
  @Res({ passthrough: true })
  download(@Res() res: Response) {
    res.set('Content-Disposition', 'attachment; filename="file.pdf"');
    return fs.createReadStream('file.pdf');
  }
}
```

### Redirect

```typescript
@Controller('users')
export class UsersController {
  @Get('redirect')
  @Redirect('https://example.com', 301)
  redirect() {
    // Redirects to example.com
  }

  @Get('redirect/:id')
  @Redirect()
  redirectById(@Param('id') id: string) {
    return {
      url: `https://example.com/users/${id}`,
      statusCode: 302,
    };
  }
}
```

---

## 🎯 Versioning

```typescript
// main.ts
import { VersioningType } from '@nestjs/common';

app.enableVersioning({
  type: VersioningType.URI,
  defaultVersion: '1',
});

// Controller
@Controller({ path: 'users', version: '1' })
export class UsersControllerV1 {
  @Get()
  findAll() {
    return { version: 'v1' };
  }
}

@Controller({ path: 'users', version: '2' })
export class UsersControllerV2 {
  @Get()
  findAll() {
    return { version: 'v2', newField: 'value' };
  }
}
```

---

## 🏷️ Host Decorator

```typescript
@Controller({ path: 'users', host: 'api.example.com' })
export class ApiController {
  @Get()
  findAll() {}
}

@Controller({ path: 'users', host: 'admin.example.com' })
export class AdminController {
  @Get()
  findAll() {}
}
```

---

## 🔗 Связанные заметки

- [[NestJS-MOC]] — NestJS индекс
- [[NestJS-Modules]] — Модули
- [[NestJS-Guards]] — Guards
- [[NestJS-DTO-Validation]] — DTO & Validation
