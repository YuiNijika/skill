# 环境变量 (Env)

Anon Framework Next 提供了对 `.env` 文件的原生支持。当前推荐将结构化配置写在 `anon.config.php` 中，而将 `.env` 用作敏感值和环境差异值的来源。

默认情况下，项目骨架在根目录下会包含一个 `anon.config.php` 与 `.env`。

## 分层环境文件

框架现已支持类似 Vite 的分层环境文件加载方式：

- `.env`
- `.env.local`
- `.env.development`
- `.env.production`
- `.env.{APP_ENV}.local`

加载顺序如下：

1. `.env`
2. `.env.local`
3. `.env.{APP_ENV}`
4. `.env.{APP_ENV}.local`

后加载的文件会覆盖前面的值；如果某个变量已经由系统环境变量、部署平台或 CLI 显式注入，则不会被 `.env*` 文件覆盖。

## 推荐搭配

- `anon.config.php`：推荐写成结构化配置，业务代码统一通过 `Config::get()` 读取；如果配置文件里需要读取环境值，优先使用 `Env::get()`。
- `.env`：由于只用于管理敏感值，推荐仅保留 `DATABASE_`、`JWT_SECRET`、第三方密钥这类环境差异值，不要把所有业务配置都塞回 `.env`。

推荐不要在业务代码里直接依赖裸 `getenv()`，因为线上环境可能禁用了 `putenv()`，或者只通过 `$_ENV` / `$_SERVER` 暴露变量。框架已经提供了更稳妥的 `Env::get()` 读取链。

## 基础配置

默认骨架中的 `.env` 保持极致简洁：

```env
[DATABASE]
DATABASE_TYPE=mysql
DATABASE_URL=127.0.0.1
DATABASE_PORT=3306
DATABASE_USER=root
DATABASE_PASSWORD=
DATABASE_NAME=anon
```

如果项目后续需要接入更多受 `.env` 保护的敏感配置，再逐步往里加即可。

### APP_ENV 环境变量

`APP_ENV` 代表了**当前应用程序的运行环境标识 (Application Environment)**。常见的定义有：

- **`local` (本地开发环境)**：往往开启详细的报错信息、使用本地轻量级的缓存驱动。
- **`testing` (测试环境)**：通常用于持续集成 (CI) 或专门的测试服务器。
- **`staging` (预发布环境)**：代码发布到正式生产服务器前的一个环境，配置与生产环境几乎一模一样。
- **`production` (正式生产环境)**：必须关闭所有的调试报错信息，开启严格的缓存和性能优化配置。

在业务代码中，你可以通过判断该标识来执行不同的逻辑：

```php
use Anon\Core\Facade\Env;

if (APP_ENV === 'local') {
    // 调用 Mock 接口
} else {
    // 调用真实线上接口
}
```

## 全局常量

框架在启动时，会自动将最常用的几个环境变量解析并挂载为 PHP 全局常量，方便在项目的任何地方直接使用，无需反复调用 `Env::get()`：

- **`APP_NAME`**：应用名称。
- **`APP_ENV`**：运行环境。
- **`APP_URL`**：应用外网访问地址（未定义时会自动根据当前 HTTP 请求智能推导）。
- **`DEBUG_MODE`**：调试模式布尔值（为 `true` 时发生致命错误会直接在页面上打印错误堆栈）。

```php
// 直接使用全局常量
echo "当前环境: " . APP_ENV;
echo "访问地址: " . APP_URL;

if (DEBUG_MODE) {
    // ...
}
```

## 获取环境变量

你可以通过 `Anon\Core\Facade\Env` 门面来获取环境变量。

```php
use Anon\Core\Facade\Env;
// 获取 DEBUG_MODE，如果不存在则返回 false
$isDebug = Env::get('DEBUG_MODE', false);
```

如果你需要读取结构化配置，请优先使用 `Config` 门面：

```php
use Anon\Core\Facade\Config;

$cacheDriver = Config::get('cache.default', 'file');
```

### 数据类型自动转换

`.env` 文件中所有的值本质上都是字符串，但 Env 组件在解析时会自动将其转换为 PHP 的标量类型：

- `true`, `on`, `yes` 会被转换为布尔值 `true`
- `false`, `off`, `no` 会被转换为布尔值 `false`
- `null`, `empty` 会被转换为 `null`
- 纯数字会被转换为对应的 `int` 或 `float`

## 动态设置环境变量

你也可以在运行时动态修改或追加环境变量。修改后的变量会被同步设置到 PHP 原生的 `$_ENV` 以及 `putenv()` 中。

```php
use Anon\Core\Facade\Env;

// 动态设置环境标识
Env::set('APP_ENV', 'production');
```
