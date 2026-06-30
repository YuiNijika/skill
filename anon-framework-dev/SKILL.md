---
name: "anon-framework-dev"
description: "Builds and modifies Anon Framework Next apps and core code. Invoke when working on Anon controllers, routes, services, auth, response format, env/config, or framework compatibility fixes."
---

# Anon Framework Dev

Use this skill when the task is about `Anon Framework Next` application development, framework-core changes, route/controller/service work, auth flow, env/config handling, response shaping, or production compatibility fixes.

This skill is intentionally opinionated. It does not just repeat docs. It merges:

- current framework capabilities
- project-level production lessons
- Linux deployment constraints
- corrections for outdated or legacy examples still visible in some docs

## Goals

- Produce app-first changes that fit the current Anon project conventions.
- Keep Linux production compatibility in mind.
- Reuse the framework's existing Request / Response / Auth / Env / Config capabilities instead of inventing parallel logic.
- Preserve the current REST response contract.
- Use the copied Markdown docs inside this skill as the first reference.

## Docs Normalization

Some copied source docs still reflect older conventions. When code examples in docs conflict with current project rules, prefer the current rules below.

Known normalizations:

- docs may still show lowercase directories like `app/controller`; prefer `app/Controller`, `app/Service`, `app/Route`
- docs or examples may still show success payload `code: "OK"`; current rule is numeric HTTP status code
- docs may still show raw `getenv()` in config examples; business code should use `Env::get()`
- docs may show `App\...` namespaces in examples; adapt them to the actual project namespace and layout in use

Normalized source-doc updates already synced into this workspace:

- router-related docs now emphasize uppercase app directories and Linux case sensitivity
- response-related docs now align success payload examples to numeric `code`
- config and env-related docs now consistently push `Env::get()`-style environment access
- generator and provider-related docs now better reflect uppercase directories and current namespace conventions

## Working Scope

Prefer this order when implementing:

1. Change `app` code first.
2. Change framework core only when the bug is truly in framework behavior.
3. If the running project depends on vendored `anon-core`, mirror framework fixes into the runtime copy when necessary.

## Directory Rules

Anon uses PSR-4 and Linux is case-sensitive.

Always prefer these app directories:

- `app/Controller`
- `app/Service`
- `app/Route`
- `app/Action`
- `app/Model`
- `app/Middleware`
- `app/Provider`

Do not introduce lowercase directories like `app/controller` or `app/service`.

If you touch generators or framework path resolution, ensure they prefer uppercase directory names while remaining backward-compatible with legacy lowercase layouts.

## App Structure

Typical project layout:

```text
app/
  Controller/
  Service/
  Route/
  Action/
run/
anon.config.php
.env
```

Typical route registration:

```php
use Anon\Core\Facade\Route;
use Anon\Controller\ExampleController;

Route::group('/example', function () {
    Route::get('/{id}', [ExampleController::class, 'show']);
    Route::post('/', [ExampleController::class, 'store']);
});
```

Typical controller/service split:

```php
namespace Anon\Controller;

use Anon\Core\Http\Request;
use Anon\Core\Http\Response;
use Anon\Service\UserService;

class UserController
{
    public function __construct(private UserService $users)
    {
    }

    public function show(Request $request, string $id): Response
    {
        $user = $this->users->findById($id);

        if ($user === null) {
            return Response::error('用户不存在', 404, null, 'USER_NOT_FOUND');
        }

        return Response::success($user, 'OK', 200);
    }
}
```

## Core Conventions

### 1. Environment Access

Never use raw `getenv()` in app or framework business code.

Use:

```php
use Anon\Core\Facade\Env;

$value = trim((string) Env::get('SOME_KEY', ''));
```

Reason:

- production may disable `putenv()`
- framework already supports fallback to `$_ENV` and `$_SERVER`
- direct `getenv()` can work locally and fail online

### 2. Config Access

Use:

```php
use Anon\Core\Facade\Config;

$sslVerify = (bool) Config::get('http.ssl_verify', true);
```

Important:

- `JWT_SECRET` must come from `.env`, not config
- `http.ssl_verify` belongs in `anon.config.php`
- config changes may be hidden by `runtime/cache/config.php`

If config changes seem ineffective, remember:

- run `php anon config:clear`, or
- delete `runtime/cache/config.php`

### 3. Request Access

Use the actual methods implemented by `Anon\Core\Http\Request`.

Available common methods:

- `input()`
- `route()`
- `header()`
- `cookie()`
- `file()`
- `method()`
- `path()`

Do not call methods that do not exist, such as:

- `query()`

For query parameters, use:

```php
$type = $request->input('type', null);
```

For route parameters, use:

```php
$id = $request->route('id');
```

For headers and cookies:

```php
$token = $request->header('Authorization', '');
$cookie = $request->cookie('access_token');
```

Prefer constructor or method injection of `Request` instead of reading globals directly.

### 4. Response Contract

Keep the current REST-style response shape:

Successful response:

```json
{
  "success": true,
  "code": 200,
  "message": "OK",
  "data": {}
}
```

Error response:

```json
{
  "success": false,
  "code": 400,
  "message": "Bad Request",
  "error_code": "BAD_REQUEST"
}
```

Rules:

- `code` must be numeric HTTP status code
- business error identifiers go into `error_code`
- use `Response::success()` and `Response::error()`
- use `withHeader()` or `withHeaders()` for extra headers
- use `Response::json()` only when you explicitly want raw output without the standard envelope

Example:

```php
return Response::success($data, 'OK', 200);
```

```php
return Response::error('缺少参数', 400, null, 'BAD_REQUEST');
```

Headers and cookies:

```php
return Response::success($data, 'OK', 200)
    ->withHeaders([
        'X-Trace-Id' => 'abc123',
        'X-RateLimit-Limit' => 60,
    ]);
```

### 5. Auth / Cookie

If local auth is needed, prefer existing `Auth` facade abilities rather than manual token logic.

Useful capabilities already exist:

- `Auth::issueTokenPair()`
- `Auth::setTokenCookie()`
- `Auth::setTokenPairCookies()`
- `Auth::forgetTokenPairCookies()`
- `Auth::user()`
- `Auth::check()`
- `Auth::guard('admin')`
- `Auth::refreshTokens()`
- `Auth::revokeSession()`
- `Auth::revokeOtherSessions()`

Recommended auth pattern:

```php
use Anon\Core\Facade\Auth;
use Anon\Core\Http\Response;

$tokens = Auth::issueTokenPair([
    'id' => 1,
    'roles' => ['admin'],
    'permissions' => ['posts.publish'],
]);

$response = Response::success($tokens, 'Login Success', 200);

return Auth::setTokenPairCookies($response, $tokens);
```

Guard guidance:

- use default `api` guard unless there is a real need for separation
- give each browser-facing guard its own cookie names
- prefer `token_sources` = `['header', 'cookie']` for mixed SPA/API cases
- keep `JWT_SECRET` in `.env`

If a task is about third-party identity but no official upstream login API exists, do not claim you can directly read another site's cookies in a browser context. Call out cross-domain cookie limitations clearly.

## Router Patterns

Prefer controller routes over heavy closure routes.

Useful patterns:

```php
Route::get('/users/{id}', [UserController::class, 'show']);
Route::post('/users', [UserController::class, 'store']);
Route::resource('/admin/users', AdminUserController::class);
```

Dynamic route rules:

- use `{id}` style placeholders
- accept route params in controller method signatures
- use `Request::route()` only when needed

Middleware on routes:

```php
Route::get('/admin/users', [AdminUserController::class, 'index'])
    ->middleware(['auth:admin', 'throttle:60,60']);
```

When grouping routes:

```php
Route::group('/admin', function () {
    Route::get('/dashboard', [DashboardController::class, 'index']);
});
```

If route examples in docs show lowercase `app/route/main.php`, translate that to current uppercase `app/Route/Main.php` in real projects.

## FormRequest And Validation

Use `FormRequest` when:

- controller validation is repeated
- authorization belongs to the request itself
- field error output needs to stay uniform

Example:

```php
namespace Anon\Http\Requests;

use Anon\Core\Http\FormRequest;

class StoreUserRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'username' => 'required|min:4|max:20',
            'email' => 'required|email',
        ];
    }
}
```

Controller:

```php
public function store(StoreUserRequest $request): Response
{
    $data = $request->input();

    return Response::success($data, 'Created', 201, 'USER_CREATED');
}
```

Use `Validator::make()` when:

- validation is local and simple
- it is not worth creating a dedicated request class
- the validation belongs to a service or action instead of a controller

Common rules seen in docs:

- `required`
- `email`
- `min`
- `max`
- `numeric`
- `integer`
- `in`

Validation failure pattern:

```php
$validator = Validator::make($data, $rules, $messages);

if ($validator->fails()) {
    return Response::error(
        $validator->firstError(),
        400,
        $validator->errors(),
        'VALIDATION_FAILED'
    );
}
```

## API Resource And Pagination

Use Resources when the main task is response shaping, field renaming, hiding fields, or attaching `meta` / `links`.

Resource rules:

- keep controllers focused on fetching business data
- move formatting into `Anon\Core\Http\Resource\Json`
- use `collection()` for lists
- prefer `Paginator` for stable pagination metadata

Example:

```php
return UserResource::make($user)
    ->meta(['loaded_at' => time()])
    ->links(['self' => '/users/' . $id]);
```

Pagination pattern:

```php
use Anon\Core\Pagination\Paginator;

$paginator = Paginator::make(
    items: $users,
    total: $total,
    page: $page,
    perPage: $perPage,
    path: '/users'
);

return UserResource::collection($paginator);
```

Important:

- keep current response contract numeric even if older docs show `code: "OK"`
- `meta.pagination` and `links` are the preferred structure for paged lists

## Middleware

Use middleware for cross-cutting concerns:

- authentication
- throttling
- cors
- request signature checks
- global response headers

Built-in aliases commonly used:

- `auth`
- `cors`
- `throttle`

Custom alias pattern:

```php
Route::aliasMiddleware('admin', App\Middleware\AdminOnly::class);
```

Guidelines:

- route-specific business guards belong on the route
- CORS and universal request logging usually belong in global middleware or provider boot
- if you add response headers in middleware, return the modified `Response`

## Container And Providers

The app is a container. Prefer constructor injection over manual service location.

Common container aliases available:

- `app`
- `router`
- `db`
- `log`
- `env`
- `cache`
- `session`
- `validator`
- `event`
- `request`
- `auth`
- `storage`

Use providers when startup logic begins to spread:

- `register()` for bindings and singletons
- `boot()` for route aliases, middleware aliases, hooks, event listeners

Provider registration belongs in:

```php
'app' => [
    'providers' => [
        App\Providers\AppServiceProvider::class,
    ],
],
```

## Session, Storage, Queue

### Session

Use `Session` facade for session-based state:

- `Session::get()`
- `Session::set()`
- `Session::has()`
- `Session::delete()`
- `Session::clear()`
- `Session::destroy()`
- `Session::regenerateId()`

Use cases:

- classic web login
- csrf-related state
- temporary multi-step forms

Do not hand-roll session cookie management unless necessary.

### Storage

Use `Storage` facade for file operations:

- `put`
- `append`
- `get`
- `exists`
- `delete`
- `copy`
- `move`
- `url`

Prefer `Storage` over direct filesystem calls when the feature may later move to another disk.

### Queue

Use queue for slow or retryable jobs:

- mail
- exports
- third-party sync
- media processing

Common commands:

- `php anon make:job SendWelcomeEmail`
- `php anon queue:work`
- `php anon queue:failed`
- `php anon queue:retry`
- `php anon queue:clear-failed`

Production note:

- queue depends on Redis
- use Supervisor or systemd for workers

## Database / ORM Rules

When touching ORM or query builder logic, preserve these known constraints:

- validate `join` operators
- `insertAll()` must align fields by the first row's keys
- soft delete scope must cover `update`, `delete`, `aggregate`, `exists`, and `cursor`
- non-auto-increment / UUID primary keys must not be treated as failed saves

## HTTP Client Rules

Use the built-in HTTP client:

```php
use Anon\Core\Http\Client as HttpClient;

$client = new HttpClient();
$client->sslVerify((bool) Config::get('http.ssl_verify', true));
```

Notes:

- SSL verification should be read dynamically at request time
- if `anon.config.php` changes do not take effect, suspect config cache first
- response shape is an array with `status`, `headers`, `body`, `json`
- for third-party APIs, parse and normalize errors in `Service`, not in controller

## Route / Proxy Patterns

For proxy-style controllers:

- keep controller thin
- move upstream request logic into `Service`
- normalize all exits through small `respondSuccess()` / `respondError()` helpers
- preserve important upstream headers such as rate-limit headers when needed

Example pattern:

```php
public function files(Request $request, string $id): Response
{
    $type = $request->input('type', null);

    if ($id === '') {
        return Response::error('缺少模组标识(id)', 400, null, 'BAD_REQUEST');
    }

    $result = $this->service->getFiles($id, $type);

    if ($result === null) {
        return Response::error('API 请求失败', 500, null, 'API_ERROR');
    }

    return Response::success($result, 'OK', 200);
}
```

For proxy integrations:

- keep upstream auth, request assembly, and error mapping in `Service`
- keep controller focused on request params and unified response exit
- preserve upstream headers only when there is actual client value
- if upstream has no official login API, do not promise cookie reuse across browser origins

## Hooks And Events

Use `Hook` for framework lifecycle extension and `Event` for domain/application events.

Prefer `Hook` when:

- touching boot lifecycle
- injecting response headers globally
- extending request/response flow

Prefer `Event` when:

- decoupling business side effects
- reacting to user registration, billing, notifications, indexing

Useful built-in hooks:

- `app_init`
- `route_loaded`
- `request_begin`
- `response_send`
- `app_end`
- `exception_render`
- `auth_login`
- `auth_logout`

Recommended registration location:

- `app/hook.php` for hooks
- provider `boot()` for organized startup registration

## OpenAPI

If the task affects public APIs, enrich routes or actions with OpenAPI metadata.

Useful route/action methods:

- `name()`
- `summary()`
- `description()`
- `tags()`
- `schema()`
- `openapi()`

Generation command:

```bash
php anon openapi:generate
```

Default output:

```text
runtime/openapi.json
```

Use OpenAPI enrichment especially for:

- external APIs
- admin APIs used by frontend teams
- server actions consumed by typed clients

## Server Actions

Use Server Actions when the operation is command-like rather than resource-like.

Good Action cases:

- `posts.publish`
- `users.disable`
- `orders.retry-payment`
- `jobs.retry-failed`

Keep using REST routes for standard resource CRUD.

Action advantages:

- command-oriented naming
- unified validation and auth
- middleware support
- OpenAPI integration
- route cache compatibility

If Action validation is non-trivial, pair it with `FormRequest`.

## Documentation Map

All copied source docs now live inside this skill:

- `docs/architecture/*.md`

Read these first:

- `docs/architecture/current-conventions.md`
- `docs/architecture/deployment-checklist.md`

Use `docs/architecture/current-conventions.md` first when the task touches directory case, response format, env/config, JWT, route style, or app-vs-core decision making.
Use `docs/architecture/deployment-checklist.md` first when the task is about publishing, online-only failures, autoload, runtime cache, vendor/runtime drift, php-fpm, or opcache.

Then read the matching local subsystem doc when needed, for example:

- `docs/architecture/router.md`
- `docs/architecture/request-response.md`
- `docs/architecture/auth.md`
- `docs/architecture/form-request.md`
- `docs/architecture/validator.md`
- `docs/architecture/api-resource.md`
- `docs/architecture/server-actions.md`
- `docs/architecture/http-client.md`
- `docs/architecture/openapi.md`
- `docs/architecture/middleware.md`
- `docs/architecture/container.md`
- `docs/architecture/service-provider.md`
- `docs/architecture/session.md`
- `docs/architecture/storage.md`
- `docs/architecture/queue.md`
- `docs/architecture/hook.md`
- `docs/architecture/event.md`

## App First / Core Boundary

Default decision rule:

1. If the issue can be solved in `app`, solve it in `app`.
2. Touch `framework/core` only when the behavior is clearly a framework bug or missing framework capability.
3. If a project is running against vendored `anon-core`, runtime-critical framework fixes may need to be mirrored into the effective vendor copy.

Typical app-only changes:

- add or change business APIs
- third-party proxy integration
- controller/request/response wiring
- business validation
- custom middleware
- resource shaping

Typical framework-level changes:

- request/response base capability is missing
- env loading is incompatible with production
- route loader or generator breaks PSR-4 conventions
- auth manager / http client / orm behavior is incorrect across projects

When changing framework core:

- state the exact root cause
- explain why app-level workaround is not enough
- note whether vendor/runtime copy must also be updated

## Task Playbooks

### 1. Add A New API Endpoint

Follow this order:

1. add route in `app/Route/Main.php` or related route file
2. add or update controller method in `app/Controller`
3. move upstream/io/business orchestration into `app/Service` if logic is non-trivial
4. validate with `FormRequest` or `Validator`
5. return unified `Response::success()` / `Response::error()`
6. if external-facing, enrich with OpenAPI metadata

Checklist:

- route path and method are explicit
- route params use `{id}` style placeholders
- no heavy logic in closures
- response contract stays stable
- diagnostics checked after edits

### 2. Fix A 500 Error

Use this flow:

1. read the exact error and stack
2. confirm whether the missing symbol or method really exists in runtime code
3. compare local path case vs Linux namespace/path expectations
4. check whether docs/example code used a method that does not exist in current runtime
5. apply the smallest compatible fix
6. if production-only, include cache/autoload/opcache follow-up

Common Anon 500 patterns:

- wrong directory case under `app`
- stale `composer` autoload mapping
- stale config or route cache
- calling non-existent request helpers like `query()`
- vendor/runtime `anon-core` version mismatch

### 3. Fix Config Not Taking Effect

Use this order:

1. verify code reads from `Config::get()`
2. verify config key lives in `anon.config.php`
3. verify secrets still come from `.env` when appropriate
4. inspect `runtime/cache/config.php`
5. clear config cache
6. only then suspect code logic

Common examples:

- `http.ssl_verify`
- auth cookie options
- session settings
- storage default disk
- queue prefix

### 4. Fix Local Works / Online Fails

Use this checklist in order:

1. directory case and namespace case
2. `Env::get()` vs raw `getenv()`
3. config cache / route cache
4. composer autoload
5. php-fpm / opcache stale runtime
6. system environment variables overriding `.env`

This is the default production-diff checklist for Anon.

### 5. Add Third-Party API Integration

Prefer this architecture:

1. controller reads request params only
2. service assembles outbound URL, headers, auth, retries, error normalization
3. controller converts service result into unified response envelope
4. preserve useful upstream headers only when clients need them

Rules:

- use `Env::get()` / `Config::get()` for keys and options
- use built-in HTTP client
- keep upstream-specific error parsing inside service
- never leak secret values in responses

### 6. Add Browser Login / Cookie Auth

Use this order:

1. decide whether login is local auth or third-party identity
2. if local auth, prefer `Auth::issueTokenPair()` + `Auth::setTokenPairCookies()`
3. configure cookie names and token sources per guard
4. if browser-side only auth is needed, verify cookie flags carefully
5. if third-party site has no official login API, explicitly document cross-domain limits

Do not promise:

- direct browser-side access to another site's cookies
- stable upstream session reuse without official support

### 7. Add Queue Work

Use queue only when the task is slow, retryable, or non-blocking.

Flow:

1. create job
2. push job from controller/service
3. ensure worker command and queue name are documented
4. mention Redis dependency and worker supervision in handoff

### 8. Add OpenAPI Support

When an API is expected to be consumed by frontend/tools:

1. add `summary()`
2. add `description()` only when useful
3. add `tags()`
4. add `schema()` or `openapi()` when request/response details matter
5. regenerate OpenAPI output if needed

## Review Heuristics

When reviewing or modifying Anon code, check these first:

- does it violate app-first principle
- does it use `Env::get()` instead of `getenv()`
- does it preserve numeric `code`
- does it keep Linux-safe directory case
- does it add logic to controller that belongs in service/resource/middleware
- does it ignore cache or autoload as part of deployment follow-up

## Code Patterns

Use these as copyable starting points.

### 1. Standard Controller

```php
namespace Anon\Controller;

use Anon\Core\Http\Request;
use Anon\Core\Http\Response;
use Anon\Service\UserService;

class UserController
{
    public function __construct(private UserService $users)
    {
    }

    public function show(Request $request, string $id): Response
    {
        if ($id === '') {
            return Response::error('缺少用户 ID', 400, null, 'BAD_REQUEST');
        }

        $user = $this->users->findById($id);

        if ($user === null) {
            return Response::error('用户不存在', 404, null, 'USER_NOT_FOUND');
        }

        return Response::success($user, 'OK', 200);
    }
}
```

### 2. Proxy Controller

```php
namespace Anon\Controller;

use Anon\Core\Http\Request;
use Anon\Core\Http\Response;
use Anon\Service\ThirdPartyClient;

class ThirdPartyProxy
{
    public function __construct(private ThirdPartyClient $client)
    {
    }

    public function detail(Request $request, string $id): Response
    {
        if ($id === '') {
            return Response::error('缺少资源 ID', 400, null, 'BAD_REQUEST');
        }

        $result = $this->client->getDetail($id);

        if ($result === null) {
            return Response::error('上游请求失败', 502, null, 'UPSTREAM_ERROR');
        }

        return Response::success($result, 'OK', 200);
    }
}
```

### 3. Service For External API

```php
namespace Anon\Service;

use Anon\Core\Facade\Config;
use Anon\Core\Facade\Env;
use Anon\Core\Http\Client as HttpClient;

class ThirdPartyClient
{
    private string $baseUrl;
    private string $apiKey;

    public function __construct()
    {
        $this->baseUrl = rtrim(trim((string) Env::get('THIRD_PARTY_BASE_URL', 'https://example.com')), '/');
        $this->apiKey = trim((string) Env::get('THIRD_PARTY_API_KEY', ''));
    }

    public function getDetail(string $id): ?array
    {
        $client = new HttpClient();
        $client->sslVerify((bool) Config::get('http.ssl_verify', true));

        $response = $client->get($this->baseUrl . '/v1/items/' . $id, [], [
            'Authorization' => 'Bearer ' . $this->apiKey,
        ]);

        if (($response['status'] ?? 500) !== 200) {
            return null;
        }

        return is_array($response['json']['data'] ?? null) ? $response['json']['data'] : null;
    }
}
```

### 4. FormRequest

```php
namespace Anon\Http\Requests;

use Anon\Core\Http\FormRequest;

class StoreArticleRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'title' => 'required|max:100',
            'content' => 'required|max:5000',
        ];
    }

    public function messages(): array
    {
        return [
            'title.required' => '标题不能为空',
            'content.required' => '内容不能为空',
        ];
    }
}
```

### 5. Controller Using FormRequest

```php
public function store(StoreArticleRequest $request): Response
{
    $data = $request->input();

    return Response::success($data, 'Created', 201, 'ARTICLE_CREATED');
}
```

### 6. Validator Inline

```php
$data = $request->input();

$validator = Validator::make($data, [
    'name' => 'required|max:50',
    'email' => 'required|email',
]);

if ($validator->fails()) {
    return Response::error(
        $validator->firstError(),
        400,
        $validator->errors(),
        'VALIDATION_FAILED'
    );
}
```

### 7. Auth Login With Cookie

```php
use Anon\Core\Facade\Auth;
use Anon\Core\Http\Response;

public function login(Request $request): Response
{
    $user = [
        'id' => 1,
        'roles' => ['admin'],
        'permissions' => ['posts.publish'],
    ];

    $tokens = Auth::issueTokenPair($user);
    $response = Response::success($tokens, 'Login Success', 200);

    return Auth::setTokenPairCookies($response, $tokens);
}
```

### 8. Auth Logout

```php
use Anon\Core\Facade\Auth;
use Anon\Core\Http\Response;

public function logout(): Response
{
    if (!Auth::logout()) {
        return Response::error('Unauthorized', 401, null, 'UNAUTHORIZED');
    }

    return Auth::forgetTokenPairCookies(
        Response::success(null, 'Logout Success', 200)
    );
}
```

### 9. Resource Response

```php
return UserResource::make($user)
    ->meta(['loaded_at' => time()])
    ->links(['self' => '/users/' . $id]);
```

### 10. Route + OpenAPI

```php
Route::get('/users/{id}', [UserController::class, 'show'])
    ->name('users.show')
    ->summary('获取用户详情')
    ->description('根据用户 ID 返回用户资料')
    ->tags(['Users'])
    ->openapi([
        'responses' => [
            '200' => ['description' => 'OK'],
            '404' => ['description' => 'User not found'],
        ],
    ]);
```

### 11. Action Pattern

```php
namespace Anon\Action;

use Anon\Core\Action\Action;
use Anon\Core\Http\Request;

class PublishPost extends Action
{
    public function handle(Request $request): array
    {
        return [
            'published' => true,
            'id' => $request->input('id'),
        ];
    }
}
```

### 12. Provider Pattern

```php
namespace Anon\Provider;

use Anon\Core\Foundation\ServiceProvider;
use Anon\Core\Facade\Route;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app()->bind('example.service', ExampleService::class);
    }

    public function boot(): void
    {
        Route::aliasMiddleware('admin', \Anon\Middleware\AdminOnly::class);
    }
}
```

### 13. Hook Pattern

```php
use Anon\Core\Facade\Hook;
use Anon\Core\Http\Response;

Hook::add('response_send', function (Response $response) {
    $response->setHeader('X-Powered-By', 'Anon Framework Next');
});
```

## Recipes

Use these when the user asks for a complete, common feature and speed matters more than abstraction design.

### 1. File Upload API

Minimal flow:

1. read uploaded file from `Request`
2. validate file presence and type
3. store via `Storage`
4. return file path or URL

Example:

```php
use Anon\Core\Http\Request;
use Anon\Core\Http\Response;
use Anon\Core\Facade\Storage;

public function upload(Request $request): Response
{
    $file = $request->file('file');

    if ($file === null) {
        return Response::error('缺少上传文件', 400, null, 'BAD_REQUEST');
    }

    $path = 'uploads/' . date('Ymd') . '/' . uniqid('', true) . '.bin';
    Storage::put($path, file_get_contents($file['tmp_name']));

    return Response::success([
        'path' => $path,
        'url' => Storage::url($path),
    ], 'Uploaded', 201);
}
```

### 2. Login API

Minimal flow:

1. validate input
2. verify user identity
3. issue token pair
4. write auth cookies

Example:

```php
use Anon\Core\Facade\Auth;
use Anon\Core\Http\Request;
use Anon\Core\Http\Response;

public function login(Request $request): Response
{
    $username = trim((string) $request->input('username', ''));
    $password = (string) $request->input('password', '');

    if ($username === '' || $password === '') {
        return Response::error('账号或密码不能为空', 400, null, 'BAD_REQUEST');
    }

    $user = ['id' => 1, 'name' => $username];
    $tokens = Auth::issueTokenPair($user);

    return Auth::setTokenPairCookies(
        Response::success($tokens, 'Login Success', 200),
        $tokens
    );
}
```

### 3. Paginated List API

Minimal flow:

1. read `page` and `per_page`
2. query records and total
3. build `Paginator`
4. return `Resource::collection($paginator)`

Example:

```php
use Anon\Core\Pagination\Paginator;

$page = max(1, (int) $request->input('page', 1));
$perPage = max(1, min(100, (int) $request->input('per_page', 15)));

$items = $service->list($page, $perPage);
$total = $service->count();

$paginator = Paginator::make(
    items: $items,
    total: $total,
    page: $page,
    perPage: $perPage,
    path: '/articles'
);

return ArticleResource::collection($paginator);
```

### 4. OpenAPI-Ready API

Minimal flow:

1. define controller route
2. add `summary`, `description`, `tags`
3. append `openapi()` details when public-facing

Example:

```php
Route::post('/articles', [ArticleController::class, 'store'])
    ->name('articles.store')
    ->summary('创建文章')
    ->description('创建一篇新的文章')
    ->tags(['Articles'])
    ->openapi([
        'responses' => [
            '201' => ['description' => 'Created'],
            '422' => ['description' => 'Validation failed'],
        ],
    ]);
```

### 5. Queue Dispatch API

Minimal flow:

1. accept request
2. push job
3. return accepted/success response immediately

Example:

```php
use Anon\Core\Facade\Queue;
use Anon\Core\Http\Request;
use Anon\Core\Http\Response;
use Anon\Job\SendReportMail;

public function sendReport(Request $request): Response
{
    $email = trim((string) $request->input('email', ''));

    if ($email === '') {
        return Response::error('缺少邮箱', 400, null, 'BAD_REQUEST');
    }

    Queue::push(new SendReportMail(['email' => $email]), 'emails', 0, 3);

    return Response::success(['queued' => true], 'Accepted', 202);
}
```

### 6. Third-Party Proxy API

Minimal flow:

1. controller reads route/query params
2. service makes upstream request
3. map upstream failure to stable local error
4. preserve useful headers only when needed

Example:

```php
public function files(Request $request, string $id): Response
{
    $type = $request->input('type', null);
    $result = $this->client->getFiles($id, $type);

    if ($result === null) {
        return Response::error('API 请求失败', 500, null, 'API_ERROR');
    }

    return Response::success($result, 'OK', 200);
}
```

### 7. Framework-Bug Triage

When the user says "这个要不要改 framework":

1. first attempt app-level fix
2. if multiple apps would hit the same issue, escalate to core
3. if current app runs against vendored `anon-core`, identify the real runtime copy
4. only then patch `framework/core` and mirror if necessary

## Change Strategy

When asked to implement something in an Anon app:

1. Inspect `app/Route`, target controller, and related service.
2. Reuse existing framework helpers first.
3. Keep response output unified.
4. Avoid introducing new abstractions unless repetition is real.
5. After edits, check diagnostics on touched files.
6. If production behavior differs from local behavior, check:
   - directory case
   - config cache
   - env loading path
   - `putenv()` compatibility
   - composer autoload refresh

## Production Checklist

When handing off Anon changes, remind about these if relevant:

- `composer dump-autoload`
- clear route cache
- clear config cache
- restart php-fpm / reload opcache
- verify Linux directory case matches namespace case
- if auth/cookie changed, verify domain, secure, httponly, samesite
- if queue changed, confirm worker is running
- if OpenAPI changed, regenerate `runtime/openapi.json`

## Anti-Patterns

Avoid these mistakes:

- using `getenv()` directly
- placing `JWT_SECRET` into config
- returning string `"OK"` in `code`
- creating lowercase PSR-4 directories
- calling unsupported request helpers like `query()`
- changing framework core when the issue is app-only
- ignoring runtime cache when config seems wrong
- putting heavy formatting logic directly into controllers instead of Resource
- using closure routes for large business flows
- bypassing Auth facade for cookie/token flows when framework already supports it

## Good Output Style

When implementing Anon tasks:

- be direct
- keep changes minimal
- explain the real root cause, not just the symptom
- mention exact deployment follow-up when cache, autoload, or Linux case sensitivity matters
- mention when docs examples are legacy and what the normalized current pattern should be

## Example Triggers

Invoke this skill when the user asks things like:

- "修复 Anon 控制器报错"
- "给 Anon 框架加一个接口"
- "改 anon.config.php 配置不生效"
- "线上 Linux 正常 / 本地异常"
- "Anon Response 返回格式不对"
- "给 app/Controller 增加业务逻辑"
- "修复 Env / Auth / Route / Service 相关问题"
