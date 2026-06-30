# Server Actions

Server Actions 适合处理“不是标准资源 CRUD，但确实要改状态”的操作。

比如：提交表单、发布文章、审核用户、批量重试任务、触发一次导出。REST 仍然适合 `GET /articles`、`POST /articles` 这类资源接口；Action 更适合 `posts.publish`、`users.disable` 这种命令式动作。

## 一个最小 Action

先写一个 Action 类：

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

然后在路由文件里注册：

```php
use Anon\Action\PublishPost;
use Anon\Core\Facade\Action;

Action::post('posts.publish', PublishPost::class)
    ->middleware(['auth:api', 'throttle:30,1'])
    ->summary('发布文章')
    ->tags(['Posts']);
```

调用方式固定走统一入口：

```http
POST /_actions/posts.publish
Content-Type: application/json

{
  "id": 1
}
```

成功响应会继续使用框架的统一结构：

```json
{
  "success": true,
  "code": 200,
  "message": "OK",
  "data": {
    "published": true,
    "id": 1
  }
}
```

## 为什么不直接写控制器

控制器更适合资源接口，Action 更适合命令接口。

常见分工可以这样理解：

| 场景 | 推荐方式 |
| --- | --- |
| 获取文章列表 | REST 路由 |
| 创建文章 | REST 路由 |
| 发布文章 | Server Action |
| 批量禁用用户 | Server Action |
| 上传文件 | REST 路由或 Server Action，都可以，看业务是否更像资源 |

Server Actions 的价值不是“少写一个控制器”，而是把这类命令统一收口：注册、鉴权、限流、验证、响应、OpenAPI 都走同一套规则。

## Action 自身 DSL

除了在注册处写中间件和 OpenAPI 信息，Action 类自己也可以声明运行约束和响应信息。

```php
namespace Anon\Action;

use Anon\Core\Action\Action;
use Anon\Core\Facade\Auth;
use Anon\Core\Http\Request;

class PublishPost extends Action
{
    public function middleware(): array
    {
        return ['auth:api', 'throttle:30,1'];
    }

    public function authorize(Request $request): bool
    {
        return Auth::hasPermission('posts.publish');
    }

    public function message(mixed $result = null, ?Request $request = null): string
    {
        return '文章已发布';
    }

    public function statusCode(mixed $result = null, ?Request $request = null): int
    {
        return 200;
    }

    public function meta(mixed $result = null, ?Request $request = null): array
    {
        return [
            'action' => 'posts.publish',
        ];
    }

    public function links(mixed $result = null, ?Request $request = null): array
    {
        return [
            'self' => '/articles/' . ($result['id'] ?? ''),
        ];
    }

    public function handle(Request $request): array
    {
        return [
            'published' => true,
            'id' => $request->input('id'),
        ];
    }
}
```

执行时会按这个顺序合并：

1. `actions.middleware` 里的全局中间件
2. 注册处 `Action::post(...)->middleware(...)` 写的中间件
3. Action 类自己 `middleware()` 返回的中间件

`authorize()` 返回 `false` 时，会直接返回 `FORBIDDEN`。`code()`、`message()`、`statusCode()`、`meta()`、`links()` 会进入成功响应 envelope；如果 `handle()` 已经直接返回 `Response`，框架不会再二次包装。

## 返回方式

最常用的写法是让 `handle()` 直接返回数组或对象，框架会自动包成统一响应：

```php
public function code(mixed $result = null, ?Request $request = null): string
{
    return 'POST_PUBLISHED';
}

public function message(mixed $result = null, ?Request $request = null): string
{
    return '文章已发布';
}

public function handle(Request $request): array
{
    return [
        'published' => true,
        'id' => $request->input('id'),
    ];
}
```

响应会是：

```json
{
  "success": true,
  "code": "POST_PUBLISHED",
  "message": "文章已发布",
  "data": {
    "published": true,
    "id": 1
  }
}
```

如果你想在 Action 里直接结束响应，可以使用基类提供的快捷方法：

```php
public function handle(Request $request)
{
    $post = [
        'id' => 1,
        'title' => $request->input('title'),
    ];

    return $this->created($post, '文章已创建');
}
```

目前内置三个快捷方法：

| 方法 | 适合场景 |
| --- | --- |
| `success($data, $message, $statusCode, $code, $meta, $links)` | 需要完整控制成功响应 |
| `created($data, $message, $meta, $links)` | 创建成功，返回 `201` 和 `CREATED` |
| `noContent()` | 操作成功但没有响应体，返回 `204` |

直接返回 `Response` 时，`code()`、`message()`、`statusCode()`、`meta()`、`links()` 不会再参与包装。这个规则适合文件下载、重定向、`204 No Content` 这类不需要标准 JSON body 的场景。

## 使用 FormRequest

Action 可以声明自己使用哪个请求类。这样验证失败会直接返回 `VALIDATION_FAILED`，字段错误也会保持统一格式。

```php
namespace Anon\Action;

use Anon\Core\Action\Action;
use Anon\Request\PublishPostRequest;

class PublishPost extends Action
{
    public function request(): string
    {
        return PublishPostRequest::class;
    }

    public function handle(PublishPostRequest $request): array
    {
        return [
            'published' => true,
            'id' => $request->input('id'),
        ];
    }
}
```

`PublishPostRequest` 仍然按 FormRequest 的方式写：

```php
namespace Anon\Request;

use Anon\Core\Http\FormRequest;

class PublishPostRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'id' => 'required|int',
        ];
    }
}
```

## 中间件

每个 Action 都能单独挂中间件：

```php
Action::post('posts.publish', PublishPost::class)
    ->middleware(['auth:api', 'throttle:30,1']);
```

也可以在配置里给所有 Actions 加默认中间件：

```php
'actions' => [
    'middleware' => ['throttle:60,1'],
]
```

这里复用的就是路由中间件别名，所以 `auth:api`、`throttle:60,1` 这类写法不需要重新学一套。

## CSRF

框架默认把 Server Actions 当成会改状态的入口，所以 `actions.csrf` 默认是开启的。

如果是浏览器表单、后台页面、带 session 的站点，建议保持开启：

```php
'actions' => [
    'csrf' => true,
]
```

请求时传入：

```http
X-CSRF-TOKEN: your-token
```

或者在请求体里传 `_token`。

如果项目是纯 API，使用 Bearer Token/JWT 做认证，可以显式关闭：

```php
'actions' => [
    'csrf' => false,
]
```

骨架项目默认就是 API-first 体验，所以示例配置里关闭了 CSRF，避免第一次调用就被 session token 卡住。

## 修改入口路径

默认入口是：

```http
POST /_actions/{action}
```

可以通过配置调整：

```php
'actions' => [
    'path' => '/actions',
]
```

调整后调用地址会变成：

```http
POST /actions/posts.publish
```

## OpenAPI

注册 Action 时写的 `summary()`、`description()`、`tags()` 会进入 OpenAPI：

```php
Action::post('posts.publish', PublishPost::class)
    ->summary('发布文章')
    ->description('把草稿文章切换为发布状态')
    ->tags(['Posts']);
```

生成 OpenAPI 后，会看到类似路径：

```http
POST /_actions/posts.publish
```

当前会输出基础 JSON 请求体和常见错误响应：`401`、`403`、`419`、`422`、`429`。简单字段可以直接用 `schema()` 声明：

```php
Action::post('posts.publish', PublishPost::class)
    ->schema([
        'id' => 'integer|required',
        'notify' => 'boolean',
    ]);
```

如果你需要完整控制 OpenAPI operation，再用 `openapi()` 自己补：

```php
Action::post('posts.publish', PublishPost::class)
    ->openapi([
        'requestBody' => [
            'content' => [
                'application/json' => [
                    'schema' => [
                        'type' => 'object',
                        'required' => ['id'],
                        'properties' => [
                            'id' => ['type' => 'integer'],
                        ],
                    ],
                ],
            ],
        ],
        'responses' => [
            '200' => ['description' => 'OK'],
            '401' => ['description' => 'Unauthorized.'],
            '403' => ['description' => 'Forbidden.'],
            '419' => ['description' => 'CSRF token mismatch.'],
            '422' => ['description' => 'Validation failed.'],
            '429' => ['description' => 'Too Many Requests.'],
        ],
    ]);
```

## 路由缓存

Server Actions 会跟路由一起进入缓存。执行 `route:cache` 后，显式注册过的 Action 不会丢。

需要注意的是：Action 注册仍然要写在应用会加载的路由文件里，比如骨架项目的 `app/Route/Main.php`。

## 当前边界

第一版故意保持克制：

- 只支持显式注册
- 只支持 `POST`
- 不允许客户端传类名或方法名
- 不做前端 import 服务端函数
- 不做 JS SDK 生成
- 不做 Attribute 扫描

后续如果要做得更像 Next.js，可以在这个基础上继续加：`#[ServerAction]` 扫描、`action:list`、`action:cache`、JS 调用器和类型声明生成。
