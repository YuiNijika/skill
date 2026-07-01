# 数据库与查询构建器

Anon Framework Next 提供 PDO 驱动的 SQL 数据库层，以及独立的 MongoDB 数据库层。

- SQL：`Connection + QueryBuilder + Model`
- Mongo：`Mongo\Connection + Mongo\QueryBuilder + Mongo\ModelQueryBuilder`
- 迁移：当前以原生 SQL 和 Mongo schema helper 为主，不提供统一字段级 Schema Builder

## 数据库驱动支持范围

当前 `Connection` 层可识别这些数据库类型：

- `mysql`
- `pgsql`
- `sqlite`
- `sqlsrv`
- `oracle`
- `oci`
- `mongo`
- `mongodb`

- `mysql` / `pgsql` / `sqlite` / `sqlsrv` / `oracle` / `oci`：走 `PDO + SQL QueryBuilder`
- `mongo` / `mongodb`：走独立的 Mongo 连接与非 SQL QueryBuilder
- “支持”表示已接入连接层或查询层，不表示所有驱动语义完全等价

Mongo 相关类已拆到独立目录：

- `Anon\Core\Database\Mongo\Connection`
- `Anon\Core\Database\Mongo\QueryBuilder`
- `Anon\Core\Database\Mongo\ModelQueryBuilder`
- `Anon\Core\Database\Model\QueryBuilder`

支持状态建议这样理解：

- `mysql` / `sqlite`：当前最稳
- `pgsql` / `sqlsrv` / `oracle` / `oci`：已接入，但要关注方言差异
- `mongo` / `mongodb`：已接入，但不是 SQL 等价实现

SQL 驱动当前需要特别注意：

- 标识符包装符不同
- `LIKE` / `ILIKE` / 分页 / `UPSERT` 已做驱动分支，但仍建议实库验证
- JSON 查询已补路径取值比较，`contains` 只在 `mysql/pgsql` 开放
- 迁移 SQL 需要按目标数据库自己写原生语法
- 迁移系统只保证迁移记录表初始化，业务迁移内容不做跨库字段抽象

Mongo 当前能力：

- 独立连接、CRUD、排序、分页、正则、`upsert`、模型查询、软删除、基础聚合管道
- `_id`、嵌套 BSON 条件、常见 `*_id` 外键会做基础归一化
- 默认主键 `id` 会自动映射为 `_id`
- 默认外键命名仍保持 `user_id`

Mongo 当前不支持：

- `join()`
- `having()`
- `selectRaw()`
- `orderByRaw()`

Mongo 事务依赖会话能力，通常要求副本集或分片集群。

## 字段类型说明

当前框架**没有提供统一的 Schema Builder 字段抽象层**：

- 迁移里的字段类型请直接写目标数据库自己的原生 SQL 类型
- `VARCHAR`, `TEXT`, `INT`, `BIGINT`, `TIMESTAMP`, `JSON`, `UUID` 等写法是否可用，取决于你当前使用的数据库
- 如果你需要跨 MySQL / PostgreSQL / SQLServer / Oracle 共用同一套迁移 SQL，需要自己处理方言差异

例如：

- MySQL 常见写法：`INT AUTO_INCREMENT PRIMARY KEY`
- SQLite 常见写法：`INTEGER PRIMARY KEY AUTOINCREMENT`
- PostgreSQL 常见写法：`GENERATED ... AS IDENTITY` 或 `SERIAL`
- SQL Server 常见写法：`INT IDENTITY(1,1) PRIMARY KEY`
- Oracle 常见写法：`NUMBER`、`VARCHAR2`，以及序列或 identity 语法

## 配置

通过项目根目录的 `anon.config.php` 配置数据库连接，通过 `.env*` 保存敏感值。配置文件建议统一使用 `Env::get()`：

```php
use Anon\Core\Facade\Env;

return [
    'database' => [
        'type' => Env::get('DATABASE_TYPE', 'mysql'),
        'host' => Env::get('DATABASE_URL', '127.0.0.1'),
        'port' => (int) Env::get('DATABASE_PORT', 3306),
        'database' => Env::get('DATABASE_NAME', 'anon_test'),
        'username' => Env::get('DATABASE_USER', 'root'),
        'password' => Env::get('DATABASE_PASSWORD', ''),
        'charset' => Env::get('DATABASE_CHARSET', 'utf8mb4'),
        'prefix' => Env::get('DATABASE_PREFIX', ''),
    ],
];
```

配合 `.env` 文件保存敏感值：

```env
[DATABASE]
DATABASE_TYPE=mysql
DATABASE_URL=127.0.0.1
DATABASE_PORT=3306
DATABASE_USER=root
DATABASE_PASSWORD=root
DATABASE_NAME=anon_test
DATABASE_PREFIX=
```

Mongo 配置示例：

```php
use Anon\Core\Facade\Env;

return [
    'database' => [
        'type' => Env::get('DATABASE_TYPE', 'mongodb'),
        'host' => Env::get('DATABASE_URL', '127.0.0.1'),
        'port' => (int) Env::get('DATABASE_PORT', 27017),
        'database' => Env::get('DATABASE_NAME', 'anon'),
        'username' => Env::get('DATABASE_USER', ''),
        'password' => Env::get('DATABASE_PASSWORD', ''),
        'auth_source' => Env::get('DATABASE_AUTH_SOURCE', 'admin'),
        'prefix' => '',
    ],
];
```

对应 `.env`：

```env
[DATABASE]
DATABASE_TYPE=mongodb
DATABASE_URL=127.0.0.1
DATABASE_PORT=27017
DATABASE_USER=
DATABASE_PASSWORD=
DATABASE_NAME=anon
DATABASE_AUTH_SOURCE=admin
```

### Mongo 扩展查询

Mongo 额外提供两类非 SQL 风格接口：

```php
use Anon\Core\Facade\DB;

// 直接追加文档过滤条件
$rows = DB::table('users')
    ->where('status', 1)
    ->whereDocument([
        '$or' => [
            ['profile.city' => 'Tokyo'],
            ['tags' => ['$in' => ['vip']]],
        ],
    ])
    ->get();
```

```php
use Anon\Core\Facade\DB;

// 直接执行聚合管道
$rows = DB::table('orders')
    ->where('status', 'paid')
    ->aggregatePipeline([
        [
            '$group' => [
                '_id' => '$user_id',
                'total_amount' => ['$sum' => '$amount'],
            ],
        ],
        [
            '$sort' => ['total_amount' => -1],
        ],
    ]);
```

基础统计仍然可以直接用：

```php
DB::table('orders')->where('status', 'paid')->count();
DB::table('orders')->where('status', 'paid')->aggregate('SUM', 'amount');
DB::table('orders')->where('status', 'paid')->sum('amount');
DB::table('orders')->avg('score');
DB::table('orders')->min('amount');
DB::table('orders')->max('amount');
```

结果提取和时间排序快捷方法：

```php
DB::table('users')->latest()->pluck('email');
DB::table('users')->oldest('created_at')->pluck('name', 'id');
DB::table('users')->where('id', 1)->value('email');
```

Mongo 查询构建器已补常用链式方法：

```php
DB::table('users')
    ->where('status', 1)
    ->orWhere('role', 'admin')
    ->orWhereIn('user_id', ['6863d9a6f58e8a7e7d6f1234'])
    ->whereNotBetween('score', [0, 59])
    ->orWhereNull('deleted_at')
    ->get();
```

Mongo 模型聚合结果也可以直接 hydrate 或分页：

```php
$models = User::query()->aggregateModels([
    ['$match' => ['status' => 1]],
]);
```

```php
$result = User::query()->aggregatePaginate(
    [
        ['$match' => ['status' => 1]],
    ],
    20,
    1,
    true
);
```

`aggregatePaginate(..., true)` 的最后一个参数表示是否 hydrate 成模型实例。

Mongo 文档型更新操作：

```php
DB::table('users')->where('_id', $id)->increment('login_count');
DB::table('users')->where('_id', $id)->decrement('credits', 5);
DB::table('users')->where('_id', $id)->push('tags', 'vip', true);
DB::table('users')->where('_id', $id)->pull('tags', 'banned');
DB::table('users')->where('_id', $id)->unset(['legacy_field']);
DB::table('users')->where('_id', $id)->renameField('nickname', 'display_name');
```

Mongo 集合和索引也可以直接走连接对象：

```php
$mongo = app('db');

$mongo->createCollection('audit_logs');
$mongo->createIndex('audit_logs', ['user_id' => 1, 'created_at' => -1], ['unique' => false]);
$mongo->listIndexes('audit_logs');
$mongo->dropIndex('audit_logs', ['user_id' => 1, 'created_at' => -1]);
```

也可以在查询构建器上直接操作当前 collection 的索引：

```php
DB::table('audit_logs')->createIndex(['user_id' => 1]);
DB::table('audit_logs')->listIndexes();
DB::table('audit_logs')->dropIndex(['user_id' => 1]);
```

Mongo 连接还提供了轻量 schema helper：

```php
use Anon\Core\Facade\DB;

DB::schema()->create('audit_logs', function ($collection) {
    $collection->index(['user_id' => 1, 'created_at' => -1]);
    $collection->ttl('expired_at', 0);
    $collection->validator([
        'bsonType' => 'object',
        'required' => ['user_id', 'action'],
        'properties' => [
            'user_id' => ['bsonType' => 'objectId'],
            'action' => ['bsonType' => 'string'],
        ],
    ]);
});
```

也可以直接拿 collection helper：

```php
$users = DB::schema()->table('users');

$users->unique(['email' => 1]);
$users->text(['nickname', 'bio']);
$users->hashed('tenant_id');
$users->indexes();
```

`DB::schema()` 当前是 Mongo 连接专用 helper，SQL 驱动下不会返回通用 SchemaBuilder。

Mongo 迁移类也可以直接使用基类里的 helper：

```php
use Anon\Core\Database\Migration\Migration;

class CreateAuditLogs extends Migration
{
    public function up(): void
    {
        $this->schema()->create('audit_logs', function ($collection) {
            $collection->index(['user_id' => 1, 'created_at' => -1]);
        });
    }

    public function down(): void
    {
        $this->schema()->dropIfExists('audit_logs');
    }
}
```

迁移运行器本身已兼容 Mongo，不再强依赖 `PDO` 记录迁移状态。

模型关系字段如果明确就是 `ObjectId` 语义，可以直接用显式 helper：

```php
public function user()
{
    return $this->objectIdBelongsTo(User::class);
}

public function posts()
{
    return $this->objectIdHasMany(Post::class);
}
```

这样会直接按 Mongo 常见的 `_id / user_id` 规则建关系，减少手动传键名。

如果不是 `mysql`，建议显式配置这些值：

- `DATABASE_TYPE`
- `DATABASE_CHARSET`
- `DATABASE_PORT`

默认值本身更偏向 MySQL 场景。

## 数据库事务

查询构建器提供了事务支持：

```php
try {
    DB::beginTransaction();

    DB::table('users')->where('id', 1)->update(['balance' => 100]);
    DB::table('orders')->insert(['user_id' => 1, 'amount' => 50]);

    DB::commit();
} catch (\Exception $e) {
    DB::rollBack();
    throw $e;
}
```

也支持事务回调写法：

```php
DB::transaction(function () {
    DB::table('users')->where('id', 1)->update(['balance' => 100]);
    DB::table('orders')->insert(['user_id' => 1, 'amount' => 50]);
});
```

### 执行原生 SQL

```php
use Anon\Core\Facade\DB;

// 执行查询
$users = DB::select('SELECT * FROM users WHERE status = ?', [1]);

// 执行写入/更新/删除等操作
$affected = DB::statement('UPDATE users SET status = ? WHERE id = ?', [0, 1]);
```

### 什么时候应该优先写原生 SQL

下面这些情况优先直接写原生 SQL：

- 需要跨数据库兼容地建表
- 需要用到数据库方言能力
- 需要复杂聚合、窗口函数、CTE
- 需要数据库特有的 JSON / ARRAY / 高级 UPSERT 语法

### 查询构建器 (Query Builder)

#### 查询数据

```php
// 获取所有数据
$users = DB::table('users')->get();

// 指定查询字段
$users = DB::table('users')->select(['id', 'name', 'email'])->get();

// 获取单条数据
$user = DB::table('users')->where('id', 1)->first();

// 获取数据总数
$count = DB::table('users')->where('status', 1)->count();

// 判断记录是否存在
$exists = DB::table('users')->where('email', 'admin@example.com')->exists();
```

#### 条件查询

```php
// AND 条件
DB::table('users')->where('status', 1)->where('role_id', 2)->get();

// OR 条件
DB::table('users')->where('status', 1)->orWhere('role_id', 2)->get();

// IN 查询
DB::table('users')->whereIn('id', [1, 2, 3])->get();
DB::table('users')->whereNotIn('id', [4, 5, 6])->get();

// BETWEEN 查询
DB::table('users')->whereBetween('created_at', ['2026-01-01', '2026-12-31'])->get();
DB::table('users')->whereNotBetween('score', [60, 100])->get();

// NULL 查询
DB::table('users')->whereNotNull('email')->get();

// LIKE 查询
DB::table('users')->whereLike('name', '%anon%')->get();
DB::table('users')->whereNotLike('email', '%spam%')->get();

// JSON 路径取值查询
DB::table('users')->whereJsonValue('profile', 'address.city', 'Tokyo')->get();
DB::table('users')->whereJsonNotValue('profile', 'settings.lang', 'zh-CN')->get();
```

`whereLike()` / `whereNotLike()` 会按驱动做最小兼容处理：

- `pgsql` 会优先使用 `ILIKE`
- `mysql` 在大小写敏感模式下会使用 `LIKE BINARY`
- 其他 SQL 驱动会退回到 `LOWER(column) LIKE LOWER(?)` 这一类通用写法

#### 正则条件

```php
DB::table('users')->whereRegex('name', '^anon', false)->get();
DB::table('users')->whereNotRegex('email', '@spam\\.com$', false)->get();
```

当前正则条件按驱动分为：

- `pgsql`：使用 `~` / `~*` / `!~` / `!~*`
- `mysql`：使用 `REGEXP` / `NOT REGEXP`
- `oracle` / `oci`：使用 `REGEXP_LIKE`
- `sqlsrv`：当前不会假装支持，会直接抛出异常
- `sqlite`：当前依赖自定义 `REGEXP` 函数，框架默认没有注册，也会直接抛出异常

#### JSON 路径值条件

```php
DB::table('users')->whereJsonValue('profile', 'address.city', 'Tokyo')->get();
DB::table('users')->orWhereJsonValue('profile', '$.settings.theme', 'dark')->get();
DB::table('users')->whereJsonValue('profile', 'flags.0', true)->get();
DB::table('users')->whereJsonNotValue('profile', 'meta.deleted_at', null)->get();
DB::table('users')->whereJsonLike('profile', 'bio', '%anon%')->get();
DB::table('users')->whereJsonIn('profile', 'status', ['active', 'pending'])->get();
DB::table('users')->whereJsonContains('profile', '$', ['tags' => ['vip']])->get();
```

`whereJsonValue()` 当前按驱动转成：

- `mysql`：`JSON_EXTRACT` / `JSON_UNQUOTE`
- `pgsql`：`#>>`
- `sqlite`：`json_extract`
- `sqlsrv` / `oracle` / `oci`：`JSON_VALUE`

当前限制：

- 只支持标量值比较
- `LIKE` / `IN` 这类 JSON 条件也是基于路径取值做比较
- `contains` 当前只支持 `mysql` / `pgsql`
- 路径只支持点号路径和数字索引
- JSON 对象、数组包含判断仍建议直接写原生 SQL

#### 聚合与 HAVING

```php
$rows = DB::table('orders')
    ->select(['user_id'])
    ->selectRaw('COUNT(*) as total_orders')
    ->groupBy('user_id')
    ->having('total_orders', '>', 10)
    ->get();

$rows = DB::table('orders')
    ->select(['user_id'])
    ->selectRaw('SUM(amount) as total_amount')
    ->groupBy('user_id')
    ->havingRaw('SUM(amount) > ?', [1000])
    ->get();
```

`having()` 适合常规字段比较；涉及聚合表达式或别名兼容时优先用 `havingRaw()`。

#### 排序与分页

> **注意：** 分页已按 `mysql / pgsql / sqlite / sqlsrv / oracle / oci` 做驱动分支，但 `sqlsrv`、`oracle / oci` 仍建议先在目标数据库验证。

分页和条数限制支持这几种写法：

```php
// 方式一：独立使用 limit() 和 offset()
$users = DB::table('users')
    ->orderBy('created_at', 'DESC')
    ->limit(10)
    ->offset(20)
    ->get();

// 方式二：直接在 limit() 中传入 limit 和 offset
$users = DB::table('users')
    ->orderBy('created_at', 'DESC')
    ->limit(10, 20)
    ->get();

// 方式三：使用 paginate()
$result = DB::table('users')->paginate(15);
```

#### 连表查询 (Join)

```php
$users = DB::table('users')
    ->select(['users.id', 'users.name', 'roles.role_name'])
    ->leftJoin('roles', 'users.role_id', '=', 'roles.id')
    ->get();
```

#### Raw 表达式

```php
$rows = DB::table('orders')
    ->select(['user_id'])
    ->selectRaw('SUM(amount) as total_amount')
    ->groupBy('user_id')
    ->orderByRaw('total_amount DESC')
    ->get();
```

当前提供的 raw 入口有：

- `selectRaw()`
- `havingRaw()`
- `orderByRaw()`

这些方法只适合你自己明确可控的 SQL 片段，不能把不可信用户输入直接拼进去。

#### 插入数据

```php
// 插入单条数据
$insertId = DB::table('users')->insert([
    'name' => 'Anon',
    'email' => 'anon@example.com'
]);

// UPSERT
DB::table('users')->upsert(
    [
        'email' => 'anon@example.com',
        'name' => 'Anon',
        'status' => 1,
    ],
    ['email'],
    ['name', 'status']
);
```

`upsert()` 当前分为两类实现：

- `mysql`：`ON DUPLICATE KEY UPDATE`
- `pgsql` / `sqlite`：`ON CONFLICT`
- `sqlsrv` / `oracle` / `oci`：事务包裹的 update-or-insert 回退

`insert()` / ORM `save()` 的主键回填当前规则：

- `pgsql` 会优先走 `RETURNING`
- `sqlsrv` 会优先走 `OUTPUT INSERTED`
- `oracle` / `oci` 会优先走 `RETURNING ... INTO`
- 其他驱动先取 `lastInsertId()`
- 取不到时，模型会回退到已显式写入的数据主键
- SQL 模型如果依赖 sequence，可在模型里声明 `protected ?string $sequence`

#### 批量插入数据

对于大量数据的插入，使用 `insertAll()` 可将所有数据合并为一条原生 SQL 执行。

```php
$data = [
    ['name' => 'User1', 'email' => 'user1@example.com'],
    ['name' => 'User2', 'email' => 'user2@example.com'],
    ['name' => 'User3', 'email' => 'user3@example.com']
];

// 批量插入
$affectedRows = DB::table('users')->insertAll($data);
```

#### 更新与删除数据

```php
// 更新数据
DB::table('users')->where('id', 1)->update(['status' => 0]);

// 删除数据
DB::table('users')->where('status', 0)->delete();
```

---

## 批量数据处理

### 1. 分块处理 (Chunk)

对于大量数据集的处理，可以使用 `chunk()` 方法。它会自动对数据进行分页查询。

```php
DB::table('users')->where('status', 1)->chunk(500, function ($users, $page) {
    foreach ($users as $user) {
        // 执行逻辑
    }
});
```

### 2. 游标迭代 (Cursor)

如果是用于数据导出且只需要顺序遍历，可以使用 `cursor()`。它利用 PHP 的 `Generator` 特性和 PDO 游标，减少内存使用。

```php
foreach (DB::table('logs')->cursor() as $log) {
    echo $log['message'];
}
```

---

## ORM 模型 (Model)

Anon Framework Next 提供了基于 ActiveRecord 模式的 ORM 模型。

定义模型类，继承 `Anon\Core\Database\Model`：

```php
namespace Anon\Model;

use Anon\Core\Database\Model;

class User extends Model
{
    protected string $table = 'users';
    protected string $primaryKey = 'id';
}
```

如果主键不是默认 `id`，或者目标数据库依赖 sequence，可以直接在模型里声明：

```php
class Invoice extends Model
{
    protected string $table = 'invoices';
    protected string $primaryKey = 'invoice_id';
    protected ?string $sequence = 'invoices_invoice_id_seq';
}
```

使用模型进行数据操作：

```php
use Anon\Model\User;

// 查询所有记录
$users = User::all();

// 根据主键查询单条记录
$user = User::find(1);

// 结合查询构建器进行条件查询
$user = User::where('status', 1)->first();

// 创建新数据
$user = User::create(['name' => 'Anon', 'email' => 'anon@example.com']);

// 更新数据
$user = User::find(1);
if ($user) {
    $user->name = 'New Name';
    $user->save();
}

// 批量删除数据
User::destroy(1);
User::destroy([1, 2, 3]);
```

## 模型事件

ORM 支持在模型的生命周期中触发事件，这对于解耦业务逻辑非常有用。例如在创建用户后自动初始化配置，或在更新时记录日志。

支持的事件列表：
- `creating` / `created`
- `updating` / `updated`
- `saving` / `saved`
- `deleting` / `deleted`

在任意前置事件（如 `creating`）中返回 `false` 将会中断随后的保存或删除操作。

```php
namespace Anon\Model;

use Anon\Core\Database\Model;

class User extends Model
{
    protected static function boot()
    {
        // 注册事件
        static::creating(function (User $user) {
            // 在插入数据库前执行
            if (empty($user->uuid)) {
                $user->uuid = \Anon\Core\Support\Str::uuid();
            }
        });
        
        static::deleted(function (User $user) {
            // 用户删除后，级联删除他的帖子
            Post::query()->where('user_id', $user->id)->delete();
        });
    }
}
```

> **注意：** 框架启动时需要在合适的生命周期（如服务提供者或全局 Hook `app_init` 中）主动调用一次模型的 `boot` 方法以注册闭包。或者你也可以在项目自己的模型基类里，例如 `Anon\Model\BaseModel` 的构造函数中做静态检测。

## 关系模型

当前 ORM 已支持常见的三种基础关系：

- `hasOne`
- `hasMany`
- `belongsTo`

### 一对一

```php
namespace Anon\Model;

use Anon\Core\Database\Model;

class User extends Model
{
    public function profile()
    {
        return $this->hasOne(Profile::class, 'user_id', 'id');
    }
}
```

### 一对多

```php
class User extends Model
{
    public function posts()
    {
        return $this->hasMany(Post::class, 'user_id', 'id');
    }
}
```

### 反向关联

```php
class Post extends Model
{
    public function user()
    {
        return $this->belongsTo(User::class, 'user_id', 'id');
    }
}
```

## 预加载

为了避免列表接口中频繁触发 N+1 查询，可以在查询时使用 `with()` 进行预加载：

```php
use Anon\Model\User;

$users = User::query()->with(['profile', 'posts'])->get();

foreach ($users as $user) {
    echo $user->profile?->nickname;
    echo count($user->posts);
}
```

单条查询同样支持：

```php
$user = User::query()->with('profile')->where('id', 1)->first();
```

## 当前能力边界

如果你准备在生产里使用多数据库，建议先按下面理解当前边界：

- 连接层：已支持多 PDO 驱动
- Query Builder：基础 CRUD、条件、Join、排序、分组可用
- ORM：基础 ActiveRecord、关系、预加载、软删除可用
- 迁移：当前是原生 SQL 方案，不是 Laravel 风格的跨库字段抽象
- 跨库一致性：分页、正则、主键回填、迁移 SQL 仍建议实库验证

## 软删除

在模型中开启软删除后，普通查询会自动排除已删除数据，`delete()` 会改为写入删除时间：

```php
class Post extends Model
{
    protected bool $softDelete = true;
}
```

### 基础用法

```php
$post = Post::find(1);
$post?->delete(); // 写入 deleted_at

$all = Post::withTrashed()->get();     // 包含已删除
$trashed = Post::onlyTrashed()->get(); // 仅已删除

$post?->restore();     // 恢复
$post?->forceDelete(); // 物理删除
```
