# 依赖注入容器 (Container)

Anon Framework Next 提供了一个依赖注入容器 (IoC Container)，负责管理类依赖并执行自动依赖注入。

---

## 获取容器实例

`Anon\Core\Foundation\App` 类继承自 `Container`。整个应用程序 `App` 就是一个容器单例。

```php
use Anon\Core\Foundation\App;

$app = App::getInstance();
```

---

## 基本用法

### 1. 自动依赖解析 (Make)

```php
namespace Anon\Controller;

use Anon\Service\UserService;

class UserController
{
    protected UserService $userService;

    // 容器会自动注入 UserService 实例
    public function __construct(UserService $userService)
    {
        $this->userService = $userService;
    }
}
```

当路由调度到该控制器时，底层会调用 `$app->make(UserController::class)`，容器会自动解析并注入依赖。

### 2. 手动绑定 (Bind)

可以给接口绑定具体实现，或自定义实例化逻辑：

```php
$app = App::getInstance();

// 绑定类名
$app->bind('cache', \Anon\Core\Cache\RedisCache::class);

// 绑定闭包（每次 make 执行闭包）
$app->bind('config', function ($app) {
    return new ConfigLoader('/path/to/config');
});

// 解析
$cache = $app->make('cache');
```

### 3. 绑定单例 (Instance)

将现有对象绑定为全局单例：

```php
$app = App::getInstance();

$myObject = new stdClass();
$myObject->name = "Anon";

$app->instance('my_shared_object', $myObject);

// 每次 make 返回相同的对象
$sameObject = $app->make('my_shared_object');
```

---

## 核心组件别名

框架启动时自动在容器中绑定了以下核心组件，可通过 `$app->make('别名')` 获取单例：

| 别名 | 绑定的类 | 作用 |
| :--- | :--- | :--- |
| `app` | `Anon\Core\Foundation\App` | 应用容器自身 |
| `router` | `Anon\Core\Routing\Router` | 路由解析器 |
| `db` | `Anon\Core\Database\Connection` | 数据库连接管理 |
| `log` | `Anon\Core\Log\Manager` | 日志记录器 |
| `env` | `Anon\Core\Support\Env` | 环境变量解析器 |
| `cache` | `Anon\Core\Cache\Manager` | 缓存管理器 |
| `session` | `Anon\Core\Session\Manager` | Session 管理器 |
| `validator` | `Anon\Core\Validation\Factory` | 验证器工厂 |
| `event` | `Anon\Core\Event\Dispatcher` | 事件调度器 |
| `request` | `Anon\Core\Http\Request` | HTTP 请求对象 |
| `auth` | `Anon\Core\Auth\Manager` | JWT 身份认证 |
| `storage` | `Anon\Core\Storage\Manager` | 文件存储管理器 |

---

## 参数反射缓存

容器底层实现了反射缓存机制 (Dependencies Cache)。首次反射解析类及其构造函数参数后，元数据会被缓存至内存。后续实例化请求直接读取缓存，减少 Reflection API 开销。
