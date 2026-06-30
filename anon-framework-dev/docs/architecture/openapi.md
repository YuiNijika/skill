# OpenAPI 生成

接口多了以后，最麻烦的不是写接口，而是让前端、测试和调试工具知道“现在到底有哪些接口”。

Anon 可以直接从已经注册的路由里生成一份 OpenAPI JSON。它不会替你脑补业务含义，但能把路径、方法、路径参数和你在路由上补充的说明整理出来。

---

## 给路由补一点说明

如果只是普通路由，也能生成文档；但文档会比较干。建议在对外接口上顺手补上这些信息：

```php
use Anon\Core\Facade\Route;
use Anon\Controller\UserController;

Route::get('/users/{id}', [UserController::class, 'show'])
    ->name('users.show')
    ->summary('获取用户详情')
    ->description('根据用户 ID 返回用户基础资料')
    ->tags(['Users'])
    ->schema([
        'name' => 'string|required',
        'email' => 'string|required',
        'age' => 'integer',
    ])
    ->openapi([
        'responses' => [
            '200' => ['description' => 'User detail'],
            '404' => ['description' => 'User not found'],
        ],
    ]);
```

这几个方法的作用很直接：

| 方法 | 用来做什么 |
|---|---|
| `name()` | 给路由取名，也会作为默认 `operationId` |
| `summary()` | 一句话说明这个接口干什么 |
| `description()` | 需要更详细时再写，不必每个接口都长篇说明 |
| `tags()` | 给接口分组，比如 `Users`、`Orders` |
| `schema()` | 用简写方式声明 JSON 请求体字段 |
| `openapi()` | 直接合并一段 OpenAPI operation 配置 |

---

## 生成文档

在项目根目录执行：

```bash
php anon openapi:generate
```

默认会生成到：

```text
runtime/openapi.json
```

如果你想放到别的位置：

```bash
php anon openapi:generate --output=runtime/docs/openapi.json
```

---

## 当前会生成哪些内容

目前生成的是一份“够用、可继续扩展”的 OpenAPI 文档，包含：

- OpenAPI 版本
- 应用名称和框架版本
- 路径与 HTTP 方法
- `{id}` 这类路径参数
- `operationId`
- `summary`、`description`、`tags`
- Server Actions 的调用入口
- 基础 `responses`
- 通过 `openapi()` 追加的自定义片段

也就是说，你可以先用它打通 Swagger、Apifox、Postman 这类工具链；更细的 request body、schema、security 规则，可以通过 `openapi()` 慢慢补。

---

## Server Actions 也会进去

如果你注册了 Server Action：

```php
use Anon\Action\PublishPost;
use Anon\Core\Facade\Action;

Action::post('posts.publish', PublishPost::class)
    ->summary('发布文章')
    ->description('把草稿文章切换为发布状态')
    ->tags(['Posts']);
```

生成出来会多一个路径：

```text
POST /_actions/posts.publish
```

默认会给它一个基础 JSON 请求体，并带上常见错误响应：`401`、`403`、`419`、`422`、`429`。如果你想把字段写细一点，可以继续用 `openapi()` 追加：

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
