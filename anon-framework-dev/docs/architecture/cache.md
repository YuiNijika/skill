# 缓存系统 (Cache)

Anon Framework Next 提供了统一的缓存系统接口，目前支持 `File` 和 `Redis` 两种驱动。

## 配置

推荐把缓存默认配置和 Redis 连接一起写在 `cache` 下，在 `.env*` 中存放具体连接值：

```php
use Anon\Core\Facade\Env;

return [
    'cache' => [
        'default' => 'file',
        'path' => __DIR__ . '/runtime/cache',
        'prefix' => Env::get('CACHE_PREFIX', 'anon:cache:'),
        'redis' => [
            'host' => Env::get('REDIS_HOST', '127.0.0.1'),
            'port' => (int) Env::get('REDIS_PORT', 6379),
            'password' => Env::get('REDIS_PASSWORD', ''),
            'database' => (int) Env::get('REDIS_DB', 0),
        ],
    ],
];
```

如果你的 Session、Queue 也使用 Redis，但不想在多个地方重复写连接信息，也可以直接复用这组 `cache.redis` 配置。

兼容模式下，仍可继续通过 `.env` 配置：

```env
# 默认缓存驱动: file 或 redis
CACHE_DRIVER=file

# Redis 缓存配置
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DB=0
CACHE_PREFIX=anon:cache:
```

## 基础使用

框架提供了 `Anon\Core\Facade\Cache` 门面供全局静态调用。

### 写入缓存

```php
use Anon\Core\Facade\Cache;

// 写入缓存，第三个参数为过期时间（秒）
Cache::set('user_1', ['name' => 'Anon'], 3600);
```

### 读取缓存

```php
// 读取缓存，如果不存在则返回 null
$user = Cache::get('user_1');

// 读取缓存，并指定默认值
$user = Cache::get('user_1', ['name' => 'Default']);
```

### 判断缓存是否存在

```php
if (Cache::has('user_1')) {
    // 缓存存在
}
```

### 删除与清空缓存

```php
// 删除指定键名的缓存
Cache::delete('user_1');

// 清空当前驱动的所有缓存数据
Cache::clear();
```

---

## 高级特性

### 1. 缓存旁路 (Cache Aside)

使用 `remember()` 方法，实现有缓存直接返回，无缓存则执行闭包写入缓存后返回：

```php
use Anon\Core\Facade\Cache;
use Anon\Core\Facade\DB;

// 获取用户列表，缓存 1 小时
$users = Cache::remember('user_list', 3600, function () {
    return DB::table('users')->where('status', 1)->get();
});
```

### 2. 阅后即焚 (Pull)

获取一个缓存值，并在获取后立即将其删除：

```php
// 获取并立即删除
$code = Cache::pull('sms_code_13800138000', 'expired');
```

### 3. 原子增减 (Increment / Decrement)

使用 `increment` 和 `decrement` 方法实现原子计数：

```php
// 浏览量 +1
Cache::increment('article_views_100');

// 浏览量 +5
Cache::increment('article_views_100', 5);

// 减少库存数量
Cache::decrement('goods_stock_100');
```

---

## 切换驱动

可以在运行时临时切换驱动：

```php
// 显式使用 redis 驱动
Cache::store('redis')->set('key', 'value');

// 显式使用 file 驱动
Cache::store('file')->get('key');
```
