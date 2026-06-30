# Facade

Facade 为应用服务容器 (Container) 中的类提供了一个“静态”代理接口。它让你能够在不进行繁琐的依赖注入（无需在构造函数声明）的情况下，直接以静态方法的形式调用底层动态对象的方法。

Anon Framework Next 中的Facade系统设计极其优雅，代码可读性极高。

---

## 工作原理

所有的Facade类都继承自基础类 `Anon\Core\Facade\Facade`。
Facade类内部并不实现任何实际的业务逻辑，它们仅仅是通过 `getFacadeAccessor()` 方法返回一个**容器绑定的标识（字符串）**。

当你调用Facade的静态方法时，例如 `Log::info('test')`，基础 Facade 会使用魔术方法 `__callStatic`：
1. 从容器中 `make('log')` 解析出真实的日志对象。
2. 将 `info('test')` 的调用转发给该真实对象。

---

## 内置Facade列表

框架为你预置了以下常用的Facade，你可以随时在控制器或业务代码中 `use` 它们：

| Facade类 | 对应容器标识 | 对应的底层真实类 | 功能描述 |
| :--- | :--- | :--- | :--- |
| `Anon\Core\Facade\Route` | `router` | `Anon\Core\Routing\Router` | 注册路由、路由组 |
| `Anon\Core\Facade\DB` | `db` | `Anon\Core\Database\Connection` | 执行数据库查询构建 |
| `Anon\Core\Facade\Log` | `log` | `Anon\Core\Log\Manager` | 记录应用日志 |
| `Anon\Core\Facade\Env` | `env` | `Anon\Core\Support\Env` | 读取 `.env` 环境变量 |

---

## 使用示例

在不使用 Facade 的情况下，如果你需要记录日志，你可能需要去获取容器单例，然后解析出日志对象：

```php
use Anon\Core\Foundation\App;

// 繁琐的调用
$logger = App::getInstance()->make('log');
$logger->info('User logged in.');
```

使用了 Facade 后，代码变得异常简洁：

```php
use Anon\Core\Facade\Log;

// 调用
Log::info('User logged in.');
```

---

## 自定义门面

你可以非常轻松地为自己的业务类创建 Facade：

1. 将你的服务类注册到容器中（例如在 App 启动阶段）：
   ```php
   App::getInstance()->instance('payment', new PaymentService());
   ```

2. 创建一个继承 `Facade` 的类：
   ```php
   namespace Anon\Facade;
   
   use Anon\Core\Facade\Facade;
   
   class Payment extends Facade
   {
       protected static function getFacadeAccessor(): string
       {
           return 'payment'; // 对应容器里的名字
       }
   }
   ```

3. 现在你可以到处使用它了：
   ```php
   \Anon\Facade\Payment::process($orderId);
   ```
