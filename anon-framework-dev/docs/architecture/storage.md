# 文件存储 (Storage)

Anon Framework Next 底层基于 `league/flysystem`，提供了统一的文件存储抽象。

默认情况下，框架使用 `local` 驱动，文件存储在项目的 `runtime/storage` 目录下。

## 配置

推荐在 `anon.config.php` 中定义默认磁盘与本地磁盘路径：

```php
use Anon\Core\Facade\Env;

return [
    'storage' => [
        'default' => 'local',
        'disks' => [
            'local' => [
                'root' => __DIR__ . '/runtime/storage',
                'url' => rtrim((string) Env::get('APP_URL', 'http://127.0.0.1:8000'), '/') . '/storage',
            ],
        ],
    ],
];
```

如果配置文件里要读取环境变量，建议和其他模块一样优先使用 `Env::get()`，不要在业务代码里散落直接的 `getenv()`。

## 基础用法

通过 `Anon\Core\Facade\Storage` 门面进行文件操作：

```php
use Anon\Core\Facade\Storage;

// 写入文件
Storage::put('avatars/1.jpg', $content);

// 追加内容
Storage::append('logs/user.log', 'User logged in.');

// 读取文件
$content = Storage::get('avatars/1.jpg');

// 判断文件是否存在
if (Storage::exists('avatars/1.jpg')) {
    // ...
}

// 删除文件
Storage::delete('avatars/1.jpg');

// 复制与移动文件
Storage::copy('avatars/1.jpg', 'avatars/1_backup.jpg');
Storage::move('avatars/1_backup.jpg', 'archive/1.jpg');

// 获取文件 URL
$url = Storage::url('avatars/1.jpg');
```
