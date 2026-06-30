# Session 管理

Anon Framework Next 提供了对 PHP 原生 Session 的封装，同时支持 `file` 和 `redis` 等驱动。

## 配置

推荐在 `anon.config.php` 中只保留 Session 自身行为配置；如果使用 Redis 驱动，连接信息可以继续放在 `cache.redis` 或 `.env*` 中：

```php
use Anon\Core\Facade\Env;

return [
    'session' => [
        'driver' => Env::get('SESSION_DRIVER', 'file'),
        'lifetime' => 86400,
        'path' => '/',
        'domain' => Env::get('SESSION_DOMAIN', ''),
        'secure' => false,
        'httponly' => true,
        'samesite' => 'Lax',
        'prefix' => Env::get('SESSION_PREFIX', 'anon:session:'),
        'path_storage' => __DIR__ . '/runtime/session',
    ],
];
```

当 `SESSION_DRIVER=redis` 时，框架会优先尝试读取：

- `session.redis`
- `cache.redis`
- 旧的顶层 `redis`
- 最后回退到 `.env*`

兼容模式下，仍可继续只通过 `.env` 配置：

```env
# Session 驱动 (file 或 redis)
SESSION_DRIVER=file

# Cookie 存活时间（秒）
SESSION_LIFETIME=86400

# Redis 配置 (当 SESSION_DRIVER=redis 时生效)
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DB=0
SESSION_PREFIX=anon:session:
```

## 基础使用

你可以通过 `Anon\Core\Facade\Session` 门面类来操作 Session 数据。框架在生命周期内会自动启动 Session。

### 写入 Session

```php
use Anon\Core\Facade\Session;

// 存储单个值
Session::set('user_id', 1001);
Session::set('user_name', 'Anon');
```

### 读取 Session

```php
// 获取 Session 值
$userId = Session::get('user_id');

// 获取 Session 值，并提供默认值
$role = Session::get('role', 'guest');
```

### 判断 Session 是否存在

```php
if (Session::has('user_id')) {
    // 存在
}
```

### 删除 Session

```php
// 删除单个键
Session::delete('user_name');

// 清空当前用户的全部 Session 数据
Session::clear();
```

### 销毁 Session

销毁 Session 会清空所有数据并使得当前 Session ID 及 Cookie 失效，通常用于退出登录时：

```php
Session::destroy();
```

### 重新生成 Session ID

为了防止 Session 固定攻击（Session Fixation），在用户登录成功后推荐重新生成 Session ID：

```php
// 默认会自动删除旧的 Session 文件
Session::regenerateId();
```
