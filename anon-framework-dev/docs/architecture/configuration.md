# 项目配置 (Configuration)

Anon Framework Next 引入了极简的 Vite 风格配置体系，支持通过项目根目录下的 `anon.config.php` 管理结构化配置，同时保留 `.env` 作为敏感信息和环境差异值的来源。

## 极简设计哲学

在现代前端开发中，像 `vite.config.ts` 一样，配置文件往往是空空如也的，只有当你需要覆盖框架的**默认行为**时，你才会往里面写东西。

我们把这种极致精简带到了后端。在框架内部，所有核心模块（数据库、缓存、会话、日志、队列等）都预设了开箱即用的默认值，并内置了对环境变量的 Fallback 处理。

因此，你的 `anon.config.php` 默认长这样：

```php
<?php

use Anon\Core\Support\Config;
use Anon\Core\Facade\Env;

/**
 * Anon Framework Next 核心配置文件
 *
 * 框架内建了极为完善的默认配置，所有敏感信息均可通过 `.env` 环境变量配置。
 * 你可以像使用 `vite.config.ts` 一样，在这里仅写入你想要覆盖或自定义的配置项。
 * 
 * @see https://anon.miomoe.cn/guide/architecture/configuration
 */

return Config::define([
    
    // 例如：你可以取消注释来修改默认的应用名称
    // 'app' => [
    //     'name' => 'Anon Framework Next'
    // ]
]);
```

这种默认方式最适合新项目起步：不再有臃肿且让人摸不着头脑的一大串预设数组，保持项目的绝对纯净。

## 加载顺序

应用启动时会先加载环境文件，随后加载项目根目录下的 `anon.config.php`。

当前支持以下环境文件：

- `.env`
- `.env.local`
- `.env.{APP_ENV}`
- `.env.{APP_ENV}.local`

## 按需覆盖 (Overrides)

虽然配置是空的，但框架允许你在里面按需覆盖任何模块的内部配置。例如：

如果你想修改缓存的前缀：

```php
return Config::define([
    'cache' => [
        'prefix' => 'my_custom_project:',
    ],
]);
```

如果你想强制开启 JWT 双 Token 刷新机制：

```php
return Config::define([
    'auth' => [
        'refresh_enabled' => true,
    ],
]);
```

如果你想自定义控制多套用户体系 (Guard)：

```php
return Config::define([
    'auth' => [
        'guards' => [
            'admin' => [
                'ttl' => 7200,
            ],
            'api' => [
                'ttl' => 86400,
            ],
        ],
    ],
]);
```

## 读取配置

框架提供了 `Config` 门面，可通过点号语法读取配置：

```php
use Anon\Core\Facade\Config;

$uploadPath = Config::get('upload.path');
$dbName = Config::get('database.database');
```

## 完整配置项参考 (Full Reference)

虽然你不需要把这些都写进 `anon.config.php`，但当你需要覆盖默认行为时，可以参考以下框架内部的完整默认配置结构。

当前更推荐在配置文件里通过 `Env::get()` 读取环境值，而不是在业务代码里到处直接写 `getenv()`。这样可以同时兼容 `putenv()` 被禁用、以及 `$_ENV` / `$_SERVER` 降级读取的场景：

```php
<?php

use Anon\Core\Support\Config;

return Config::define([
    
    // ----------------------------------------------------------------------
    // 1. 基础应用配置
    // ----------------------------------------------------------------------
    'app' => [
        'name'  => Env::get('APP_NAME', 'Anon Framework Next'),
        'env'   => Env::get('APP_ENV', 'production'),
        'debug' => (bool) Env::get('APP_DEBUG', false),
        'url'   => Env::get('APP_URL', 'http://localhost'),
    ],

    // ----------------------------------------------------------------------
    // 2. 数据库配置
    // ----------------------------------------------------------------------
    'database' => [
        'type'     => Env::get('DATABASE_TYPE', 'mysql'),
        'host'     => Env::get('DATABASE_URL', '127.0.0.1'),
        'port'     => (int) Env::get('DATABASE_PORT', 3306),
        'database' => Env::get('DATABASE_NAME', 'anon'),
        'username' => Env::get('DATABASE_USER', 'root'),
        'password' => Env::get('DATABASE_PASSWORD', ''),
        'charset'  => Env::get('DATABASE_CHARSET', 'utf8mb4'),
        'prefix'   => Env::get('DATABASE_PREFIX', ''),
    ],

    // ----------------------------------------------------------------------
    // 3. 缓存配置
    // ----------------------------------------------------------------------
    'cache' => [
        'default' => Env::get('CACHE_DRIVER', 'file'), // 可选: file, redis
        'path'    => BASE_PATH . '/runtime/cache',
        'prefix'  => 'anon:cache:',
        'redis'   => [
            'host'     => Env::get('REDIS_HOST', '127.0.0.1'),
            'port'     => (int) Env::get('REDIS_PORT', 6379),
            'password' => Env::get('REDIS_PASSWORD', ''),
            'database' => (int) Env::get('REDIS_DB', 0),
        ],
    ],

    // ----------------------------------------------------------------------
    // 4. Session 会话配置
    // ----------------------------------------------------------------------
    'session' => [
        'driver'       => Env::get('SESSION_DRIVER', 'file'),
        'lifetime'     => (int) Env::get('SESSION_LIFETIME', 86400),
        'path'         => '/',
        'domain'       => Env::get('SESSION_DOMAIN', ''),
        'secure'       => (bool) Env::get('SESSION_SECURE', false),
        'httponly'     => true,
        'samesite'     => 'Lax',
        'prefix'       => 'anon:session:',
        'path_storage' => BASE_PATH . '/runtime/session',
    ],

    // ----------------------------------------------------------------------
    // 5. JWT 认证与守卫配置
    // ----------------------------------------------------------------------
    'auth' => [
        'default_guard'   => 'api',
        'refresh_enabled' => false,
        'token_sources'   => ['header', 'cookie'], // 默认从 Header 和 Cookie 解析
        'guards' => [
            'api' => [
                'ttl'    => 86400, // Token 有效期
            ],
            // 你可以自行追加 admin guard
        ],
    ],

    // ----------------------------------------------------------------------
    // 6. 文件上传与存储
    // ----------------------------------------------------------------------
    'upload' => [
        'path' => BASE_PATH . '/run/storage',
    ],

    // ----------------------------------------------------------------------
    // 7. 异步队列配置
    // ----------------------------------------------------------------------
    'queue' => [
        'default'   => 'default',
        'prefix'    => 'anon:queue:',
        'max_tries' => 3,
    ],

    // ----------------------------------------------------------------------
    // 8. 日志系统
    // ----------------------------------------------------------------------
    'log' => [
        'path' => BASE_PATH . '/runtime/log',
    ],

    // ----------------------------------------------------------------------
    // 9. 开发服务器配置 (Dev Server)
    // ----------------------------------------------------------------------
    'server' => [
        'host' => '127.0.0.1',
        'port' => 8000,
    ],

    // ----------------------------------------------------------------------
    // 10. HTTP 客户端配置
    // ----------------------------------------------------------------------
    'http' => [
        'ssl_verify' => true,
    ],
]);
```

## 配置缓存

在生产环境中为了提高性能，避免频繁的文件 IO 读取和数组合并，框架支持将配置预编译为单文件缓存：

```bash
# 生成配置缓存
php anon config:cache

# 清理配置缓存
php anon config:clear
```

当配置被缓存后，系统将直接从 `runtime/cache/config.php` 加载静态数组，此时 `.env` 环境变量文件将不再被重新解析，所有通过 `Env::get()` 或环境读取链得到的值都会被固定为缓存生成时的结果。

**注意：** 如果你在修改了 `.env` 或者 `anon.config.php` 之后发现配置没有生效，请确保先运行 `php anon config:clear`。

## 当前已接入模块

目前以下核心模块支持从 `anon.config.php` 读取配置，但骨架默认不会全部预置：

- `app`
- `database`
- `redis`
- `cache`
- `session`
- `storage`
- `auth`
- `queue`
- `log`
- `server`

## 迁移建议

- 旧项目可以先保留 `.env` 原配置不动。
- 新增 `anon.config.php`，只迁移你想结构化管理的部分。
- 待项目稳定后，再逐步把非敏感配置从 `.env` 挪到 `anon.config.php`。
