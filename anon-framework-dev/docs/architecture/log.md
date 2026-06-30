# 日志系统 (Log)

Anon Framework Next 提供了基于文件系统的日志组件，能够按天切割日志文件，并将不同类型的日志进行隔离。

默认情况下，所有的日志文件都会被写入到 `runtime/log` 目录下。

## 基础使用

通过 `Anon\Core\Facade\Log` 门面记录不同级别的日志。该方法接受字符串或数组，数组会自动转换为 JSON。

```php
use Anon\Core\Facade\Log;

// 记录普通信息
Log::info('This is an info message.');

// 记录错误
Log::error(['error_code' => 500, 'message' => 'Something went wrong']);

// 记录调试信息（仅在 DEBUG_MODE=true 或 app.debug=true 时才会真正写入日志文件）
Log::debug('Debugging connection...');

// 记录警告
Log::warning('Disk space is running low.');
```

---

## 内存缓冲日志 (Buffered I/O)

框架底层实现了日志缓冲池。当调用 `Log` 方法时，日志会先追加到内存中。
只有在以下两种情况才会批量写入磁盘：
1. 请求生命周期结束时（通过 `register_shutdown_function` 触发）。
2. 单个请求产生的同类日志条数达到 1000 条。

这有效减少了频繁的磁盘 I/O 操作。

---

## 自定义日志通道 (分类)

日志方法接收可选的第二个参数 `$type`，表示日志的分类通道（默认是 `'app'`）。
当传入分类时，日志将被写入独立文件，例如 `access.log` 或 `sql.log`。

```php
// 写入 runtime/log/YYYY-MM-DD/access.log
Log::info('User login successful', 'access');

// 写入 runtime/log/YYYY-MM-DD/sql.log
Log::debug('SELECT * FROM users', 'sql');
```

## 调试模式专属日志 (Debug Mode)

为了在开发阶段提供充足的上下文，又不在生产环境产生大量的日志垃圾占用磁盘，框架专门为 `Log::debug()` 做了调试模式级别拦截：

1. **依赖调试配置**：只有在 `.env*` 中设置 `DEBUG_MODE=true`，或在 `anon.config.php` 中设置 `app.debug=true` 时，`Log::debug()` 才会落盘。
2. **生产环境静默**：当调试模式关闭时，所有调用 `Log::debug()` 的代码会直接返回，没有额外 I/O 开销。

**动态调整 Debug 状态**

你可以随时通过 `setDebug()` 方法动态更改调试记录状态（适用于某些需要临时开启高频日志的脚本）：

```php
use Anon\Core\Facade\Log;

// 检查当前是否开启了 debug 模式
$isDebug = Log::isDebug();

// 临时在生产环境为某个复杂任务开启 debug 日志记录
Log::setDebug(true);
Log::debug('This will be logged regardless of DEBUG_MODE value');
Log::setDebug($isDebug); // 恢复原有状态
```

## 异常接管

框架底层实现了全局异常处理器 `Anon\Core\Exception\Handler`。当发生 500 及以上级别的致命错误或未捕获异常时，系统会自动拦截并记录到 `exception.log`。
