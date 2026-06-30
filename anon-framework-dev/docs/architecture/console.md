# 命令行控制台 (Console)

Anon Framework Next 提供了一个独立的命令行工具 `anon`，它类似于 Laravel 的 Artisan 或 ThinkPHP 的 Console，旨在帮助你快速执行后台任务、启动服务或生成代码文件。

---

## 启动控制台

在项目骨架的根目录中，存在一个名为 `anon` 的可执行文件。你可以通过 PHP 直接运行它：

```bash
php anon
```

如果不带任何参数，它会列出当前应用支持的所有命令及帮助信息：

```bash
$ php anon
Anon Framework Next Console Tool 1.0.0

Usage:
  command [options] [arguments]

Available commands:
  config:cache         Build and cache the project configuration
  config:clear         Remove the cached project configuration
  dev                  Start the built-in PHP development server
  make:controller      Create a new controller class
  make:model           Create a new model class
  make:middleware      Create a new middleware class
  queue:clear-failed   Clear all failed jobs from the queue
  queue:failed         List failed jobs from the queue
  queue:retry          Retry one or all failed jobs from the queue
  route:cache          Build and cache the application routes
  route:clear          Remove the cached application routes
  run                  Start the built-in PHP server in production mode
```

---

## 内置命令

### dev (开发模式)
启动 PHP 内置的高性能开发服务器，它会自动将文档根目录指向 `run/`，便于你进行本地 API 开发和调试。
此命令会自动向环境中注入 `DEBUG_MODE=true`、`APP_DEBUG=true` 和 `APP_ENV=local`。

```bash
php anon dev
```
默认监听 `127.0.0.1:8000`。你可以通过选项覆盖：
```bash
php anon dev --host=0.0.0.0 --port=8080
```

### run (生产模式)
启动 PHP 内置的服务器用于普通运行或生产环境，与 `dev` 命令类似，但它会自动向环境中注入 `DEBUG_MODE=false`、`APP_DEBUG=false` 和 `APP_ENV=production` 以关闭调试信息并提升性能。

```bash
php anon run
```
使用方法和选项与 `dev` 命令完全一致：
```bash
php anon run --host=0.0.0.0 --port=8080
```

### version
你可以通过该命令快速查看当前 Anon Framework Next 的框架版本号，它会动态调用 `\Anon\Core\Facade\App::version()` 获取实际版本。

```bash
php anon version
# 或使用缩写
php anon -v
php anon --version
```

### config:cache / config:clear
用于生成和清理配置缓存文件：

```bash
php anon config:cache
php anon config:clear
```

执行 `config:cache` 后，框架会把最终合并后的配置写入 `runtime/cache/config.php`，后续启动会优先读取该文件。

### route:cache / route:clear
用于生成和清理路由缓存文件：

```bash
php anon route:cache
php anon route:clear
```

执行 `route:cache` 后，框架会把路由定义导出到 `runtime/cache/routes.php`。当前仅支持控制器路由缓存，闭包路由会在构建时直接报错，避免不安全的序列化行为。

### queue:failed / queue:retry / queue:clear-failed
用于查看失败任务、重新投递失败队列中的任务，以及清空失败任务：

```bash
php anon queue:failed --queue=default --limit=20
php anon queue:retry --queue=default --id=job-id
php anon queue:retry --queue=default --all
php anon queue:clear-failed --queue=default
```

如果你希望重试任务延迟重新执行，可以额外传入 `--delay=5` 这类参数。

### 推荐发布流程
如果你准备在生产环境发布一个已经完成代码更新的项目，推荐按以下顺序执行：

```bash
composer install --no-dev --optimize-autoloader
php anon config:clear
php anon route:clear
php anon config:cache
php anon route:cache
```

其中：

- `config:cache` 适合在配置稳定后执行
- `route:cache` 适合在路由全部使用控制器动作后执行
- 如果发布平台本身会注入环境变量，应先确保这些变量已就位，再生成缓存

### 生成器命令
框架提供了一系列 `make:*` 命令来快速生成基础代码文件：

```bash
# 生成单文件控制器
php anon make:controller User

# 会生成
# app/Controller/User.php

# 生成目录式控制器，默认生成 Index.php
php anon make:controller User --group
php anon make:controller Admin/User --group

# 会生成
# app/Controller/User/Index.php
# app/Controller/Admin/User/Index.php

# 生成资源控制器模板
php anon make:controller User --resource
php anon make:controller Admin/User --group --resource

# 会生成 index/show/store/update/delete 五个常用动作
# 适合直接配合 Route::resource() 使用

# 生成模型
php anon make:model User
php anon make:model Admin/User

# 会生成
# app/Model/User.php
# app/Model/Admin/User.php

# 生成中间件
php anon make:middleware AuthMiddleware
php anon make:middleware Admin/CheckLogin

# 会生成
# app/Middleware/AuthMiddleware.php
# app/Middleware/Admin/CheckLogin.php

# 生成 Server Action
php anon make:action PublishPost
php anon make:action Admin/RetryJob

# 会生成
# app/Action/PublishPost.php
# app/Action/Admin/RetryJob.php

# 生成 FormRequest
php anon make:request PublishPostRequest

# 会生成
# app/Http/Requests/PublishPostRequest.php

# 生成 API Resource
php anon make:resource PostResource

# 会生成
# app/Http/Resources/PostResource.php

# 生成服务提供者
php anon make:provider AppServiceProvider

# 会生成
# app/Provider/AppServiceProvider.php
```

推荐统一使用大写目录名，例如 `Controller`、`Model`、`Middleware`、`Action`、`Provider`。Windows 本地通常不会暴露这个问题，但 Linux 线上环境会严格区分大小写，生成器结果也应按最终线上可用状态来理解。

### Server Actions 列表

注册的 Server Actions 可以直接列出来，调试时很省事：

```bash
php anon action:list
```

如果要给脚本或后续客户端生成器使用，可以输出 JSON：

```bash
php anon action:list --json
```

输出内容来自当前应用真实加载后的 Action 注册表，所以执行前要确保路由文件会被正常加载。

---

## 编写自定义命令

随着业务发展，你可能需要编写自己的命令（如定时任务、数据迁移等）。

1. 继承基础的 `Anon\Core\Console\Command` 类。
2. 设置 `$name` 和 `$description` 属性。
3. 实现 `execute(array $args): int` 方法。

```php
<?php

namespace Anon\Command;

use Anon\Core\Console\Command;

class HelloCommand extends Command
{
    protected string $name = 'hello';
    protected string $description = 'Say hello to the framework';

    public function execute(array $args): int
    {
        // $this->info() 会输出带绿色的提示文字
        $this->info("Hello, Anon Framework Next!");
        
        // 也可以使用 $this->getOption() 获取命令行参数
        $name = $this->getOption($args, 'name', 'Guest');
        echo "Nice to meet you, {$name}\n";

        return 0; // 返回 0 代表执行成功
    }
}
```

随后，你需要将该命令注册到 `Console\Application` 中即可生效（未来我们可能会支持自动扫描加载）。
