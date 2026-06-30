# 异步任务队列 (Queue)

在构建 API 时，部分操作（如发送注册邮件、生成报表、调用慢速第三方接口）会显著拖慢响应时间。Anon Framework Next 提供了基于 Redis 的轻量级异步任务队列，让你能够将耗时任务放入后台处理。

---

## 配置文件与依赖

队列功能依赖于 `Redis`。推荐在 `anon.config.php` 中只声明队列自身配置，Redis 连接则复用 `cache.redis` 或继续放在 `.env*` 中：

```php
use Anon\Core\Facade\Env;

return [
    'queue' => [
        'default' => 'default',
        'prefix' => Env::get('QUEUE_PREFIX', 'anon:queue:'),
        'max_tries' => 3,
    ],
];
```

当你没有单独声明 `queue.redis` 时，框架会自动按以下顺序寻找 Redis 连接：

- `queue.redis`
- `cache.redis`
- 旧的顶层 `redis`
- `.env*`

兼容模式下，仍可继续通过 `.env` 配置：

```env
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DB=0
QUEUE_PREFIX=anon:queue:
```

---

## 创建任务 (Job)

你可以使用 CLI 工具快速生成一个任务类：

```bash
php anon make:job SendWelcomeEmail
```

这会在 `app/jobs/SendWelcomeEmail.php` 下生成文件。

### 编写任务逻辑

任务类必须实现 `Anon\Core\Queue\Job` 接口。你可以将需要的数据通过构造函数传入，并在 `handle` 方法中编写实际执行的耗时逻辑。

```php
namespace App\Jobs;

use Anon\Core\Queue\Job;
use Anon\Core\Facade\Log;

class SendWelcomeEmail implements Job
{
    protected array $user;

    public function __construct(array $user)
    {
        $this->user = $user;
    }

    public function handle(): void
    {
        // 模拟耗时的发邮件操作
        sleep(2);
        
        Log::info("Welcome email sent to: " . $this->user['email']);
    }
}
```

---

## 推送任务到队列

在你的控制器或业务逻辑中，通过 `Queue` 门面将任务对象推送至后台。

```php
namespace Anon\Controller;

use Anon\Core\Http\Response;
use Anon\Core\Facade\Queue;
use Anon\Jobs\SendWelcomeEmail;

class AuthController
{
    public function register(): Response
    {
        // ... 注册逻辑，将用户存入数据库
        $user = ['email' => 'newuser@example.com'];

        // 将发邮件任务推送到默认队列
        Queue::push(new SendWelcomeEmail($user));

        // 延迟 10 秒执行，并限制最多重试 5 次
        Queue::push(new SendWelcomeEmail($user), 'emails', 10, 5);

        return Response::json(['msg' => 'Registration successful!']);
    }
}
```

此时请求会立即返回，而任务已经序列化并持久化在 Redis 队列中。

## 重试、延迟与失败队列

当前队列已支持以下能力：

- 延迟任务：`push($job, $queue, $delay)`
- 最大重试次数：`push($job, $queue, $delay, $maxTries)`
- 失败队列：超过最大重试次数后自动写入 `:failed` 队列
- 延迟重试：worker 处理异常后会按 `--backoff` 秒数重新入队

例如：

```php
// 推送到 emails 队列，延迟 30 秒执行，最多尝试 5 次
Queue::push(new SendWelcomeEmail($user), 'emails', 30, 5);
```

---

## 运行队列处理器 (Worker)

为了让推送到队列的任务得以执行，你需要启动队列处理器守护进程：

```bash
php anon queue:work
```

你也可以指定要监听的队列名称：

```bash
php anon queue:work --queue=emails
```

你也可以指定失败后的重试退避秒数：

```bash
php anon queue:work --queue=emails --backoff=5
```

> **注意：** 在生产环境中，建议使用 `Supervisor` 或 `Systemd` 来管理 `queue:work` 进程，确保它在意外退出时能自动重启。

## 失败任务查看、重试与清理

当任务超过最大重试次数后，会自动进入 `:failed` 队列。你可以通过以下命令查看、重新投递或清理失败任务：

```bash
# 查看默认队列最近 20 条失败任务
php anon queue:failed

# 查看指定队列
php anon queue:failed --queue=emails --limit=50

# 重试单个失败任务
php anon queue:retry --queue=emails --id=job-id

# 重试指定队列的全部失败任务
php anon queue:retry --queue=emails --all

# 清空指定队列的失败任务
php anon queue:clear-failed --queue=emails
```

`queue:retry` 支持 `--delay=<seconds>`，用于把重试任务重新投递到延迟队列中。

手动重试时，框架会将任务重新放回待消费队列，并重置本轮尝试计数，便于重新执行完整的重试流程。

`queue:clear-failed` 会直接删除当前失败队列中的全部记录，适合在你确认这些失败任务已经不再需要保留时执行。
