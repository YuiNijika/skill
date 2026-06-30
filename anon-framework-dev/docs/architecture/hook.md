# 钩子系统 (Hook)

在 Anon Framework Next 中，**Hook（钩子）**本质上是事件系统（Event）在框架生命周期层面的语义化封装。钩子系统允许你在不修改框架核心代码的情况下，通过挂载行为（Behavior）来介入框架的运行流程。

## 基础使用

你可以通过 `Anon\Core\Facade\Hook` 门面类来注册和触发钩子。

### 注册钩子 (Add)

注册钩子的语法与注册事件监听器完全一致，支持闭包、类方法字符串以及数组形式：

```php
use Anon\Core\Facade\Hook;

// 1. 使用闭包
Hook::add('app_init', function ($app) {
    // 应用初始化逻辑
});

// 2. 使用 "类名@方法名" 字符串 (方法默认名为 handle 时可省略)
Hook::add('request_begin', 'App\Behaviors\RequestCheck@handle');

// 3. 使用数组形式
Hook::add('response_send', [\App\Behaviors\CorsBehavior::class, 'handle']);
```

> **注意：** 当使用类作为行为（Behavior）时，系统会自动通过依赖注入容器 (`Container`) 实例化该类。

### 触发钩子 (Trigger)

在开发扩展包或者自定义模块时，你也可以在自己的代码中主动埋入钩子，供其他开发者接入：

```php
// 触发钩子，并将关键数据作为载荷传递给各个挂载的行为
$results = Hook::trigger('my_custom_hook', ['param' => 'value']);
```

### 拦截执行

如果某个钩子行为在执行时返回了严格的 `false`，则会立即中断后续所有行为的执行（这点与事件系统的拦截机制一致）：

```php
Hook::add('request_begin', function ($request) {
    if ($request->method() === 'OPTIONS') {
        // 处理完毕，终止后续流程
        return false; 
    }
});
```

## 默认生命周期钩子

框架在启动和运行的过程中，已经在核心架构的关键节点埋入了丰富的系统钩子，供开发者自定义扩展。

### 全局生命周期 (App)

| 钩子名称 | 触发时机 | 传递载荷 | 用途示例 |
| --- | --- | --- | --- |
| `app_init` | 应用容器初始化完毕，且环境变量 `.env` 加载完成后 | `$app` (App实例) | 初始化全局设置、修改动态配置 |
| `route_loaded` | 路由文件加载完毕后 | 无 | 动态追加路由、调整路由组中间件 |
| `request_begin` | 捕获到 Request 对象，即将进行路由分发前 | `$request` (Request实例) | 跨域预检处理、全局请求日志记录 |
| `response_send` | 响应数据准备就绪，即将发送给客户端前 | `$response` (Response实例) | 统一注入跨域 Header、追加响应签名 |
| `app_end` | 响应数据已经输出并结束连接后 | `$app` (App实例) | 执行不阻碍响应的后台清理、数据统计任务 |
| `exception_render`| 捕获到异常即将输出响应时 | `['exception', 'statusCode', 'message', 'data']` | 覆盖默认的异常响应，若返回 `Response` 实例则直接发送 |

### 数据库查询 (Database)

| 钩子名称 | 触发时机 | 传递载荷 | 用途示例 |
| --- | --- | --- | --- |
| `db_query_begin` | 原生 SQL 查询执行前 | `['sql', 'bindings']` | SQL 注入防火墙过滤、打印预执行 SQL |
| `db_query_end` | 原生 SQL 查询执行完毕后 | `['sql', 'bindings', 'time', 'result'/'affected']` | 慢查询日志记录、性能监控 |

### 认证授权 (Auth)

| 钩子名称 | 触发时机 | 传递载荷 | 用途示例 |
| --- | --- | --- | --- |
| `auth_login` | 成功签发登录 Token 及 Session 时 | `['user', 'guard', 'subject', 'session_id', 'tokens']` | 记录登录日志、同步到第三方系统 |
| `auth_logout` | 用户注销且清除了 Session 时 | `['guard', 'payload']` | 记录登出日志 |

### 异步队列 (Queue)

| 钩子名称 | 触发时机 | 传递载荷 | 用途示例 |
| --- | --- | --- | --- |
| `queue_push` | 任务被成功推送到 Redis 队列时 | `['job', 'queue', 'delay', 'payload']` | 统计队列积压数据 |
| `queue_job_process` | 守护进程从队列中取出任务准备执行前 | `['job', 'payload', 'attempt']` | 初始化任务运行环境（如重置数据库连接） |
| `queue_job_success` | 守护进程成功执行完任务后 | `['job', 'payload', 'time']` | 记录任务执行耗时 |
| `queue_job_failed` | 守护进程执行任务遭遇异常时 | `['job', 'payload', 'exception']` | 发送报警邮件、通知开发者 |

## 注册钩子行为的推荐位置

为了方便管理，框架在项目骨架中提供了一个专门的配置文件：`app/hook.php`。

应用在启动时会自动加载这个文件，你可以将所有的系统钩子注册逻辑统一写在这里：

```php
// app/hook.php 示例

use Anon\Core\Facade\Hook;
use Anon\Core\Http\Response;

Hook::add('response_send', function (Response $response) {
    // 统一为所有响应添加自定义标识头
    $response->setHeader('X-Powered-By', 'Anon Framework Next');
});
```