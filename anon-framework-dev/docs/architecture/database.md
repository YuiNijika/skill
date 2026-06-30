# 数据库与查询构建器

Anon Framework Next 内置了基于 PDO 的数据库查询构建器和 ActiveRecord 风格的 ORM，支持 `MySQL`, `PostgreSQL`, `SQLite`, `SQLServer`, `Oracle` 等数据库。

## 配置

你可以在项目根目录下通过 `anon.config.php` 配置数据库连接，并使用 `.env*` 保存账号密码等敏感值。配置文件示例更推荐使用 `Env::get()`，这样能兼容更完整的环境变量读取链：

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

可以通过 `Anon\Core\Facade\DB` 直接进行数据库操作。

### 执行原生 SQL

```php
use Anon\Core\Facade\DB;

// 执行查询
$users = DB::select('SELECT * FROM users WHERE status = ?', [1]);

// 执行写入/更新/删除等操作
$affected = DB::statement('UPDATE users SET status = ? WHERE id = ?', [0, 1]);
```

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

// NULL 查询
DB::table('users')->whereNotNull('email')->get();
```

#### 排序与分页

在使用 Query Builder 进行分页和条数限制时，Anon Framework 支持多种灵活的调用方式。你可以通过独立的 `limit()` 和 `offset()` 方法链式调用，也可以直接在 `limit()` 方法中传递两个参数（推荐，更符合原生 SQL 直觉）：

```php
// 方式一：独立使用 limit() 和 offset()
$users = DB::table('users')
    ->orderBy('created_at', 'DESC')
    ->limit(10)
    ->offset(20)
    ->get();

// 方式二：直接在 limit() 中传入 limit 和 offset (推荐)
// 第一个参数为 limit (查询条数)，第二个参数为 offset (偏移量)
$users = DB::table('users')
    ->orderBy('created_at', 'DESC')
    ->limit(10, 20)
    ->get();

// 方式三：使用高层级的 paginate 方法，自动获取 $_GET['page'] 进行分页
// 并在返回结果中自动包装 total、per_page、current_page 等元数据
$result = DB::table('users')->paginate(15);
```

#### 连表查询 (Join)

```php
$users = DB::table('users')
    ->select(['users.id', 'users.name', 'roles.role_name'])
    ->leftJoin('roles', 'users.role_id', '=', 'roles.id')
    ->get();
```

#### 插入数据

```php
// 插入单条数据
$insertId = DB::table('users')->insert([
    'name' => 'Anon',
    'email' => 'anon@example.com'
]);
```

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
