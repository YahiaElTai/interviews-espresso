# NestJS

> **25 questions** — 8 theory, 13 practical, 4 experience

- Core building blocks: modules, controllers, providers, and enforced structure
- Dependency injection: DI containers, provider scopes (DEFAULT, REQUEST, TRANSIENT), custom providers (useClass, useFactory, useValue, useExisting)
- NestJS vs Express/Fastify: when the framework overhead is worth it, Fastify adapter tradeoffs
- Request lifecycle: middleware → guards → interceptors → pipes → handler → exception filters
- Exception filters: built-in HttpException hierarchy, custom exception filters for domain errors, global vs controller-scoped filters, consistent error response formatting
- Module encapsulation: exports, dynamic modules (forRoot/forRootAsync), circular dependencies
- Guards: canActivate lifecycle, Passport JWT authentication, role-based authorization with custom decorators
- Pipes: validation with class-validator/class-transformer, built-in transform pipes, global pipe configuration
- Interceptors: Observable-based pipeline (tap/map), response transformation, timeout handling, caching with @nestjs/cache-manager
- Configuration management: @nestjs/config with validation and DI integration
- Structuring NestJS for clean architecture: mapping modules to domain boundaries, keeping framework at the edges
- Lifecycle hooks: onModuleInit, onModuleDestroy, enableShutdownHooks, graceful shutdown with K8s preStop
- Testing: Test.createTestingModule, mocking with overrideProvider, integration testing with supertest

---

## Foundational

<details>
<summary>1. What are the core building blocks of NestJS (modules, controllers, providers) and why does it enforce this structure — what problems does opinionated structure solve compared to Express's flexibility, and how do these pieces compose to form a complete application?</summary>

**Three core building blocks:**

- **Modules** (`@Module()`) — organizational units that group related functionality. Every app has a root `AppModule`. Modules declare which controllers handle routes, which providers contain business logic, which providers to export for other modules to use, and which other modules to import.
- **Controllers** (`@Controller()`) — handle incoming HTTP requests and return responses. They define routes via decorators (`@Get()`, `@Post()`, etc.) and delegate business logic to providers. Controllers should be thin — route definition and request/response mapping only.
- **Providers** (`@Injectable()`) — any class that can be injected as a dependency. Services, repositories, factories, helpers — all are providers. They contain the actual business logic and are managed by the NestJS DI container.

**How they compose:**

```typescript
@Module({
  imports: [DatabaseModule],       // other modules this module depends on
  controllers: [UsersController],  // route handlers
  providers: [UsersService],       // injectable business logic
  exports: [UsersService],         // what other modules can use
})
export class UsersModule {}
```

**Why enforce structure — what problems it solves:**

With Express, every team invents its own project structure. Two Express codebases from different teams look completely different — different folder layouts, different patterns for middleware, different approaches to dependency management. This creates real costs:

- **Onboarding**: New team members must learn the custom architecture before contributing.
- **Consistency**: Without conventions, code organization degrades as the team grows. Business logic leaks into route handlers, dependencies get passed around manually.
- **Testability**: Without DI, mocking dependencies in Express requires manual wiring or tools like `proxyquire`.

NestJS solves this by making structure a framework concern rather than a team decision. Every NestJS project has the same patterns — modules for boundaries, controllers for HTTP, providers for logic, DI for wiring. The tradeoff is less flexibility and more boilerplate, but for teams beyond 2-3 developers, that consistency pays for itself.

</details>

<details>
<summary>2. How does the NestJS request lifecycle work — walk through the full pipeline (middleware → guards → interceptors → pipes → handler → interceptors → exception filters), explain what each layer is responsible for, and why does NestJS separate these concerns instead of using a single middleware chain like Express?</summary>

**Full request lifecycle in order:**

1. **Middleware** — Runs first, identical to Express middleware. Has access to `req`, `res`, `next()`. Used for low-level concerns: body parsing, CORS, request logging, helmet. No access to NestJS execution context (doesn't know which controller/handler will run).

2. **Guards** — Decide whether the request should proceed (`canActivate()` returns boolean). Used for authentication and authorization. Has access to `ExecutionContext`, so it knows which controller and handler the request targets — this is what makes metadata-based checks (like `@Roles()`) possible.

3. **Interceptors (before)** — Wrap the handler execution using RxJS Observables. The "before" part runs before the handler. Used for: logging start time, setting up correlation IDs, binding request context.

4. **Pipes** — Transform and validate input data. Run on each parameter decorated with `@Body()`, `@Param()`, `@Query()`, etc. The `ValidationPipe` with class-validator runs here. If validation fails, the request never reaches the handler.

5. **Handler** — Your actual controller method. By this point, the request is authenticated, authorized, and input is validated and transformed.

6. **Interceptors (after)** — The response flows back through interceptors via the Observable pipeline. Used for: response transformation (wrapping in `{ data: ... }`), logging duration, caching.

7. **Exception filters** — Catch any unhandled exceptions thrown at any point in the pipeline. Map exceptions to HTTP responses. The built-in filter handles `HttpException` subclasses; custom filters handle domain errors.

**Why separate layers instead of Express's single middleware chain?**

Express middleware is a flat chain where everything — auth, validation, logging, error handling — uses the same `(req, res, next)` signature. This means:

- Auth middleware can't inspect route metadata (it doesn't know which handler will run)
- Validation middleware can't access the DTO class definition for the target handler
- There's no standard "after handler" hook — you'd need to monkey-patch `res.json()`
- Error handling relies on convention (error middleware with 4 params) rather than explicit types

NestJS's layered approach gives each concern the right API. Guards get `ExecutionContext` with reflector access. Pipes get the parameter metadata. Interceptors get the Observable stream for before/after logic. Each layer does one thing with the right tools.

</details>

## Conceptual Depth

<details>
<summary>3. How does NestJS dependency injection work — what does the DI container manage, what are the provider scopes (DEFAULT singleton, REQUEST per-request, TRANSIENT per-injection), when would you use each scope, and what goes wrong when you inject a REQUEST-scoped provider into a DEFAULT-scoped one?</summary>

**How the DI container works:**

At bootstrap, NestJS scans all modules, reads `@Injectable()` metadata, resolves the dependency graph, and instantiates providers in the correct order. When a controller or service declares a constructor parameter typed as `UsersService`, the container injects the managed instance automatically. The container handles the full lifecycle — creation, injection, and destruction.

**Three provider scopes:**

| Scope | Lifetime | Instance count |
|-------|----------|----------------|
| `DEFAULT` | Application lifetime | 1 (singleton) |
| `REQUEST` | Per incoming request | 1 per request |
| `TRANSIENT` | Per injection point | 1 per consumer |

```typescript
@Injectable({ scope: Scope.REQUEST })
export class RequestContextService {
  constructor(@Inject(REQUEST) private request: Request) {}
  // Fresh instance per request — can safely store request-specific data
}
```

**When to use each:**

- **DEFAULT (singleton)** — The vast majority of providers. Stateless services, repositories, utility classes. One instance shared across all requests. Best for performance since there's no instantiation overhead per request.
- **REQUEST** — When you need request-specific state accessible deep in the call chain without passing it through every function. Examples: multi-tenant context (current tenant from JWT), request-scoped logging with correlation ID, per-request caching.
- **TRANSIENT** — When each consumer needs its own isolated instance. Rare in practice. Example: a logger that automatically tags output with the consuming class name.

**The scope bubble-up problem:**

If you inject a REQUEST-scoped provider into a DEFAULT-scoped (singleton) provider, the singleton would capture one instance of the request-scoped provider and reuse it for all requests — breaking request isolation. NestJS prevents this by **forcing scope propagation upward**: any provider that depends on a REQUEST-scoped provider automatically becomes REQUEST-scoped too.

This means a single REQUEST-scoped provider deep in your dependency tree can turn a large portion of your providers into request-scoped ones, creating a new instance chain for every request. This has real performance implications — it's why you should use REQUEST scope sparingly and consider alternatives like `AsyncLocalStorage` (via `cls-hooked` or Node's built-in) for request context propagation without scope bubbling.

</details>

<details>
<summary>4. How do custom providers (useClass, useFactory, useValue, useExisting) extend basic class injection in NestJS — when would you use each type, and how do they enable swapping implementations for testing or environment-specific behavior?</summary>

By default, NestJS providers map a class token to itself: `providers: [UsersService]` is shorthand for `{ provide: UsersService, useClass: UsersService }`. Custom providers break this 1:1 mapping for flexibility.

**The four custom provider types:**

**`useClass`** — Swap the implementation class behind a token. The container instantiates the class and resolves its dependencies.

```typescript
// Swap storage backend based on environment
{
  provide: StorageService,
  useClass: process.env.STORAGE === 's3' ? S3StorageService : LocalStorageService,
}
```

**`useFactory`** — Create a provider using a factory function. Supports async and can inject other providers as dependencies. Most flexible option.

```typescript
{
  provide: 'DATABASE_CONNECTION',
  useFactory: async (config: ConfigService) => {
    const client = new Client({ host: config.get('DB_HOST') });
    await client.connect();
    return client;
  },
  inject: [ConfigService], // dependencies passed to factory
}
```

**`useValue`** — Provide a static value directly. No instantiation, no DI resolution. Good for constants, mock objects in tests, or pre-configured instances.

```typescript
{
  provide: 'API_KEY',
  useValue: 'sk-1234567890',
}

// In tests — inject a mock directly
{
  provide: UsersService,
  useValue: { findById: jest.fn().mockResolvedValue(mockUser) },
}
```

**`useExisting`** — Alias one token to another. Both tokens resolve to the same instance. Useful when you want to expose a provider under an interface token while keeping the concrete class injectable too.

```typescript
{
  provide: 'AliasedLogger',
  useExisting: LoggerService, // resolves to the same LoggerService instance
}
```

**How this enables testing and environment switching:**

The key insight is that consumer code depends on the **token**, not the **implementation**. A service that injects `StorageService` doesn't know or care whether it's getting `S3StorageService` or `MockStorageService`. In tests, you swap the implementation via `overrideProvider`:

```typescript
const module = await Test.createTestingModule({
  providers: [UsersService, StorageService],
})
  .overrideProvider(StorageService)
  .useValue(mockStorage)
  .compile();
```

</details>

<details>
<summary>5. When is NestJS worth its framework overhead compared to Express or Fastify — what specific problems does NestJS solve that justify the learning curve and abstraction cost, what are the tradeoffs of using the Fastify adapter instead of Express, and when should you choose plain Express or Fastify instead?</summary>

**When NestJS is worth it:**

- **Team size > 2-3 developers** — Enforced structure means less time debating project organization and easier onboarding. Everyone knows where things go.
- **Long-lived production services** — DI, module boundaries, and lifecycle hooks pay off over months/years of maintenance. Testing is straightforward from day one.
- **Complex domain logic** — Guards, pipes, interceptors, and exception filters give you clean separation of cross-cutting concerns that would otherwise pollute your route handlers.
- **Microservice architectures** — Built-in support for transport layers (TCP, Redis, NATS, gRPC, Kafka) via `@nestjs/microservices` means you're not wiring up message consumers from scratch.

**When plain Express or Fastify is better:**

- **Simple APIs or Lambda functions** — NestJS's bootstrap time and abstraction overhead isn't justified for a 5-endpoint service or a serverless function (though NestJS does have a serverless adapter).
- **Maximum performance sensitivity** — NestJS adds latency through its decorator resolution, DI container lookups, and interceptor/pipe/guard chain. For hot paths where microseconds matter, the abstraction cost is measurable.
- **Small team, small scope** — If 1-2 developers are building something straightforward, Express's flexibility with a lightweight custom structure is faster to set up and easier to reason about.

**Fastify adapter tradeoffs:**

NestJS runs on Express by default but supports Fastify via `@nestjs/platform-fastify`.

| Aspect | Express adapter | Fastify adapter |
|--------|----------------|-----------------|
| Performance | Baseline | ~2-3x faster in benchmarks (schema-based serialization, efficient routing) |
| Ecosystem | Massive — every Express middleware works | Growing — but some Express middleware needs wrappers or alternatives |
| Community/examples | Most NestJS tutorials use Express | Fewer examples, occasional compatibility issues |
| Type safety | Loose (`req: any` patterns common) | Better — Fastify has built-in schema validation and TypeScript support |

**Practical recommendation:** Start with Express adapter (the default) unless you have a measured performance need. Switching adapters later is mostly a find-and-replace on platform-specific imports (`@Req()` type changes from Express to Fastify), but middleware compatibility can require work.

</details>

<details>
<summary>6. How does module encapsulation work in NestJS — what do exports control, how do dynamic modules (forRoot/forRootAsync) enable configurable shared modules, and how do you detect and resolve circular dependency issues between modules?</summary>

**Module encapsulation and exports:**

By default, providers declared in a module are **private** to that module. Other modules cannot inject them even if they import the module. The `exports` array explicitly controls what's available to importing modules — it's the module's public API.

```typescript
@Module({
  providers: [UsersService, UsersRepository], // both available within this module
  exports: [UsersService], // only UsersService is available to importers
  // UsersRepository stays private — an implementation detail
})
export class UsersModule {}
```

This encapsulation is what makes modules real boundaries rather than just folders. Importing `UsersModule` gives you `UsersService` but not `UsersRepository` — the module controls its own surface area.

**Dynamic modules — forRoot / forRootAsync:**

Static modules have hardcoded providers. Dynamic modules accept configuration at import time and return a module definition dynamically.

- **`forRoot(options)`** — Accepts synchronous configuration. Used when config values are known at compile time.
- **`forRootAsync(options)`** — Accepts async factory functions that can inject other providers (like `ConfigService`). Used when config comes from environment variables, secrets managers, or other async sources.

```typescript
@Module({})
export class DatabaseModule {
  static forRootAsync(options: {
    useFactory: (...args: any[]) => DatabaseConfig | Promise<DatabaseConfig>;
    inject?: any[];
  }): DynamicModule {
    return {
      module: DatabaseModule,
      global: true, // available everywhere without explicit imports
      providers: [
        {
          provide: 'DB_CONFIG',
          useFactory: options.useFactory,
          inject: options.inject || [],
        },
        DatabaseService,
      ],
      exports: [DatabaseService],
    };
  }
}

// Usage in AppModule:
@Module({
  imports: [
    DatabaseModule.forRootAsync({
      useFactory: (config: ConfigService) => ({
        host: config.get('DB_HOST'),
        port: config.get<number>('DB_PORT'),
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AppModule {}
```

**Circular dependency detection and resolution:**

NestJS throws a clear error when it detects circular dependencies: `"A circular dependency has been detected"`. Two approaches to resolve:

1. **`forwardRef()`** — Tells the DI container to resolve the reference lazily. Works for both module-to-module and provider-to-provider circular deps.

```typescript
@Module({
  imports: [forwardRef(() => OrdersModule)],
})
export class UsersModule {}
```

2. **Restructure** — Usually the better fix. Circular dependencies often signal that shared logic should be extracted into a third module that both depend on. If `UsersModule` and `OrdersModule` depend on each other, extract the shared concept (e.g., `SharedDomainModule`) and have both import it.

`forwardRef` is a band-aid. If you find yourself using it often, your module boundaries likely need rethinking.

</details>

<details>
<summary>7. How should you structure a NestJS application for clean architecture — how do you map modules to domain boundaries, keep NestJS framework code at the edges (controllers, decorators) while domain logic stays framework-agnostic, and what's the practical boundary between "NestJS structure" and "over-abstraction"?</summary>

**Mapping modules to domain boundaries:**

Organize by business domain, not technical layer. Instead of `controllers/`, `services/`, `repositories/` folders, each module encapsulates a full vertical slice:

```
src/
  users/
    users.module.ts
    users.controller.ts      # Framework edge — HTTP concerns only
    users.service.ts          # Orchestration — calls domain logic
    users.repository.ts       # Data access abstraction
    dto/
      create-user.dto.ts      # Framework edge — validation decorators
    entities/
      user.entity.ts          # Domain — plain TypeScript, no NestJS deps
  orders/
    orders.module.ts
    orders.controller.ts
    orders.service.ts
    ...
  shared/
    shared.module.ts          # Cross-cutting: logging, config, common utilities
```

**Keeping framework at the edges:**

The principle: NestJS decorators and framework types live in controllers, DTOs, and module definitions. Domain logic (business rules, calculations, state transitions) should be plain TypeScript classes/functions with no `@Injectable()`, no `@nestjs/*` imports.

```typescript
// Domain logic — framework-agnostic, easily testable
export class OrderPricing {
  static calculateTotal(items: OrderItem[], discount?: Discount): Money {
    const subtotal = items.reduce((sum, item) => sum + item.price * item.qty, 0);
    return discount ? applyDiscount(subtotal, discount) : subtotal;
  }
}

// Service — NestJS-aware orchestration layer
@Injectable()
export class OrdersService {
  constructor(private ordersRepo: OrdersRepository) {}

  async createOrder(dto: CreateOrderDto): Promise<Order> {
    const total = OrderPricing.calculateTotal(dto.items, dto.discount);
    return this.ordersRepo.create({ ...dto, total });
  }
}
```

The service is the bridge: it's `@Injectable()` for DI, but delegates business rules to framework-agnostic domain code. The controller handles HTTP; the service orchestrates; domain classes contain pure logic.

**The practical boundary — when does this become over-abstraction?**

- **Justified**: Separating domain logic from framework code when that logic has real complexity (pricing rules, state machines, eligibility checks). These benefit from isolated unit testing without NestJS test harness.
- **Over-abstraction**: Creating repository interfaces + implementations + abstract factories for simple CRUD operations. If your "domain logic" is just `this.prisma.user.create(data)`, the extra layers add indirection without value.
- **Rule of thumb**: If a service method is essentially a pass-through to the database with no business rules, skip the domain layer abstraction. Add it when business rules emerge, not preemptively.

NestJS already provides structure. Clean architecture on top of NestJS should add clarity, not ceremony. Two layers of indirection for a CRUD endpoint is worse than one well-organized service.

</details>

<details>
<summary>8. How do NestJS lifecycle hooks enable clean startup and shutdown — what do onModuleInit and onModuleDestroy do, why is enableShutdownHooks necessary, and how do you implement graceful shutdown that works with Kubernetes preStop hooks?</summary>

**Lifecycle hooks:**

- **`onModuleInit()`** — Called once the module's dependencies have been resolved. Use it for initialization: connect to databases, warm caches, start background workers. Supports `async` — NestJS waits for the promise to resolve before proceeding to the next module.
- **`onModuleDestroy()`** — Called when the application is shutting down. Use it for cleanup: close database connections, flush logs, stop background jobs.
- **`onApplicationBootstrap()`** — Called after all modules are initialized. Good for logic that depends on the entire app being ready.
- **`beforeApplicationShutdown(signal)`** — Called before `onModuleDestroy`. Receives the signal (e.g., `SIGTERM`). Good for stopping the HTTP server from accepting new connections while existing requests drain.

**Why `enableShutdownHooks()` is necessary:**

By default, NestJS does **not** listen for OS signals (`SIGTERM`, `SIGINT`). Without enabling shutdown hooks, your `onModuleDestroy` methods never run when the process is killed — connections stay open, in-flight work is lost.

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableShutdownHooks(); // now SIGTERM/SIGINT trigger lifecycle hooks
  await app.listen(3000);
}
```

It's disabled by default because the signal listeners interfere with some development tools and testing frameworks. Always enable it in production.

**Graceful shutdown with Kubernetes:**

When K8s terminates a pod, it sends `SIGTERM` and waits `terminationGracePeriodSeconds` (default 30s) before `SIGKILL`. The problem: K8s also updates the Service endpoints, but this takes a few seconds. During that window, new requests may still route to the dying pod.

The `preStop` hook adds a delay before `SIGTERM`, giving K8s time to update routing:

```yaml
# Kubernetes deployment spec
containers:
  - name: api
    lifecycle:
      preStop:
        exec:
          command: ["sh", "-c", "sleep 5"]  # wait for endpoints to update
    terminationGracePeriodSeconds: 30
```

```typescript
@Injectable()
export class PrismaService extends PrismaClient
  implements OnModuleInit, OnModuleDestroy {

  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect(); // clean up DB connections on shutdown
  }
}
```

**What happens without proper shutdown:** In-flight database transactions are aborted, connection pools are orphaned (the database keeps connections open until timeout), WebSocket clients get disconnected without close frames, and message consumers may lose unacknowledged messages.

</details>

## Practical — Guards, Pipes & Interceptors

<details>
<summary>9. Why does NestJS use guards for authentication instead of middleware or inline controller checks — implement JWT authentication with Passport showing the AuthGuard using @nestjs/passport, the JWT strategy configuration, how to extract the user from the token and attach it to the request, and how to make specific routes public with a custom decorator</summary>

**Why guards over middleware for auth:**

Middleware runs before NestJS's routing layer — it has no access to `ExecutionContext`, so it can't inspect which controller or method will handle the request. This means middleware can't read route metadata (like `@Roles('admin')` or `@Public()`). Guards run after routing, have full `ExecutionContext` access, and can use the `Reflector` to read decorator metadata. This is exactly what auth and authorization need.

Inline controller checks (`if (!user) throw new UnauthorizedException()`) work but violate DRY — you'd repeat the same logic in every handler.

**Implementation:**

**1. JWT Strategy — extracts and validates the token:**

```typescript
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(config: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: config.get<string>('JWT_SECRET'),
    });
  }

  // Return value is attached to request.user
  async validate(payload: { sub: string; email: string; roles: string[] }) {
    return { userId: payload.sub, email: payload.email, roles: payload.roles };
  }
}
```

**2. Custom `@Public()` decorator — marks routes that skip auth:**

```typescript
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

**3. Global JWT AuthGuard — protects everything by default, respects `@Public()`:**

```typescript
import { ExecutionContext, Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { Reflector } from '@nestjs/core';
import { IS_PUBLIC_KEY } from './public.decorator';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext) {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true; // skip auth for @Public() routes
    return super.canActivate(context);
  }
}
```

**4. Register globally in the AuthModule:**

```typescript
@Module({
  imports: [
    PassportModule,
    JwtModule.registerAsync({
      useFactory: (config: ConfigService) => ({
        secret: config.get('JWT_SECRET'),
        signOptions: { expiresIn: '1h' },
      }),
      inject: [ConfigService],
    }),
  ],
  providers: [
    JwtStrategy,
    { provide: APP_GUARD, useClass: JwtAuthGuard }, // every route is protected
  ],
})
export class AuthModule {}
```

**5. Usage:**

```typescript
@Controller('users')
export class UsersController {
  @Public()
  @Post('register')
  register(@Body() dto: RegisterDto) { /* no auth needed */ }

  @Get('profile')
  getProfile(@Req() req) {
    return req.user; // { userId, email, roles } from JwtStrategy.validate()
  }
}
```

The "protect everything, opt-out with `@Public()`" pattern is safer than "protect nothing, opt-in per route" — you can't accidentally forget to add auth to a new endpoint.

</details>

<details>
<summary>10. Implement role-based authorization with a custom guard and decorator — show a @Roles() decorator, a RolesGuard that checks the user's roles against the required roles, and demonstrate how to use it on controllers and individual routes. Explain why checking permissions is better than checking role names</summary>

**1. `@Roles()` decorator — attaches required roles as metadata:**

```typescript
import { SetMetadata } from '@nestjs/common';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);
```

**2. RolesGuard — reads metadata, checks user's roles:**

```typescript
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { ROLES_KEY } from './roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    // Get roles from handler first, fall back to controller
    const requiredRoles = this.reflector.getAllAndOverride<string[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    if (!requiredRoles) return true; // no @Roles() = allow all authenticated users

    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some((role) => user.roles?.includes(role));
  }
}
```

**3. Register globally (runs after the JwtAuthGuard from question 9):**

```typescript
providers: [
  { provide: APP_GUARD, useClass: JwtAuthGuard },  // authn first
  { provide: APP_GUARD, useClass: RolesGuard },     // authz second
],
```

**4. Usage on controllers and routes:**

```typescript
@Controller('admin')
@Roles('admin') // all routes in this controller require admin
export class AdminController {
  @Get('dashboard')
  getDashboard() { /* admin only */ }

  @Roles('super-admin') // override: this specific route needs super-admin
  @Delete('users/:id')
  deleteUser(@Param('id') id: string) { /* super-admin only */ }
}

@Controller('orders')
export class OrdersController {
  @Get()
  findAll() { /* any authenticated user — no @Roles() */ }

  @Roles('manager', 'admin')
  @Post('refund')
  processRefund() { /* managers or admins */ }
}
```

**Why permissions are better than role names:**

The implementation above checks role names (`admin`, `manager`), which is simpler but creates problems at scale:

- **Role explosion**: When you need `admin-who-can-refund` vs `admin-who-can-delete-users`, you create new roles instead of composing capabilities.
- **Tight coupling**: Controller code knows about role names. If you rename `manager` to `team-lead`, you update every `@Roles()` decorator.
- **Inflexible**: What if a `support` agent should also process refunds? You must add `support` to every route that `manager` had access to.

**Permission-based approach:**

```typescript
// Instead of @Roles('admin', 'manager')
@Permissions('orders:refund')
@Post('refund')
processRefund() {}
```

Roles become mappings to permissions (`admin → [orders:*, users:*]`, `manager → [orders:refund, orders:view]`), configured in your auth system rather than hardcoded in decorators. The guard checks `user.permissions.includes('orders:refund')` regardless of which role granted it. Adding a new role or changing role capabilities requires zero code changes — just an admin configuration update.

</details>

<details>
<summary>11. Why does NestJS use pipes for validation and transformation instead of handling it in controllers — show a DTO with class-validator decorators, configure the global ValidationPipe with whitelist and transform options, demonstrate what happens with invalid input, and explain why whitelist prevents mass assignment vulnerabilities</summary>

**Why pipes instead of controller-level validation:**

Pipes run before the handler executes, on each parameter individually. This means invalid requests are rejected before your business logic runs — no need for `if (!email) throw ...` checks in every handler. Validation logic is defined once in the DTO and enforced automatically everywhere that DTO is used.

**1. DTO with class-validator decorators:**

```typescript
import { IsEmail, IsString, MinLength, IsOptional, IsEnum } from 'class-validator';

export enum UserRole {
  USER = 'user',
  ADMIN = 'admin',
}

export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(8)
  password: string;

  @IsString()
  name: string;

  @IsEnum(UserRole)
  @IsOptional()
  role?: UserRole;
}
```

**2. Global ValidationPipe configuration:**

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,            // strip properties not in DTO
      forbidNonWhitelisted: true, // throw error if extra properties are sent
      transform: true,            // auto-transform payloads to DTO class instances
    }),
  );
  await app.listen(3000);
}
```

- **`whitelist: true`** — Strips any properties not decorated in the DTO. A request with `{ email, password, name, isAdmin: true }` silently drops `isAdmin`.
- **`forbidNonWhitelisted: true`** — Instead of silently stripping, throws a 400 error listing the unexpected properties. Stricter — good for catching client bugs.
- **`transform: true`** — Converts the plain JSON object into an actual `CreateUserDto` class instance. Also converts primitive types (e.g., `@Param('id')` string to number if the parameter is typed as `number`).

**3. What happens with invalid input:**

```json
// POST /users with invalid body:
{
  "email": "not-an-email",
  "password": "short",
  "isAdmin": true
}

// Response (400 Bad Request):
{
  "statusCode": 400,
  "message": [
    "email must be an email",
    "password must be longer than or equal to 8 characters",
    "name must be a string",
    "name should not be empty",
    "property isAdmin should not exist"
  ],
  "error": "Bad Request"
}
```

**4. Why whitelist prevents mass assignment:**

Mass assignment is when an attacker sends extra properties that map to model fields the API didn't intend to expose. Classic example: sending `{ "role": "admin" }` alongside a registration form. If the controller spreads the body directly into a database create call (`this.prisma.user.create({ data: body })`), the attacker just promoted themselves.

`whitelist: true` makes this impossible — only properties with class-validator decorators on the DTO survive. Even if `role` is a valid database column, it's stripped unless the DTO explicitly declares and decorates it. This is defense in depth — the DTO becomes the contract for what the endpoint accepts.

</details>

<details>
<summary>12. Why are cross-cutting concerns like logging, correlation IDs, and response shaping handled in interceptors rather than in controllers or middleware — show an interceptor implementation using the Observable pipeline (tap for logging, map for transformation), how to generate and propagate a correlation ID, and how to register it globally</summary>

**Why interceptors over middleware or controllers:**

- **Middleware** has no access to the execution context — it doesn't know which controller or handler will run. It also has no standard "after handler" hook (you'd need to monkey-patch `res.json`).
- **Controller-level** logic means repeating the same logging/wrapping in every handler — violates DRY and clutters business logic.
- **Interceptors** wrap the handler execution as an RxJS Observable, giving you clean before/after hooks with full execution context. They know which controller, handler, and decorators are involved.

**1. Logging interceptor with timing (using `tap`):**

```typescript
import {
  Injectable, NestInterceptor, ExecutionContext, CallHandler, Logger,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  private readonly logger = new Logger('HTTP');

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const req = context.switchToHttp().getRequest();
    const { method, url } = req;
    const start = Date.now();

    return next.handle().pipe(
      tap(() => {
        const res = context.switchToHttp().getResponse();
        this.logger.log(`${method} ${url} ${res.statusCode} ${Date.now() - start}ms`);
      }),
    );
  }
}
```

**2. Correlation ID interceptor (generate + propagate):**

```typescript
import { randomUUID } from 'crypto';
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class CorrelationIdInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const req = context.switchToHttp().getRequest();
    const res = context.switchToHttp().getResponse();

    // Use incoming header or generate new one
    const correlationId = req.headers['x-correlation-id'] || randomUUID();
    req.correlationId = correlationId;
    res.setHeader('x-correlation-id', correlationId);

    return next.handle();
  }
}
```

**3. Response transformation interceptor (using `map`):**

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface StandardResponse<T> {
  data: T;
  timestamp: string;
}

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, StandardResponse<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<StandardResponse<T>> {
    return next.handle().pipe(
      map((data) => ({
        data,
        timestamp: new Date().toISOString(),
      })),
    );
  }
}
```

**4. Register globally:**

```typescript
// Option A: in main.ts (no DI access)
app.useGlobalInterceptors(
  new CorrelationIdInterceptor(),
  new LoggingInterceptor(),
  new TransformInterceptor(),
);

// Option B: via module providers (has DI access — preferred)
@Module({
  providers: [
    { provide: APP_INTERCEPTOR, useClass: CorrelationIdInterceptor },
    { provide: APP_INTERCEPTOR, useClass: LoggingInterceptor },
    { provide: APP_INTERCEPTOR, useClass: TransformInterceptor },
  ],
})
export class AppModule {}
```

Option B is preferred because interceptors registered via `APP_INTERCEPTOR` participate in DI — they can inject `ConfigService`, loggers, or any other provider. Interceptors registered in `main.ts` are instantiated manually and can't use DI.

</details>

## Practical — Modules & Configuration

<details>
<summary>13. Create a dynamic module using forRoot/forRootAsync pattern — show a DatabaseModule that accepts configuration asynchronously (e.g., reading from ConfigService), demonstrate the module factory, and explain when you need forRootAsync vs forRoot and what the difference is</summary>

This was covered conceptually in question 6. Here's the full implementation.

**The DatabaseModule with both patterns:**

```typescript
import { Module, DynamicModule, Global } from '@nestjs/common';
import { Pool } from 'pg';

export interface DatabaseConfig {
  host: string;
  port: number;
  database: string;
  user: string;
  password: string;
  max?: number; // connection pool size
}

export interface DatabaseAsyncOptions {
  useFactory: (...args: any[]) => DatabaseConfig | Promise<DatabaseConfig>;
  inject?: any[];
  imports?: any[];
}

export const DATABASE_POOL = 'DATABASE_POOL';

@Global() // available everywhere without explicit imports
@Module({})
export class DatabaseModule {
  // Synchronous — config values known at compile time
  static forRoot(config: DatabaseConfig): DynamicModule {
    return {
      module: DatabaseModule,
      providers: [
        {
          provide: DATABASE_POOL,
          useValue: new Pool(config),
        },
      ],
      exports: [DATABASE_POOL],
    };
  }

  // Async — config depends on other providers (ConfigService, secrets manager)
  static forRootAsync(options: DatabaseAsyncOptions): DynamicModule {
    return {
      module: DatabaseModule,
      imports: options.imports || [],
      providers: [
        {
          provide: DATABASE_POOL,
          useFactory: async (...args: any[]) => {
            const config = await options.useFactory(...args);
            return new Pool(config);
          },
          inject: options.inject || [],
        },
      ],
      exports: [DATABASE_POOL],
    };
  }
}
```

**Usage in AppModule:**

```typescript
// forRoot — hardcoded config (dev/testing)
@Module({
  imports: [
    DatabaseModule.forRoot({
      host: 'localhost',
      port: 5432,
      database: 'myapp',
      user: 'postgres',
      password: 'password',
    }),
  ],
})
export class AppModule {}

// forRootAsync — config from ConfigService (production)
@Module({
  imports: [
    ConfigModule.forRoot(),
    DatabaseModule.forRootAsync({
      useFactory: (config: ConfigService) => ({
        host: config.get<string>('DB_HOST'),
        port: config.get<number>('DB_PORT'),
        database: config.get<string>('DB_NAME'),
        user: config.get<string>('DB_USER'),
        password: config.get<string>('DB_PASSWORD'),
        max: config.get<number>('DB_POOL_SIZE', 10),
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AppModule {}
```

**When to use which:**

- **`forRoot`** — Configuration values are static and known at module registration time. Common in tests, local development, or when config is hardcoded.
- **`forRootAsync`** — Configuration depends on other providers that need to be resolved first. This is the production pattern — you almost always need `ConfigService` (which reads from env vars) or an async secrets manager to provide database credentials.

The key difference is timing: `forRoot` runs synchronously during module registration, `forRootAsync` defers provider creation until the DI container can resolve the injected dependencies.

</details>

<details>
<summary>14. Set up @nestjs/config with validation and DI integration — show the ConfigModule setup with a validation schema (using Joi or class-validator), demonstrate injecting ConfigService into a provider, and explain why validating configuration at startup prevents runtime errors in production</summary>

**1. ConfigModule setup with Joi validation:**

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import * as Joi from 'joi';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true, // available everywhere without importing ConfigModule
      envFilePath: ['.env.local', '.env'], // first match wins
      validationSchema: Joi.object({
        NODE_ENV: Joi.string().valid('development', 'production', 'test').default('development'),
        PORT: Joi.number().default(3000),
        DB_HOST: Joi.string().required(),
        DB_PORT: Joi.number().default(5432),
        DB_NAME: Joi.string().required(),
        DB_USER: Joi.string().required(),
        DB_PASSWORD: Joi.string().required(),
        JWT_SECRET: Joi.string().min(32).required(),
        REDIS_URL: Joi.string().uri().required(),
      }),
      validationOptions: {
        abortEarly: false, // report ALL missing vars, not just the first
      },
    }),
  ],
})
export class AppModule {}
```

If `DB_HOST` or `JWT_SECRET` is missing, the app fails at startup with a clear error:

```
Error: Config validation error: "DB_HOST" is required. "JWT_SECRET" is required.
```

**2. Injecting ConfigService into a provider:**

```typescript
@Injectable()
export class AuthService {
  constructor(private config: ConfigService) {}

  async generateToken(user: User): Promise<string> {
    const secret = this.config.get<string>('JWT_SECRET');
    const expiresIn = this.config.get<string>('JWT_EXPIRES_IN', '1h'); // default fallback
    return this.jwtService.sign({ sub: user.id }, { secret, expiresIn });
  }
}
```

**3. Typed configuration with namespaces (alternative to Joi):**

```typescript
// config/database.config.ts
import { registerAs } from '@nestjs/config';

export default registerAs('database', () => ({
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT, 10) || 5432,
  name: process.env.DB_NAME,
}));

// Usage — type-safe access via namespace
const dbHost = this.config.get<string>('database.host');
```

**Why validate at startup:**

Without validation, a missing environment variable like `DB_HOST` doesn't surface until the first database query — which might be minutes or hours after deployment, in production, under load. At that point you're debugging a runtime crash in a live service.

Startup validation fails the deployment immediately with a descriptive error. In a CI/CD pipeline, this means the container exits with a non-zero code, the health check fails, and Kubernetes doesn't route traffic to it. The broken config never serves a single request. This is the difference between "the deploy failed cleanly" and "the service crashed at 3 AM when someone hit an obscure endpoint."

</details>

<details>
<summary>15. Implement custom providers for different scenarios — show useFactory for creating a provider that depends on configuration, useClass for swapping implementations (e.g., different storage backends), and useExisting for aliasing. Demonstrate how this enables testing and environment-specific behavior</summary>

The four provider types were explained in question 4. Here's a cohesive, real-world implementation showing them working together.

**Scenario: A notification service that sends via email in production and logs to console in development.**

**1. Define the interface (token):**

```typescript
export interface NotificationSender {
  send(to: string, subject: string, body: string): Promise<void>;
}

export const NOTIFICATION_SENDER = 'NOTIFICATION_SENDER';
```

**2. `useClass` — swap implementations by environment:**

```typescript
@Injectable()
export class EmailNotificationSender implements NotificationSender {
  constructor(@Inject('SMTP_CLIENT') private smtp: SmtpClient) {}

  async send(to: string, subject: string, body: string): Promise<void> {
    await this.smtp.sendMail({ to, subject, html: body });
  }
}

@Injectable()
export class ConsoleNotificationSender implements NotificationSender {
  async send(to: string, subject: string, body: string): Promise<void> {
    console.log(`[NOTIFICATION] To: ${to}, Subject: ${subject}`);
  }
}

// In the module — swap based on environment
{
  provide: NOTIFICATION_SENDER,
  useClass: process.env.NODE_ENV === 'production'
    ? EmailNotificationSender
    : ConsoleNotificationSender,
}
```

**3. `useFactory` — create provider from async config:**

```typescript
{
  provide: 'SMTP_CLIENT',
  useFactory: async (config: ConfigService) => {
    const client = new SmtpClient({
      host: config.get('SMTP_HOST'),
      port: config.get<number>('SMTP_PORT'),
      auth: {
        user: config.get('SMTP_USER'),
        pass: config.get('SMTP_PASSWORD'),
      },
    });
    await client.verify(); // fail fast if SMTP is unreachable
    return client;
  },
  inject: [ConfigService],
}
```

**4. `useExisting` — alias for backward compatibility:**

```typescript
// Old code injects 'MAILER', new code uses NOTIFICATION_SENDER
// Both resolve to the same instance
{
  provide: 'MAILER',
  useExisting: NOTIFICATION_SENDER,
}
```

**5. Testing — `useValue` to inject mocks:**

```typescript
describe('OrdersService', () => {
  let service: OrdersService;
  let mockNotifier: jest.Mocked<NotificationSender>;

  beforeEach(async () => {
    mockNotifier = {
      send: jest.fn().mockResolvedValue(undefined),
    };

    const module = await Test.createTestingModule({
      providers: [
        OrdersService,
        { provide: NOTIFICATION_SENDER, useValue: mockNotifier },
      ],
    }).compile();

    service = module.get(OrdersService);
  });

  it('sends order confirmation', async () => {
    await service.placeOrder(orderDto);
    expect(mockNotifier.send).toHaveBeenCalledWith(
      'user@example.com',
      'Order Confirmed',
      expect.stringContaining('Order #'),
    );
  });
});
```

**The key insight:** Consumer code always injects `NOTIFICATION_SENDER` and calls `.send()`. It never knows or cares whether it's talking to an SMTP client, a console logger, or a Jest mock. The DI container handles the wiring; the business logic stays clean.

</details>

## Practical — Data Access & Lifecycle

<details>
<summary>16. Integrate Prisma with NestJS — show the PrismaService extending PrismaClient with onModuleInit and onModuleDestroy, inject it into a domain service, and implement a repository-like abstraction on top of it. Explain why wrapping Prisma in a service matters for testability and lifecycle management.</summary>

**1. PrismaService — wraps PrismaClient with lifecycle hooks:**

```typescript
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

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

**2. PrismaModule — shared across the app:**

```typescript
@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

**3. Repository abstraction on top of Prisma:**

```typescript
@Injectable()
export class UsersRepository {
  constructor(private prisma: PrismaService) {}

  async findById(id: string): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { id } });
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { email } });
  }

  async create(data: { email: string; name: string; passwordHash: string }): Promise<User> {
    return this.prisma.user.create({ data });
  }

  async updateLastLogin(id: string): Promise<void> {
    await this.prisma.user.update({
      where: { id },
      data: { lastLoginAt: new Date() },
    });
  }
}
```

**4. Domain service uses the repository, not Prisma directly:**

```typescript
@Injectable()
export class UsersService {
  constructor(
    private usersRepo: UsersRepository,
    private authService: AuthService,
  ) {}

  async register(dto: CreateUserDto): Promise<User> {
    const existing = await this.usersRepo.findByEmail(dto.email);
    if (existing) throw new ConflictException('Email already registered');

    const passwordHash = await this.authService.hashPassword(dto.password);
    return this.usersRepo.create({ email: dto.email, name: dto.name, passwordHash });
  }
}
```

**Why wrapping Prisma matters:**

**Lifecycle management:** Without the `OnModuleInit`/`OnModuleDestroy` hooks, Prisma connections aren't tied to the NestJS application lifecycle. Connections wouldn't be established at startup or cleaned up on shutdown — you'd leak database connections during hot reloads in development and lose graceful shutdown in production (as covered in question 8).

**Testability:** If services use `PrismaService` directly, you must mock the entire Prisma client in tests — including all its models and methods. With a repository layer, you mock a small, focused interface:

```typescript
const module = await Test.createTestingModule({
  providers: [
    UsersService,
    {
      provide: UsersRepository,
      useValue: {
        findByEmail: jest.fn().mockResolvedValue(null),
        create: jest.fn().mockResolvedValue(mockUser),
      },
    },
    { provide: AuthService, useValue: { hashPassword: jest.fn() } },
  ],
}).compile();
```

**When to skip the repository layer:** For simple CRUD services where the service is essentially a pass-through to Prisma, the repository adds indirection without value (as discussed in question 7). Add it when business logic and query complexity justify the abstraction.

</details>

<details>
<summary>17. Implement graceful shutdown for a NestJS application running in Kubernetes — show enableShutdownHooks, the onModuleDestroy hooks that close database connections and drain active requests, the Kubernetes preStop hook configuration, and explain what happens to in-flight requests without proper shutdown</summary>

The lifecycle hooks and K8s integration were covered conceptually in question 8. Here's the full implementation.

**1. Enable shutdown hooks and track in-flight requests:**

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableShutdownHooks(); // listen for SIGTERM/SIGINT
  await app.listen(3000);
}
```

**2. Request-tracking interceptor — knows when all requests are drained:**

```typescript
@Injectable()
export class ShutdownService {
  private activeRequests = 0;
  private isShuttingDown = false;

  requestStarted() { this.activeRequests++; }
  requestFinished() { this.activeRequests--; }
  startShutdown() { this.isShuttingDown = true; }

  async waitForDrain(timeoutMs = 10_000): Promise<void> {
    const start = Date.now();
    while (this.activeRequests > 0 && Date.now() - start < timeoutMs) {
      await new Promise((resolve) => setTimeout(resolve, 100));
    }
  }
}

@Injectable()
export class RequestTrackingInterceptor implements NestInterceptor {
  constructor(private shutdownService: ShutdownService) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    this.shutdownService.requestStarted();
    return next.handle().pipe(
      finalize(() => this.shutdownService.requestFinished()),
    );
  }
}
```

**3. Lifecycle hooks that clean up resources:**

```typescript
@Injectable()
export class AppLifecycle implements BeforeApplicationShutdown, OnModuleDestroy {
  private readonly logger = new Logger('Shutdown');

  constructor(
    private shutdownService: ShutdownService,
    private prisma: PrismaService,
  ) {}

  async beforeApplicationShutdown(signal?: string) {
    this.logger.log(`Shutdown signal received: ${signal}`);
    this.shutdownService.startShutdown();
    // Wait for in-flight requests to complete (up to 10s)
    await this.shutdownService.waitForDrain(10_000);
    this.logger.log('All requests drained');
  }

  async onModuleDestroy() {
    // Close connections after requests are drained
    await this.prisma.$disconnect();
    this.logger.log('Database connections closed');
  }
}
```

**4. Kubernetes deployment configuration:**

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: api
          ports:
            - containerPort: 3000
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            periodSeconds: 5
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 5"]
```

**The shutdown timeline:**

1. K8s sends `preStop` hook -- pod sleeps 5s while K8s updates Service endpoints (removes pod from load balancer)
2. K8s sends `SIGTERM` -- NestJS `enableShutdownHooks` catches it
3. `beforeApplicationShutdown` fires -- stops accepting new connections, drains in-flight requests
4. `onModuleDestroy` fires -- closes database pools, Redis connections, message consumers
5. Process exits cleanly
6. If still running after `terminationGracePeriodSeconds` (30s), K8s sends `SIGKILL`

**Without proper shutdown:** Database connections are orphaned (the database keeps them open until idle timeout — potentially minutes), in-flight transactions are aborted mid-operation (risking data inconsistency), message queue consumers lose unacknowledged messages, and the next pod startup may hit connection pool limits because the old connections haven't timed out yet.

</details>

## Practical — Testing & Debugging

<details>
<summary>18. Write unit tests for a NestJS service using Test.createTestingModule — show how to create the testing module, mock dependencies with overrideProvider, test the service methods, and explain when to mock vs when to use real implementations</summary>

**Full unit test example for an OrdersService:**

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { NotFoundException, ConflictException } from '@nestjs/common';
import { OrdersService } from './orders.service';
import { OrdersRepository } from './orders.repository';
import { PaymentsService } from '../payments/payments.service';

describe('OrdersService', () => {
  let service: OrdersService;
  let ordersRepo: jest.Mocked<OrdersRepository>;
  let paymentsService: jest.Mocked<PaymentsService>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [OrdersService, OrdersRepository, PaymentsService],
    })
      .overrideProvider(OrdersRepository)
      .useValue({
        findById: jest.fn(),
        create: jest.fn(),
        updateStatus: jest.fn(),
      })
      .overrideProvider(PaymentsService)
      .useValue({
        charge: jest.fn(),
        refund: jest.fn(),
      })
      .compile();

    service = module.get(OrdersService);
    ordersRepo = module.get(OrdersRepository);
    paymentsService = module.get(PaymentsService);
  });

  describe('findById', () => {
    it('returns the order when found', async () => {
      const mockOrder = { id: '1', status: 'pending', total: 100 };
      ordersRepo.findById.mockResolvedValue(mockOrder as any);

      const result = await service.findById('1');
      expect(result).toEqual(mockOrder);
      expect(ordersRepo.findById).toHaveBeenCalledWith('1');
    });

    it('throws NotFoundException when order does not exist', async () => {
      ordersRepo.findById.mockResolvedValue(null);

      await expect(service.findById('nonexistent')).rejects.toThrow(NotFoundException);
    });
  });

  describe('placeOrder', () => {
    it('creates order and charges payment', async () => {
      const dto = { items: [{ productId: 'p1', qty: 2, price: 50 }], userId: 'u1' };
      const createdOrder = { id: 'ord-1', ...dto, total: 100, status: 'confirmed' };

      ordersRepo.create.mockResolvedValue(createdOrder as any);
      paymentsService.charge.mockResolvedValue({ transactionId: 'tx-1' } as any);

      const result = await service.placeOrder(dto as any);

      expect(ordersRepo.create).toHaveBeenCalled();
      expect(paymentsService.charge).toHaveBeenCalledWith('u1', 100);
      expect(result.status).toBe('confirmed');
    });

    it('rolls back order if payment fails', async () => {
      ordersRepo.create.mockResolvedValue({ id: 'ord-1' } as any);
      paymentsService.charge.mockRejectedValue(new Error('Card declined'));

      await expect(service.placeOrder({} as any)).rejects.toThrow('Card declined');
      expect(ordersRepo.updateStatus).toHaveBeenCalledWith('ord-1', 'failed');
    });
  });
});
```

**When to mock vs use real implementations:**

| Mock | Use real |
|------|----------|
| External services (payment APIs, email) | Pure domain logic classes (no I/O) |
| Database repositories (unit tests) | Utility/helper functions |
| HTTP clients | In-memory implementations (e.g., in-memory cache) |
| Third-party SDKs | The service under test itself |

**Rules of thumb:**

- **Unit tests**: Mock everything except the class under test. The goal is to test business logic in isolation — if a test fails, you know exactly which class has the bug.
- **Integration tests**: Use real implementations for the system you're testing (real database, real NestJS pipeline). Mock only external systems you don't control (third-party APIs).
- **Don't mock what you own** at the integration level — if your repository has a bug, you want integration tests to catch it.
- **Always mock what you don't own** — calling a real payment API in tests is slow, flaky, and expensive.

</details>

<details>
<summary>19. Write integration tests for a NestJS controller using supertest — show the test setup that creates the full NestJS application (with real database or test database), make HTTP requests to endpoints, assert on responses, and demonstrate testing the full request lifecycle (guards, pipes, interceptors)</summary>

**Full integration test setup:**

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';
import { PrismaService } from '../src/prisma/prisma.service';
import { JwtService } from '@nestjs/jwt';

describe('UsersController (e2e)', () => {
  let app: INestApplication;
  let prisma: PrismaService;
  let jwtService: JwtService;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule], // import the FULL module — real guards, pipes, interceptors
    }).compile();

    app = moduleFixture.createNestApplication();

    // Apply the same global config as main.ts — critical for realistic tests
    app.useGlobalPipes(
      new ValidationPipe({
        whitelist: true,
        forbidNonWhitelisted: true,
        transform: true,
      }),
    );

    await app.init();

    prisma = app.get(PrismaService);
    jwtService = app.get(JwtService);
  });

  afterAll(async () => {
    await prisma.$executeRaw`TRUNCATE TABLE "User" CASCADE`;
    await app.close();
  });

  // Helper to generate auth token
  function authToken(userId: string, roles: string[] = ['user']): string {
    return jwtService.sign({ sub: userId, roles });
  }

  describe('POST /users', () => {
    it('creates a user with valid input', async () => {
      const response = await request(app.getHttpServer())
        .post('/users')
        .send({ email: 'test@example.com', name: 'Test', password: 'securepass123' })
        .expect(201);

      expect(response.body.data).toMatchObject({
        email: 'test@example.com',
        name: 'Test',
      });
      expect(response.body.data.password).toBeUndefined(); // not exposed

      // Verify it actually hit the database
      const user = await prisma.user.findUnique({ where: { email: 'test@example.com' } });
      expect(user).not.toBeNull();
    });

    it('rejects invalid input (ValidationPipe)', async () => {
      const response = await request(app.getHttpServer())
        .post('/users')
        .send({ email: 'not-an-email' })
        .expect(400);

      expect(response.body.message).toEqual(
        expect.arrayContaining([
          expect.stringContaining('email must be an email'),
          expect.stringContaining('name'),
        ]),
      );
    });

    it('strips non-whitelisted properties', async () => {
      const response = await request(app.getHttpServer())
        .post('/users')
        .send({ email: 'a@b.com', name: 'Test', password: 'pass12345678', isAdmin: true })
        .expect(400); // forbidNonWhitelisted throws

      expect(response.body.message).toEqual(
        expect.arrayContaining([expect.stringContaining('isAdmin')]),
      );
    });
  });

  describe('GET /users/profile (auth required)', () => {
    it('returns 401 without token (JwtAuthGuard)', async () => {
      await request(app.getHttpServer())
        .get('/users/profile')
        .expect(401);
    });

    it('returns profile with valid token', async () => {
      const token = authToken('user-1');
      const response = await request(app.getHttpServer())
        .get('/users/profile')
        .set('Authorization', `Bearer ${token}`)
        .expect(200);

      expect(response.body.data.userId).toBe('user-1');
    });
  });

  describe('DELETE /admin/users/:id (role-protected)', () => {
    it('returns 403 for non-admin users (RolesGuard)', async () => {
      const token = authToken('user-1', ['user']);
      await request(app.getHttpServer())
        .delete('/admin/users/user-2')
        .set('Authorization', `Bearer ${token}`)
        .expect(403);
    });

    it('allows admin access', async () => {
      const token = authToken('admin-1', ['admin']);
      await request(app.getHttpServer())
        .delete('/admin/users/user-2')
        .set('Authorization', `Bearer ${token}`)
        .expect(200);
    });
  });
});
```

**Key differences from unit tests:**

- **Import `AppModule`** instead of individual providers — this boots the real application with all guards, pipes, interceptors, and middleware.
- **Apply global config** in the test setup (`useGlobalPipes`, etc.) exactly as `main.ts` does. A common bug is forgetting this — tests pass but the real app validates differently.
- **Use a test database** — either a dedicated test DB (via `.env.test`), Docker container spun up by the test suite, or in-memory SQLite for speed (with Prisma adapter limitations).
- **Test the HTTP contract**, not internal methods. Assert on status codes, response shape, headers. This catches issues in the full pipeline that unit tests miss — a pipe that strips fields, a guard that rejects, an interceptor that wraps the response.

</details><details>
<summary>20. Build a custom exception filter that handles domain-specific errors — show an exception filter that catches custom application errors (NotFoundError, ValidationError, AuthorizationError), maps them to appropriate HTTP responses with consistent error format, and explain how exception filters differ from try/catch in controllers</summary>

**1. Define domain errors (framework-agnostic):**

```typescript
// domain/errors.ts — no NestJS imports
export class DomainError extends Error {
  constructor(message: string, public readonly code: string) {
    super(message);
    this.name = this.constructor.name;
  }
}

export class NotFoundError extends DomainError {
  constructor(entity: string, id: string) {
    super(`${entity} with id '${id}' not found`, 'NOT_FOUND');
  }
}

export class BusinessValidationError extends DomainError {
  constructor(message: string, public readonly violations: string[]) {
    super(message, 'VALIDATION_ERROR');
  }
}

export class AuthorizationError extends DomainError {
  constructor(message = 'Insufficient permissions') {
    super(message, 'FORBIDDEN');
  }
}
```

**2. Custom exception filter — maps domain errors to HTTP responses:**

```typescript
import { ExceptionFilter, Catch, ArgumentsHost, HttpStatus, Logger } from '@nestjs/common';
import { Response, Request } from 'express';
import { DomainError, NotFoundError, BusinessValidationError, AuthorizationError } from '../domain/errors';

@Catch(DomainError)
export class DomainExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger('DomainExceptionFilter');

  private readonly statusMap = new Map<string, HttpStatus>([
    ['NOT_FOUND', HttpStatus.NOT_FOUND],
    ['VALIDATION_ERROR', HttpStatus.UNPROCESSABLE_ENTITY],
    ['FORBIDDEN', HttpStatus.FORBIDDEN],
  ]);

  catch(exception: DomainError, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    const status = this.statusMap.get(exception.code) || HttpStatus.INTERNAL_SERVER_ERROR;

    const body: Record<string, any> = {
      statusCode: status,
      error: exception.code,
      message: exception.message,
      timestamp: new Date().toISOString(),
      path: request.url,
    };

    // Include validation details when available
    if (exception instanceof BusinessValidationError) {
      body.violations = exception.violations;
    }

    this.logger.warn(`${exception.code}: ${exception.message} [${request.method} ${request.url}]`);
    response.status(status).json(body);
  }
}
```

**3. Register globally:**

```typescript
@Module({
  providers: [
    { provide: APP_FILTER, useClass: DomainExceptionFilter },
  ],
})
export class AppModule {}
```

**4. Usage in services — throw domain errors, not HTTP exceptions:**

```typescript
@Injectable()
export class OrdersService {
  async cancel(orderId: string, userId: string): Promise<void> {
    const order = await this.ordersRepo.findById(orderId);
    if (!order) throw new NotFoundError('Order', orderId);
    if (order.userId !== userId) throw new AuthorizationError('Cannot cancel another user\'s order');
    if (order.status === 'shipped') {
      throw new BusinessValidationError('Cannot cancel shipped order', [
        'Order has already been shipped',
        'Use the returns process instead',
      ]);
    }
    await this.ordersRepo.updateStatus(orderId, 'cancelled');
  }
}
```

**How exception filters differ from try/catch in controllers:**

| Aspect | try/catch in controller | Exception filter |
|--------|------------------------|------------------|
| Scope | Single handler | Global, controller, or method scope |
| DRY | Repeated in every handler | Defined once, applies everywhere |
| Separation of concerns | HTTP error formatting mixed with route logic | Error mapping is a separate layer |
| Coverage | Only catches errors in that handler | Catches errors from guards, pipes, interceptors, and handlers |
| Domain purity | Services must throw `HttpException` (framework coupling) | Services throw domain errors (framework-agnostic) |

The key benefit: your domain layer throws meaningful errors (`NotFoundError`, `AuthorizationError`) without knowing they'll become HTTP responses. The exception filter at the framework edge handles the translation. If you later add a gRPC transport, you write a new filter that maps the same domain errors to gRPC status codes — the domain layer doesn't change.

</details>

<details>
<summary>21. Set up a NestJS application with proper module boundaries for a domain-driven structure — show how to organize modules around business domains (users, orders, payments) rather than technical layers, demonstrate how modules communicate through well-defined exports, and what the module dependency graph should look like</summary>

**Domain-driven module structure:**

```
src/
  app.module.ts              # Root — imports all domain modules + infra
  users/
    users.module.ts
    users.controller.ts
    users.service.ts
    users.repository.ts
    dto/
    entities/
  orders/
    orders.module.ts
    orders.controller.ts
    orders.service.ts
    orders.repository.ts
    dto/
    entities/
  payments/
    payments.module.ts
    payments.service.ts       # No controller — internal module, not HTTP-facing
    payments.repository.ts
  notifications/
    notifications.module.ts
    notifications.service.ts
  shared/
    shared.module.ts          # Cross-cutting: logging, config, common utilities
    prisma/
      prisma.module.ts
      prisma.service.ts
```

Each module is a vertical slice: controller + service + repository + DTOs for one business domain. This contrasts with the technical-layer approach (`controllers/`, `services/`, `repositories/`) where files from unrelated domains sit together.

**Modules communicate through exports — the public API:**

```typescript
@Module({
  imports: [PrismaModule],
  controllers: [UsersController],
  providers: [UsersService, UsersRepository],
  exports: [UsersService], // only the service — repository is an implementation detail
})
export class UsersModule {}

@Module({
  imports: [PrismaModule],
  providers: [PaymentsService, PaymentsRepository, StripeClient],
  exports: [PaymentsService], // other modules can charge/refund, not access Stripe directly
})
export class PaymentsModule {}

@Module({
  imports: [
    UsersModule,      // needs user lookups for order validation
    PaymentsModule,   // needs payment processing
    PrismaModule,
  ],
  controllers: [OrdersController],
  providers: [OrdersService, OrdersRepository],
  exports: [OrdersService],
})
export class OrdersModule {}
```

**The module dependency graph:**

```
AppModule
  ├── SharedModule (global: config, logging)
  ├── PrismaModule (global: database access)
  ├── UsersModule
  ├── PaymentsModule
  ├── OrdersModule → depends on UsersModule, PaymentsModule
  └── NotificationsModule → depends on UsersModule (for user email lookups)
```

**Key principles:**

- **Dependencies flow one direction.** `OrdersModule` depends on `PaymentsModule`, not the reverse. If payments need to notify orders (e.g., payment webhook), use an event emitter or message queue rather than a circular import.
- **Export services, not repositories.** The repository is a private implementation detail. If `OrdersModule` needs user data, it calls `UsersService.findById()` — not `UsersRepository` directly. This means `UsersModule` can change its data access layer without breaking consumers.
- **Global modules sparingly.** Only truly cross-cutting infrastructure (`PrismaModule`, `ConfigModule`, `LoggerModule`) should be `@Global()`. Domain modules should always be explicitly imported so the dependency graph stays visible.
- **No module should import more than 3-4 other domain modules.** If a module has many imports, it's either doing too much (split it) or there's a missing shared abstraction.

**When to introduce events instead of direct imports:**

If `OrdersModule` needs `PaymentsModule` AND `PaymentsModule` needs `OrdersModule` (e.g., payment webhooks update order status), direct imports create a circular dependency. The fix isn't `forwardRef()` — it's introducing an event bus:

```typescript
// In PaymentsService — emit event instead of importing OrdersModule
this.eventEmitter.emit('payment.completed', { orderId, transactionId });

// In OrdersModule — listen without importing PaymentsModule
@OnEvent('payment.completed')
async handlePaymentCompleted(payload: PaymentCompletedEvent) {
  await this.ordersService.markPaid(payload.orderId, payload.transactionId);
}
```

This keeps the dependency graph acyclic and modules loosely coupled.

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>22. Tell me about a time you chose NestJS for a project (or inherited a NestJS codebase) — what were the benefits and drawbacks compared to plain Express, and how did the framework's opinions help or hinder the team?</summary>

**What the interviewer is looking for:** Evidence that you've made (or evaluated) a real framework decision with tradeoffs, not just "NestJS is better." They want to hear about team context, the problems the framework solved, and the costs you paid for it.

**Key points to hit:**

- The specific project context and team size that made NestJS attractive (or problematic)
- Concrete benefits you experienced (not abstract bullet points from docs)
- Honest drawbacks or friction points
- Whether you'd make the same choice again and why

**Suggested structure (STAR-like):**

1. **Context**: What was the project? Team size? What were you building? What was the codebase like before (greenfield vs migration)?
2. **Decision/inheritance**: Did you choose NestJS or inherit it? What alternatives were considered? What drove the decision?
3. **Benefits realized**: Be specific — "onboarding was faster because new developers already knew where to find things" is better than "it gave us structure."
4. **Drawbacks encountered**: Where did the framework fight you? Boilerplate for simple things? DI scope issues? Decorator magic making debugging harder?
5. **Outcome**: Would you make the same choice? What would you do differently?

**Example outline to personalize:**

> "Our team of 6 was building a microservice for [domain]. We inherited an Express codebase that had grown organically — three different patterns for error handling, manual dependency wiring, no consistent validation. We migrated to NestJS incrementally, module by module.

> **Benefits**: Onboarding dropped from ~2 weeks to ~3 days for the NestJS portions. The guard/pipe/interceptor pipeline eliminated our inconsistent auth and validation middleware. DI made unit testing straightforward where it had been painful before.

> **Drawbacks**: Simple endpoints that were 10 lines in Express became 3 files in NestJS (controller, service, DTO). We hit the REQUEST-scope bubble-up problem (covered in question 3) when we added multi-tenant context — it turned half our providers into request-scoped ones before we switched to AsyncLocalStorage. Debugging decorator-based issues was harder because the stack trace pointed to NestJS internals rather than our code.

> **Verdict**: Right choice for our team size and project complexity. Wouldn't use it for a 2-endpoint Lambda function."

</details>

<details>
<summary>23. Describe a time you structured a NestJS application to scale across a team — how did you organize modules, what conventions did you establish, and what architectural decisions did you make about domain boundaries?</summary>

**What the interviewer is looking for:** Your ability to think about code organization as a team scaling problem, not just a technical one. They want to hear about conventions that reduced coordination overhead and architectural decisions that let multiple developers work in parallel without stepping on each other.

**Key points to hit:**

- How you drew module boundaries (by domain, not by technical layer)
- Specific conventions you established and enforced (naming, file structure, export rules)
- How the structure enabled parallel development
- Tradeoffs you navigated (too many small modules vs too few large ones)

**Suggested structure:**

1. **Context**: Team size, growth trajectory, the application's domain complexity
2. **Module organization**: How you divided the codebase — what domains became modules, how granular you went
3. **Conventions**: Naming patterns, file structure per module, rules about exports, where shared code lives
4. **Domain boundaries**: How you decided what belongs in which module — the hard cases where something could go in multiple places
5. **Scaling outcomes**: Did it work? Where did the structure break down as the team or codebase grew?

**Example outline to personalize:**

> "We were a team of 8 working on an e-commerce backend with NestJS. When I joined, the app had 3 modules that had grown into god-modules with 15+ providers each. PRs constantly conflicted.

> **Module reorganization**: We split into domain modules (users, products, orders, inventory, payments, notifications) with a rule: each module maps to one bounded context. If a concept appeared in two modules, we asked 'which module owns this data?' — the owner gets the service, the other module imports it.

> **Conventions we established**: Every module gets `[domain].module.ts`, `[domain].controller.ts`, `[domain].service.ts`, `[domain].repository.ts`. DTOs in a `dto/` subfolder. Only services go in `exports` — never repositories. Shared utilities go in `SharedModule` but we review PRs carefully to prevent it from becoming a dumping ground.

> **The hard boundary decisions**: Should 'order pricing' live in OrdersModule or a separate PricingModule? We kept it in OrdersModule initially, with a rule that if it grew beyond 3 files or another module needed it, we'd extract. This 'extract when needed' approach prevented premature abstraction.

> **Result**: Teams could work on separate modules with minimal conflicts. The dependency graph stayed manageable — we drew it on a whiteboard and reviewed it quarterly."

</details>

<details>
<summary>24. Tell me about a time you debugged a complex NestJS issue (DI problems, lifecycle issues, middleware ordering) — what was the symptom, how did you diagnose it, and what did you learn about NestJS internals?</summary>

**What the interviewer is looking for:** Systematic debugging ability under framework abstraction. They want to see that you can go beyond "the error message said X" to understanding the underlying mechanism. NestJS hides a lot behind decorators and DI magic — demonstrating you can peel that back is senior-level.

**Key points to hit:**

- A clear description of the symptom and why it was confusing
- Your systematic approach to diagnosis (not "I Googled it")
- What you learned about how NestJS works internally
- How you prevented the issue from recurring

**Suggested structure:**

1. **Symptom**: What went wrong? What made it hard to diagnose initially?
2. **Investigation**: What did you try? How did you narrow it down?
3. **Root cause**: What was actually happening? Why did NestJS behave that way?
4. **Fix**: What did you change?
5. **Takeaway**: What did you learn about NestJS internals that you didn't know before?

**Common NestJS debugging scenarios to draw from:**

- **Scope bubble-up**: A REQUEST-scoped provider deep in the tree silently turned dozens of providers into request-scoped, causing massive performance degradation. Diagnosed by logging provider instantiation counts.
- **Circular dependency**: Cryptic error at startup. The error message shows the module chain but not which specific provider is circular. Fixed by tracing the injection graph manually or using `forwardRef()` as a temporary fix while restructuring.
- **Guard/interceptor ordering**: A global guard registered via `APP_GUARD` ran in a different order than expected because multiple modules each registered their own `APP_GUARD`. Learned that registration order depends on module import order in `AppModule`.
- **Lifecycle hook not firing**: `onModuleDestroy` never ran because `enableShutdownHooks()` wasn't called. The service leaked database connections for weeks before someone noticed connection pool exhaustion in production.
- **Middleware not applying**: Middleware configured in `configure(consumer)` didn't apply to certain routes because the route path pattern didn't match (wildcards, parameterized routes).

**Example outline to personalize:**

> "We had intermittent 500 errors in production. The logs showed 'Cannot read property of undefined' inside a service that worked perfectly in most requests. After adding detailed logging, I discovered the service was receiving a stale instance of a request-scoped provider.

> Root cause: A singleton service was injecting a REQUEST-scoped service, but NestJS had forced the singleton to become request-scoped (scope bubble-up). However, a cached reference in a closure was holding onto a previous request's instance. I learned that NestJS's scope propagation changes the instantiation behavior of the entire dependency chain — not just the directly injected provider.

> Fix: Replaced the REQUEST-scoped provider with AsyncLocalStorage for request context propagation, which avoids scope bubbling entirely."

</details>

<details>
<summary>25. Describe a time you set up testing for a NestJS application — what testing strategy did you choose, how did you handle mocking vs real dependencies, and what was the most challenging aspect?</summary>

**What the interviewer is looking for:** Practical testing philosophy grounded in real experience. They want to hear that you understand the tradeoffs between test types, have opinions about mocking boundaries, and have dealt with the real challenges of testing a DI-heavy framework.

**Key points to hit:**

- Your testing strategy (the mix of unit/integration/e2e and why)
- Concrete decisions about what to mock vs what to use real
- How NestJS's DI system helped or complicated testing
- The hardest part and how you solved it

**Suggested structure:**

1. **Context**: What was the project? What was the testing situation when you started?
2. **Strategy**: What testing pyramid did you implement? What did each layer cover?
3. **Mocking decisions**: Where did you draw the line between mocks and real implementations? Why?
4. **NestJS-specific challenges**: How did you handle `Test.createTestingModule`, provider overrides, global pipes/guards in tests?
5. **Hardest part**: What was the most challenging testing problem and how did you solve it?

**Example outline to personalize:**

> "I set up testing for a NestJS service that had zero tests. The codebase had 15 modules, Prisma for data access, Redis for caching, and external API integrations for payments and notifications.

> **Strategy**: Unit tests for services (mocked dependencies, fast, isolated), integration tests for the critical flows (real database via Docker testcontainers, real NestJS pipeline), no e2e tests initially — we added those later for the 3 most critical user journeys.

> **Mocking boundary**: Unit tests mock everything except the class under test (as shown in question 18). Integration tests use a real PostgreSQL container and real NestJS guards/pipes/interceptors but mock external services (payment API, email). The rule: mock what you don't own, use real implementations for what you do.

> **NestJS-specific challenge**: The biggest pain was global pipes and guards not being applied in test modules unless you explicitly configure them. Our integration tests were passing with invalid input because `ValidationPipe` wasn't registered in the test setup. We fixed this by creating a shared `createTestApp()` helper that mirrors `main.ts` configuration exactly — `useGlobalPipes`, `useGlobalFilters`, global interceptors (as shown in question 19).

> **Hardest part**: Testing services that depended on REQUEST-scoped providers. The `Test.createTestingModule` doesn't simulate a request context by default, so REQUEST-scoped providers weren't instantiated. We had to use `moduleRef.resolve()` instead of `moduleRef.get()` and pass a context identifier. This taught us to minimize REQUEST scope usage — it complicates both runtime performance and testing."

</details>
