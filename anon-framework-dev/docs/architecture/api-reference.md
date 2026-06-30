# API 参考

Anon Framework Next 提供了一些内置方法（通过 Facade 外观类和核心对象提供）。本章节汇总了框架中常用的 API 与内置常量。

---

## 1. 框架内置常量

框架在启动时会注册全局路径和环境常量：

| 常量名称 | 描述 | 示例值 |
|---|---|---|
| `BASE_PATH` | 项目根目录（通常为 `skeleton` 目录） | `/var/www/project` |
| `APP_PATH` | 应用核心目录（`app` 目录） | `/var/www/project/app` |
| `RUNTIME_PATH` | 运行时目录（日志、缓存等存放地） | `/var/www/project/runtime` |
| `APP_NAME` | 应用名称（读取自 `.env`） | `Anon Next` |
| `APP_ENV` | 运行环境（读取自 `.env`） | `production` 或 `local` |
| `DEBUG_MODE` | 调试模式开关（读取自 `.env`） | `true` 或 `false` |
| `APP_URL` | 应用的 URL（自动识别或读取自 `.env`） | `http://localhost:8000` |

---

## 2. 环境与配置 (Env / Config)

使用 `Anon\Core\Facade\Env` 获取环境变量，使用 `Anon\Core\Facade\Config` 获取 `anon.config.php` 中的结构化配置。

```php
use Anon\Core\Facade\Env;
use Anon\Core\Facade\Config;

// 获取环境变量，如果不存在则返回默认值 'default_value'
$value = Env::get('KEY_NAME', 'default_value');

// 获取结构化配置
$driver = Config::get('cache.default', 'file');
```

---

## 3. 日志记录 (Log)

使用 `Anon\Core\Facade\Log` 记录日志，日志自动按天切割并隔离分类。

```php
use Anon\Core\Facade\Log;

// 记录普通信息日志 (默认写入 app-YYYY-MM-DD.log)
Log::info('用户登录成功');

// 记录带上下文数组的日志
Log::info('用户操作', ['user_id' => 1001, 'action' => 'update_profile']);

// 指定独立的日志分类 (会写入 auth-YYYY-MM-DD.log)
Log::info('用户登录成功', 'auth');

// 记录错误日志
Log::error('数据库连接失败');

// 记录调试日志
Log::debug('当前内存消耗: ' . memory_get_usage());
```

---

## 4. 缓存管理 (Cache)

使用 `Anon\Core\Facade\Cache` 管理缓存。

```php
use Anon\Core\Facade\Cache;

// 设置缓存，有效期 3600 秒
Cache::set('user_list', $users, 3600);

// 获取缓存，如果不存在返回 null
$users = Cache::get('user_list');

// 获取缓存，如果不存在返回默认值 []
$users = Cache::get('user_list', []);

// 判断缓存是否存在
$exists = Cache::has('user_list');

// 删除缓存
Cache::delete('user_list');

// 自增/自减（常用于计数器，Redis驱动下性能极佳）
Cache::increment('view_count', 1);
```

---

## 5. 会话状态 (Session)

使用 `Anon\Core\Facade\Session` 来管理客户端 Session 状态。

```php
use Anon\Core\Facade\Session;

// 获取 Session
$userId = Session::get('user_id');

// 获取并设置默认值
$role = Session::get('role', 'guest');

// 设置 Session
Session::set('user_id', 1001);

// 判断 Session 是否存在
if (Session::has('user_id')) { ... }

// 删除 Session
Session::delete('user_id');

// 获取当前 Session ID
$sessionId = Session::getId();
```

---

## 6. 文件存储 (Storage)

使用 `Anon\Core\Facade\Storage` 进行文件管理。默认存储在 `RUNTIME_PATH/storage`。

```php
use Anon\Core\Facade\Storage;

// 写入文件内容
Storage::put('avatars/1.png', $fileContent);

// 读取文件内容
$content = Storage::get('avatars/1.png');

// 判断文件是否存在
$exists = Storage::exists('avatars/1.png');

// 获取文件的相对访问 URL
$url = Storage::url('avatars/1.png');
```

---

## 7. 身份认证 (Auth)

使用 `Anon\Core\Facade\Auth` 进行无状态的 JWT 登录与鉴权。

```php
use Anon\Core\Facade\Auth;

// 生成 JWT Token (默认有效期 7200 秒)
$token = Auth::login(['id' => 1001, 'name' => 'admin']);

// 检查当前请求是否携带了有效的 Token
$isLogged = Auth::check();

// 获取当前解析后的 Token 载荷（例如包含用户信息）
$user = Auth::user();
```

---

## 8. 事件与钩子 (Event / Hook)

使用 `Anon\Core\Facade\Event` 和 `Anon\Core\Facade\Hook` 实现事件驱动开发。

```php
use Anon\Core\Facade\Event;
use Anon\Core\Facade\Hook;

// --- 事件 (Event) ---
// 注册事件监听器
Event::listen('user.registered', function ($user) {
    // 发送欢迎邮件...
});

// 触发事件 (会同步执行所有监听器)
Event::dispatch('user.registered', ['id' => 1, 'name' => 'Anon']);

// --- 钩子 (Hook) ---
// Hook 底层继承自 Event，常用于系统级切面拦截
Hook::add('request_begin', function ($request) {
    // 请求开始时的预处理...
});

Hook::trigger('request_begin', $request);
```

---

## 9. 标准化响应 (Response)

`Anon\Core\Http\Response` 统一 API 输出格式。

```php
use Anon\Core\Http\Response;

// 返回成功响应
// {"code": 200, "message": "success", "data": {"id": 1}}
return Response::success(['id' => 1]);

// 返回带自定义消息的成功响应
return Response::success(null, '操作成功');

// 返回失败响应
// {"code": 400, "message": "参数错误", "data": null}
return Response::error('参数错误', 400);

// 返回原生 JSON 与自定义状态码
return Response::json(['status' => 'ok'], 201);
```

---

## 10. 容器级方法 (App / Container)

`App` 本身是一个容器单例。如果你需要手动解析服务、获取容器单例或实例化带有依赖注入的类，可以使用 `App` 容器。

```php
use Anon\Core\Foundation\App;

// 获取容器单例
$app = App::getInstance();

// 自动依赖注入并实例化类
$controller = $app->make(UserController::class);

// 获取框架信息
$info = $app->getInfo();
// 返回 ['name' => 'Anon Next', 'version' => 'v4.0.0-next', ...]
```
