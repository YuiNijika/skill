# 服务提供者

当项目开始变大，很多初始化逻辑会慢慢散落到路由、钩子、配置文件里：绑定服务、注册中间件别名、挂事件监听、启动模块……这些东西继续堆在入口文件里会很快失控。

服务提供者就是用来收拢这类启动逻辑的地方。简单说：

- `register()`：先把东西注册进容器
- `boot()`：等基础服务准备好后，再做依赖路由、事件、中间件的启动工作

---

## 写一个服务提供者

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

一般建议这样分工：

- 容器绑定、单例、基础服务：放 `register()`
- 中间件别名、事件监听、启动钩子：放 `boot()`

---

## 让框架加载它

在 `anon.config.php` 里加到 `app.providers`：

```php
return Config::define([
    'app' => [
        'providers' => [
            Anon\Provider\AppServiceProvider::class,
        ],
    ],
]);
```

你可以放多个 provider。项目模块多的时候，可以按领域拆开，比如：

```php
'providers' => [
    Anon\Provider\AppServiceProvider::class,
    Anon\Provider\AuthServiceProvider::class,
    Anon\Provider\AdminServiceProvider::class,
]
```

如果你的项目骨架使用的是 `app/Provider`，那命名空间也应同步保持为 `Anon\Provider` 这类与目录大小写一致的形式，不要混用旧的 `App\Providers` 示例。

---

## 启动顺序

大致顺序是：

1. 读取 `.env`
2. 读取 `anon.config.php`
3. 定义框架常量
4. 执行所有 provider 的 `register()`
5. 执行所有 provider 的 `boot()`
6. 加载路由
7. 处理请求

所以，如果你想在路由加载前注册中间件别名，放在 `boot()` 里是合适的。
