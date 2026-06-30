# 数据库迁移与填充 (Migration & Seeder)

数据库迁移就像是数据库的版本控制，它让团队能够轻松修改和共享应用程序的数据库结构。搭配数据填充（Seeder），你可以用代码管理所有的建表语句和初始数据。

---

## 数据库迁移 (Migration)

### 创建迁移

使用内置的 CLI 工具快速生成一个迁移文件：

```bash
php anon make:migration CreateUsersTable
```

这会在 `app/database/migrations/` 目录下生成一个带时间戳前缀的迁移文件。

### 编写迁移

在生成的迁移文件中，你可以在 `up` 方法中编写执行的 SQL，在 `down` 方法中编写回滚的 SQL。

```php
namespace App\Database\Migrations;

use Anon\Core\Database\Migration\Migration;

class CreateUsersTable extends Migration
{
    public function up(): void
    {
        $this->statement("
            CREATE TABLE IF NOT EXISTS users (
                id INT AUTO_INCREMENT PRIMARY KEY,
                username VARCHAR(50) NOT NULL,
                email VARCHAR(100) NOT NULL UNIQUE,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        ");
    }

    public function down(): void
    {
        $this->statement("DROP TABLE IF EXISTS users");
    }
}
```

### 运行迁移

编写完毕后，执行以下命令即可运行所有尚未执行的迁移：

```bash
php anon migrate
```

框架会自动在数据库中创建一张 `migrations` 表来记录已执行的迁移批次，防止重复执行。

---

## 数据填充 (Seeder)

### 创建 Seeder

同样使用 CLI 工具生成：

```bash
php anon make:seeder UserSeeder
```

文件将生成在 `app/database/seeders/UserSeeder.php`。

### 编写 Seeder

```php
namespace App\Database\Seeders;

use Anon\Core\Database\Migration\Seeder;
use Anon\Core\Facade\DB;

class UserSeeder extends Seeder
{
    public function run(): void
    {
        DB::table('users')->insert([
            'username' => 'admin',
            'email' => 'admin@example.com'
        ]);
        
        DB::table('users')->insert([
            'username' => 'test',
            'email' => 'test@example.com'
        ]);
    }
}
```

### 运行填充

执行以下命令即可将初始数据写入数据库：

```bash
# 默认会执行 DatabaseSeeder (你可以创建一个 DatabaseSeeder 来调用其他 Seeder)
php anon db:seed

# 或者指定执行某一个特定的 Seeder
php anon db:seed --class=UserSeeder
```
