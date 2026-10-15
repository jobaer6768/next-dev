# Reward Course 02 — NestJS Backend

> Reward module. Builds on Mission 2 (scalable backends) and Mission 4 (SaaS product). Estimated effort: 14–18 hours.

---

## Why this module matters

In Mission 2 you built a backend with raw Express. In Mission 4 you shipped a multi-tenant SaaS. Somewhere in there, you probably noticed that you kept re-implementing the same things: a router, a request validator, a JWT guard, an error handler, a config loader, a DI container (or the lack of one, which is worse).

NestJS is what happens when an opinionated framework says: stop reinventing this, and start using the version 1,000,000 engineers have battle-tested. It is TypeScript-first, decorator-driven, and structurally similar to Angular — which means the moment you learn it, you have also learned the architectural vocabulary of a huge slice of the industry.

This module is not "Nest vs Express." It is "Express, with a backbone." We will take the layered architecture from Mission 4 and put it on rails (literally — `nest new` gives you rails).

---

## Learning objectives

By the end of this module you will be able to:

1. Explain why NestJS exists, and what problems it solves that Express does not.
2. Create a new Nest project and read its structure.
3. Organize code into modules, controllers, and services.
4. Use the IoC container for dependency injection.
5. Write custom providers (factory, value, class, alias).
6. Build routes, request handlers, and DTOs.
7. Compose middleware, pipes, guards, and interceptors.
8. Centralize error handling with exception filters.
9. Validate request bodies with `class-validator` (the Zod-of-Nest).
10. Load and validate environment variables with `@nestjs/config`.
11. Authenticate requests with JWT (`@nestjs/jwt` + `passport-jwt`).
12. Integrate Prisma as a data layer.
13. Apply NestJS best practices (folder structure, DI, testing).
14. Refactor a slice of the Mission 4 SaaS into NestJS as a worked example.

---

## 1. Introduction to NestJS

NestJS is a **server-side framework** for Node.js built by Kamil Myśliwiec in 2017. It is written in TypeScript, uses decorators (à la Angular), and is built on top of Express by default (with Fastify as an opt-in alternative).

Three things define it:

1. **TypeScript-first.** Decorators, types, generics are not optional. You write less boilerplate per request because the types do work.
2. **Opinionated architecture.** The framework decides the folder structure, the layering, the lifecycle. This is a feature: it makes every Nest codebase feel familiar.
3. **A real IoC container.** Instead of `app.get(UserService)` and circular `require`s, you declare what depends on what and the container wires it up.

```ts
// src/users/users.controller.ts
@Controller('users')
export class UsersController {
  constructor(private readonly users: UsersService) {}

  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.users.findById(id);
  }
}
```

That is the entire controller. The framework reads the decorators, registers the route, injects the service, parses the path param with `ParseIntPipe`, and returns the JSON. Five lines. Compare to the equivalent in raw Express (router, parser, try/catch, res.json).

---

## 2. Why NestJS

The argument is not "Nest is faster than Express." It is not. The argument is **"Nest gives you the architecture for free, and that architecture is the one every senior engineer wants."**

What you get out of the box:

| Concern | Express (raw) | NestJS |
|---|---|---|
| DI container | Build your own (or skip it) | Built-in |
| Validation | Wire Zod manually | `class-validator` via `ValidationPipe` |
| Error handling | Try/catch everywhere | Exception filters |
| Auth/guards | Middleware soup | First-class guards |
| Modular boundaries | Suggest with folders | Enforced by modules |
| Testing | Your problem | `Test.createTestingModule` |
| Docs / hireability | Express is universal | Nest is the framework 60% of TS backends now use |

**The same layered architecture from Mission 4** — Controller → Service → Repository — is the *only* way to write Nest. The framework literally won't let you put DB calls in a controller without you doing something weird.

The cost: more abstraction, more decorators, a learning curve. The payoff: every Nest codebase is navigable in 15 minutes.

---

## 3. Installing and Creating a NestJS Project

```bash
npm i -g @nestjs/cli
nest new my-saas --package-manager npm --strict
cd my-saas
npm run start:dev
```

This gives you:

```text
my-saas/
├── src/
│   ├── app.controller.ts
│   ├── app.controller.spec.ts
│   ├── app.module.ts
│   ├── app.service.ts
│   └── main.ts
├── test/
├── nest-cli.json
├── package.json
└── tsconfig.json
```

`npm run start:dev` boots a hot-reloading server on `:3000`. You are running.

---

## 4. Project Structure & Architecture

The unit of organization is the **module**. A module bundles a controller, a service, its DTOs, and its tests. The root `AppModule` imports feature modules; feature modules import each other only through their exported services.

```text
src/
├── app.module.ts          ← root
├── main.ts                ← bootstrap
├── common/                ← shared pipes, filters, decorators
│   ├── filters/
│   ├── guards/
│   ├── pipes/
│   └── decorators/
├── config/                ← env loading & validation
│   └── configuration.ts
├── prisma/                ← the data layer
│   ├── prisma.module.ts
│   └── prisma.service.ts
└── users/                 ← a feature
    ├── users.module.ts
    ├── users.controller.ts
    ├── users.service.ts
    ├── users.repository.ts
    └── dto/
        ├── create-user.dto.ts
        └── update-user.dto.ts
```

A module that wants to use `UsersService` must import `UsersModule` and have `UsersService` in `UsersModule`'s `exports` array. This is the dependency direction made explicit.

---

## 5. Modules, Controllers, Services

### Module

```ts
@Module({
  imports: [PrismaModule],
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

### Controller

The HTTP layer. Methods map to routes via decorators.

```ts
@Controller('users')
export class UsersController {
  constructor(private readonly users: UsersService) {}

  @Post()
  create(@Body() dto: CreateUserDto) {
    return this.users.create(dto);
  }

  @Get()
  list(@Query() q: ListUsersQuery) {
    return this.users.list(q);
  }

  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.users.findById(id);
  }
}
```

### Service

The business logic. Pure TypeScript, no HTTP.

```ts
@Injectable()
export class UsersService {
  constructor(private readonly prisma: PrismaService) {}

  create(dto: CreateUserDto) {
    return this.prisma.user.create({ data: dto });
  }

  findById(id: number) {
    return this.prisma.user.findUnique({ where: { id } });
  }
}
```

The controller is thin. The service is thick. This is the layering.

---

## 6. Dependency Injection (the IoC container)

Nest's DI container is a runtime map of *tokens* to *providers*. When a class declares a constructor parameter, the container resolves it by TypeScript metadata.

```ts
constructor(
  private readonly users: UsersService,         // class token
  private readonly cfg: ConfigService,          // class token
  @Inject('MAILER') private readonly mailer: Mailer, // string token
) {}
```

The container does this in 3 phases:

1. **Module loading** — Nest reads `@Module({ providers: [...] })` and registers them.
2. **Resolution** — when a class is instantiated, its constructor params are looked up.
3. **Instantiation** — singletons by default, instantiated lazily on first use.

**Why this matters:** you can swap `UsersService` for a `MockUsersService` in tests by overriding the provider, without touching `UsersController`.

```ts
const moduleRef = await Test.createTestingModule({
  controllers: [UsersController],
  providers: [
    { provide: UsersService, useValue: mockUsersService },
  ],
}).compile();
```

This is the unit test pattern. One line swaps the entire data layer.

---

## 7. Providers & Custom Providers

The default provider is a class. But you can also provide values, factories, and aliases.

```ts
@Module({
  providers: [
    // 1. class (shorthand)
    UsersService,

    // 2. value
    { provide: 'API_KEY', useValue: process.env.API_KEY },

    // 3. factory (lazy, async)
    {
      provide: 'STRIPE_CLIENT',
      useFactory: (cfg: ConfigService) =>
        new Stripe(cfg.get('STRIPE_SECRET')),
      inject: [ConfigService],
    },

    // 4. alias
    { provide: 'Logger', useExisting: MyLogger },
  ],
})
export class PaymentsModule {}
```

**Factory providers** are how you connect to a DB on boot, set up a Redis client, or instantiate a 3rd-party SDK exactly once. They run *before* any controller handles a request.

---

## 8. Routing & Request Handling

Decorators map HTTP verbs and route params:

```ts
@Get()              // GET /users
@Post()             // POST /users
@Put(':id')         // PUT /users/:id
@Patch(':id')       // PATCH /users/:id
@Delete(':id')      // DELETE /users/:id

// Wildcards and dynamic paths
@Get('me')          // GET /users/me  (declared BEFORE :id to win)
@Get(':id')
```

Param decorators:

```ts
@Param('id') id: string
@Query() query: ListUsersQuery
@Body() dto: CreateUserDto
@Headers('authorization') auth: string
@Req() req: Request
@Res() res: Response   // escape hatch
```

The idiomatic Nest style: take the data you need (typed DTOs) and return the value. Let the framework serialize it. Don't touch `res` unless you must.

---

## 9. Middleware, Pipes, Guards, and Interceptors

These are the four "cross-cutting" tools. They run in a fixed order on every request:

```text
Middleware  →  Guards  →  Interceptors (before)  →  Pipe  →  Controller  →  Interceptors (after)
```

### Middleware

Runs first. Used for logging, request ID, body parsing, CORS. Express-style `(req, res, next)`.

```ts
@Injectable()
export class RequestIdMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    req.id = req.header('X-Request-Id') ?? randomUUID();
    res.setHeader('X-Request-Id', req.id);
    next();
  }
}
```

### Pipes

Transform or validate input. The two built-in modes are *transformation* (`ParseIntPipe`) and *validation* (`ValidationPipe` with class-validator).

```ts
@Param('id', ParseIntPipe) id: number
```

### Guards

Return `true` / `false`. Use for auth and authorization.

```ts
@Injectable()
export class JwtAuthGuard implements CanActivate {
  canActivate(ctx: ExecutionContext): boolean {
    const req = ctx.switchToHttp().getRequest();
    return !!req.user; // set by passport-jwt
  }
}

@UseGuards(JwtAuthGuard, RolesGuard)
@Roles('admin')
@Delete(':id')
remove(@Param('id', ParseIntPipe) id: number) { ... }
```

### Interceptors

Wrap the handler. Used for logging, caching, transforming the response, timing.

```ts
@Injectable()
export class TimingInterceptor implements NestInterceptor {
  intercept(ctx: ExecutionContext, next: CallHandler) {
    const start = Date.now();
    return next.handle().pipe(
      tap(() => log(`request took ${Date.now() - start}ms`)),
    );
  }
}
```

`tap` lets you do something on success without changing the response. `map` lets you change it.

---

## 10. Exception Filters & Error Handling

Throw `HttpException` or a subclass (`NotFoundException`, `BadRequestException`, `UnauthorizedException`, …). The framework turns it into a JSON response with the right status code. Custom filters let you control the shape.

```ts
@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const res = ctx.getResponse<Response>();
    const status = exception.getStatus();
    const body = exception.getResponse();

    res.status(status).json({
      statusCode: status,
      path: ctx.getRequest().url,
      timestamp: new Date().toISOString(),
      ...(typeof body === 'object' ? body : { message: body }),
    });
  }
}
```

Register globally in `main.ts`:

```ts
app.useGlobalFilters(new HttpExceptionFilter());
```

This is where you turn raw stack traces into safe, structured error responses.

---

## 11. Request Validation with class-validator

The Nest equivalent of Zod. Decorators on a DTO class, validated by the global `ValidationPipe`.

```ts
// dto/create-user.dto.ts
import { IsEmail, IsString, MinLength, IsOptional, IsEnum } from 'class-validator';

export class CreateUserDto {
  @IsEmail()
  email!: string;

  @IsString()
  @MinLength(8)
  password!: string;

  @IsOptional()
  @IsEnum(Role)
  role?: Role = Role.MEMBER;
}
```

```ts
// main.ts
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,           // strip unknown fields
  forbidNonWhitelisted: true,// 400 if unknown
  transform: true,           // class-transformer: plain → class
}));
```

Now any controller that takes `@Body() dto: CreateUserDto` gets a validated, typed object — or the framework returns 400 before the handler runs.

**Zod or class-validator?** Use class-validator when you're in Nest. Use Zod for ad-hoc validation outside Nest. Both are fine; mixing is annoying.

---

## 12. Configuration & Environment Variables

`@nestjs/config` wraps `dotenv` with validation.

```ts
// config/configuration.ts
export const configuration = () => ({
  port: parseInt(process.env.PORT ?? '3000', 10),
  database: {
    url: process.env.DATABASE_URL!,
  },
  jwt: {
    secret: process.env.JWT_SECRET!,
    expiresIn: '1h',
  },
  stripe: {
    secretKey: process.env.STRIPE_SECRET_KEY!,
  },
});
```

```ts
// app.module.ts
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      load: [configuration],
      validate: validateEnv,    // joi or class-validator
    }),
  ],
})
export class AppModule {}
```

Inject `ConfigService` anywhere:

```ts
constructor(private readonly cfg: ConfigService) {}

@Post('login')
login() {
  const secret = this.cfg.get<string>('jwt.secret');
}
```

The `validate` step fails fast at boot if `JWT_SECRET` is missing — much better than crashing on the first login.

---

## 13. Authentication with JWT (overview)

The full pattern is `@nestjs/passport` + `passport-jwt`. You define a strategy, register it, and guard routes with `@UseGuards(AuthGuard('jwt'))`.

```ts
// auth/jwt.strategy.ts
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(cfg: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: cfg.get<string>('jwt.secret')!,
    });
  }

  async validate(payload: JwtPayload) {
    return { id: payload.sub, email: payload.email, role: payload.role };
  }
}
```

```ts
// auth/auth.controller.ts
@Post('login')
async login(@Body() dto: LoginDto) {
  const user = await this.users.verifyPassword(dto);
  return { accessToken: this.jwt.sign({ sub: user.id, role: user.role }) };
}
```

```ts
// anywhere
@UseGuards(AuthGuard('jwt'))
@Get('me')
me(@Req() req: Request) { return req.user; }
```

This is a 20-line overview; the real implementation is in the worked example at the end of this module.

---

## 14. Database Integration with Prisma (overview)

```bash
npm i @prisma/client
npx prisma init
```

```ts
// prisma/prisma.service.ts
@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit() { await this.$connect(); }
  async onModuleDestroy() { await this.$disconnect(); }
}
```

```ts
// prisma/prisma.module.ts
@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

Now any service can `constructor(private prisma: PrismaService) {}` and use it. The `@Global()` decorator means you don't have to re-import `PrismaModule` in every feature module.

---

## 15. NestJS Best Practices

1. **One module per bounded context.** `UsersModule`, `BillingModule`, `ProjectsModule`. Not one giant `AppModule`.
2. **Controllers stay thin.** Validation, parsing, auth, response. No business logic.
3. **Services own the rules.** They orchestrate repositories, call other services, enforce invariants.
4. **Use DTOs at the boundary, domain types inside.** Don't leak `class-validator` decorators into your service layer.
5. **Test with `Test.createTestingModule`.** Override providers; assert on the service, not on the HTTP layer, for unit tests. Use `supertest` for e2e.
6. **Don't put logic in `main.ts`.** It should be ~20 lines: bootstrap, global pipes, global filters, listen.
7. **Avoid circular dependencies.** If `A` needs `B` and `B` needs `A`, you have a design problem. Use `forwardRef` only as a last resort.
8. **Use a logger, not `console.log`.** `Logger` from `@nestjs/common` with context.
9. **Document the API.** Swagger via `@nestjs/swagger` decorators on your DTOs.
10. **Be explicit about lifecycle.** `OnModuleInit`, `OnModuleDestroy`, `OnApplicationBootstrap`. Connect to DB on init, disconnect on destroy.

---

## 16. Worked Example: Refactor the Mission 4 SaaS to NestJS

**Goal:** take the SaaS from Mission 4 (Express + Prisma + JWT) and refactor `auth`, `workspaces`, and `projects` into Nest modules, *without changing behavior*. This is a structural refactor.

### Step 1 — Bootstrap

```bash
nest new my-saas-backend
cd my-saas-backend
npm i @prisma/client @nestjs/config @nestjs/jwt @nestjs/passport passport passport-jwt bcrypt
npm i -D prisma @types/passport-jwt
npx prisma init
```

Copy the existing `schema.prisma` from Mission 4.

### Step 2 — Common building blocks

```ts
// common/filters/http-exception.filter.ts  (as in §10)
// common/guards/roles.guard.ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}
  canActivate(ctx: ExecutionContext) {
    const required = this.reflector.getAllAndOverride<Role[]>('roles', [
      ctx.getHandler(), ctx.getClass(),
    ]);
    if (!required) return true;
    const { user } = ctx.switchToHttp().getRequest();
    return required.includes(user.role);
  }
}

// common/decorators/roles.decorator.ts
export const Roles = (...roles: Role[]) => SetMetadata('roles', roles);
```

### Step 3 — The `auth` module

```ts
// auth/auth.module.ts
@Module({
  imports: [UsersModule, PassportModule, JwtModule.registerAsync({
    inject: [ConfigService],
    useFactory: (cfg: ConfigService) => ({
      secret: cfg.get<string>('jwt.secret'),
      signOptions: { expiresIn: '1h' },
    }),
  })],
  controllers: [AuthController],
  providers: [AuthService, JwtStrategy],
  exports: [AuthService],
})
export class AuthModule {}
```

```ts
// auth/auth.controller.ts
@Controller('auth')
export class AuthController {
  constructor(private readonly auth: AuthService) {}

  @Post('register')
  register(@Body() dto: RegisterDto) { return this.auth.register(dto); }

  @Post('login')
  @HttpCode(200)
  login(@Body() dto: LoginDto) { return this.auth.login(dto); }
}
```

```ts
// auth/auth.service.ts
@Injectable()
export class AuthService {
  constructor(
    private readonly users: UsersService,
    private readonly jwt: JwtService,
  ) {}

  async register(dto: RegisterDto) {
    const hash = await bcrypt.hash(dto.password, 12);
    return this.users.create({ ...dto, password: hash });
  }

  async login(dto: LoginDto) {
    const user = await this.users.findByEmail(dto.email);
    if (!user || !(await bcrypt.compare(dto.password, user.password))) {
      throw new UnauthorizedException('invalid credentials');
    }
    return { accessToken: this.jwt.sign({ sub: user.id, role: user.role }) };
  }
}
```

### Step 4 — The `workspaces` and `projects` modules

Same pattern. Each has a controller, a service, a repository (or uses Prisma directly), and DTOs. The `ProjectsController` is guarded:

```ts
@UseGuards(AuthGuard('jwt'), RolesGuard)
@Controller('projects')
export class ProjectsController {
  constructor(private readonly projects: ProjectsService) {}

  @Post()
  create(@Req() req: Request, @Body() dto: CreateProjectDto) {
    return this.projects.create({ ...dto, ownerId: req.user.id });
  }
}
```

### Step 5 — Wire it up

```ts
// app.module.ts
@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true, load: [configuration] }),
    PrismaModule,
    AuthModule,
    UsersModule,
    WorkspacesModule,
    ProjectsModule,
  ],
})
export class AppModule {}
```

### Step 6 — Tests

```ts
// auth/auth.service.spec.ts
describe('AuthService', () => {
  let service: AuthService;
  let users: UsersService;
  let jwt: JwtService;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        AuthService,
        { provide: UsersService, useValue: { create: vi.fn(), findByEmail: vi.fn() } },
        { provide: JwtService, useValue: { sign: vi.fn().mockReturnValue('token') } },
      ],
    }).compile();
    service = module.get(AuthService);
    users = module.get(UsersService);
    jwt = module.get(JwtService);
  });

  it('logs in with valid credentials', async () => {
    vi.spyOn(users, 'findByEmail').mockResolvedValue({ id: 1, password: 'h' } as any);
    vi.spyOn(bcrypt, 'compare').mockResolvedValue(true as any);
    await expect(service.login({ email: 'a@b.c', password: 'p' })).resolves.toEqual({ accessToken: 'token' });
  });
});
```

### What you learned

- The same routes, the same DB, the same business rules — but now the dependencies are explicit, the boundaries are enforced, and every test runs in 50ms because the IoC container swaps real Prisma for a mock.

That is the entire value proposition of Nest, in one PR.

---

## Common mistakes

1. **Putting business logic in controllers.** The controller is a translator from HTTP to service calls. That's it.
2. **Forgetting `whitelist: true` in `ValidationPipe`.** Without it, unknown fields are silently accepted.
3. **Not validating env vars at boot.** A missing `JWT_SECRET` should crash on startup, not on the first login.
4. **Circular module dependencies.** Symptom: `Nest can't resolve dependencies`. Fix: rethink the boundary, don't reach for `forwardRef`.
5. **One `AppModule` with everything.** Breaks the point of modules. Each feature gets its own.
6. **Overriding providers in production.** `useValue` belongs in tests, not in `main.ts`.
7. **Not using DTOs.** Taking `@Body() body: any` and validating by hand defeats the framework.
8. **Bypassing `Logger` for `console.log`.** Loses context, breaks structured logging.
9. **Reaching for `Res()` to set status codes.** Throw the right exception instead.
10. **Not reading the lifecycle docs.** `OnModuleInit` exists for a reason — use it for DB connections.

---

## Exercise

**Refactor the Mission 4 SaaS to NestJS.**

1. Create a new `apps/api-nest` workspace.
2. Move `auth`, `workspaces`, `projects`, and `users` into Nest modules.
3. Add `class-validator` DTOs for every request body.
4. Add a global `ValidationPipe` with `whitelist: true, forbidNonWhitelisted: true`.
5. Add a global `HttpExceptionFilter` that returns `{ statusCode, path, timestamp, message }`.
6. Configure env via `@nestjs/config` with a Joi or class-validator schema.
7. Add JWT auth with `@nestjs/jwt` + `@nestjs/passport`.
8. Wire Prisma through a global `PrismaModule`.
9. Write at least 3 unit tests using `Test.createTestingModule` and 1 e2e test with `supertest`.
10. Confirm the integration test from Mission 4 still passes against the new backend.

**Stretch:** add a `BullModule` queue for project-export jobs (you'll need a Redis instance). Write a worker that listens for `export.project` and writes a CSV to S3.

Submit: the PR, the test output, a 2-paragraph writeup on what felt better and what felt worse versus raw Express.

---

## AI Industry Note (2026)

In 2026, the demand for NestJS-style opinionated backends is higher than ever, not lower. The reason: AI coding tools (Claude Code, Cursor) generate boilerplate much faster in opinionated frameworks. When the framework dictates the structure, the model gets it right the first time. The result: a senior engineer using Cursor + Nest is 3–5x more productive than a senior engineer using Cursor + raw Express, because the framework's opinions are exactly the prompt context the model needs.

The meta-skill: **learn a framework deeply enough that you can describe its conventions in 200 words.** That description is your prompt. The engineer with the best prompt wins.

---

*Next: Reward Course 03 — Testing & Quality, where we cover Vitest, Supertest, and the testing pyramid.*
