# 中间件

中间件可以理解成请求进入控制器前后的一层“关卡”。常见场景有：鉴权、限流、跨域、日志、签名校验。

请求会一层层进入中间件，再到路由动作；响应会按相反方向一层层返回。这就是常说的洋葱模型。

---

## 写一个中间件

中间件只需要提供一个 `handle` 方法。它会拿到当前请求 `$request`，以及继续往后执行的 `$next`。

### 拦截请求

下面这个例子会检查请求里有没有 `token`。没有就直接返回错误；有就继续往后走。

```php
namespace Anon\Middleware;

use Anon\Core\Http\Request;
use Anon\Core\Http\Response;

class AuthMiddleware
{
    public function handle(Request $request, \Closure $next): Response
    {
        $token = $request->input('token');

        if (empty($token)) {
            return Response::error('Missing token', 401, null, 'UNAUTHORIZED');
        }

        return $next($request);
    }
}
```

### 修改响应

你也可以先让请求继续执行，拿到响应后再补充一些东西，比如响应头。

```php
namespace Anon\Middleware;

use Anon\Core\Http\Request;
use Anon\Core\Http\Response;

class HeaderMiddleware
{
    public function handle(Request $request, \Closure $next): Response
    {
        $response = $next($request);

        return $response->setHeader('X-Powered-By', 'Anon');
    }
}
```

---

## 绑定到路由

最常见的用法是直接挂在路由上。

```php
use Anon\Middleware\AuthMiddleware;

Route::get('/user/profile', function () {
    return 'Profile Page';
})->middleware(AuthMiddleware::class);
```

也可以一次挂多个：

```php
Route::get('/admin/reports', [ReportController::class, 'index'])
    ->middleware([
        AuthMiddleware::class,
        AdminOnly::class,
    ]);
```

---

## 绑定到路由组

如果一组接口都需要同样的中间件，就放到路由组上。

```php
use Anon\Middleware\AuthMiddleware;

Route::group(['prefix' => '/admin', 'middleware' => AuthMiddleware::class], function ($route) {
    $route->get('/dashboard', function () {
        return 'Admin Dashboard';
    });
});
```

组可以嵌套，中间件会从外到内依次执行。

---

## 全局中间件

全局中间件会跑在每个请求上，适合放 CORS、请求日志这类基础能力。

```php
use Anon\Core\Facade\Route;
use Anon\Core\Http\Middleware\Cors;

Route::globalMiddleware(Cors::class);
```

如果项目已经使用服务提供者，建议把这类注册放到 provider 的 `boot()` 里，入口文件会更干净。

---

## 内置中间件

框架内置了几个 API 项目里常用的中间件。

### CORS

前后端分离项目基本都会用到 CORS。内置 `Cors` 会自动补充跨域响应头；如果请求是 `OPTIONS` 预检，并且没有显式注册对应路由，路由器会直接返回 `204`。

```php
Route::globalMiddleware('cors');
```

如果默认策略不够用，可以继承 `Anon\Core\Http\Middleware\Cors` 后覆盖相关配置。

### Auth

`auth` 用来保护需要登录的接口。

```php
Route::get('/me', [UserController::class, 'me'])
    ->middleware('auth');
```

如果你配置了多个 guard，可以把 guard 名放在参数里：

```php
Route::get('/admin/me', [AdminController::class, 'me'])
    ->middleware('auth:admin');
```

### Throttle

`throttle` 用来做简单限流，防止接口被刷。

```php
// 每 60 秒最多请求 60 次
Route::get('/api/data', function () {
    return 'Sensitive Data';
})->middleware('throttle:60,60');
```

超限后会抛出 `429 Too Many Requests`，最终由异常处理器输出统一错误结构。

---

## 别名和参数

内置别名：

| 别名 | 对应中间件 |
|---|---|
| `auth` | `Anon\Core\Http\Middleware\Authenticate` |
| `cors` | `Anon\Core\Http\Middleware\Cors` |
| `throttle` | `Anon\Core\Http\Middleware\Throttle` |

你可以继续写完整类名，也可以写别名。别名更适合路由文件，短一些，也更像 DSL。

```php
Route::globalMiddleware('cors');

Route::get('/me', [UserController::class, 'me'])
    ->middleware('auth:api');

Route::get('/reports', [ReportController::class, 'index'])
    ->middleware(['auth', 'throttle:30,60']);
```

参数写在冒号后面，多个参数用英文逗号分隔。全局中间件和路由中间件都支持同一套写法。

---

## 自定义别名

如果你的中间件经常被使用，可以给它起个短名字。

```php
Route::aliasMiddleware('admin', Anon\Middleware\AdminOnly::class);

Route::aliasMiddlewares([
    'signed' => Anon\Middleware\ValidateSignature::class,
]);
```

如果你的骨架目录已经统一成 `app/Middleware`，命名空间也应同步用 `Anon\Middleware` 这类与 PSR-4 目录一致的写法。Windows 本地可能不会报错，但 Linux 线上会严格区分路径大小写。

之后路由里就可以这样写：

```php
Route::get('/admin/users', [AdminUserController::class, 'index'])
    ->middleware(['auth', 'admin']);
```
