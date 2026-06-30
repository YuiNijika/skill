# 部署检查清单

这篇文档专门用于上线前和线上排障。

它不讲框架怎么开发，而是讲：**Anon 项目上线后最容易出问题的几个点到底该怎么检查。**

如果你遇到这些情况，优先看这篇：

- 本地正常，线上异常
- 明明改了配置却不生效
- 控制器或路由在线上找不到
- 改了 `framework/core` 但项目运行结果没变化
- 明明代码已经上传，线上仍像旧版本

---

## 1. 发布后第一件事

代码同步到服务器后，建议至少执行这些动作：

```bash
composer dump-autoload
php anon config:clear
php anon route:clear
```

如果你的发布流程本来就会构建缓存，再继续执行：

```bash
php anon config:cache
php anon route:cache
```

如果项目启用了 `php-fpm` 或 opcache，还要记得刷新运行时：

- 重启 `php-fpm`
- 或重新加载 PHP 服务
- 或确认 opcache 已经失效

---

## 2. Linux 大小写检查

这是 Anon 项目最容易出现“本地正常，线上 500”的原因之一。

### 当前推荐目录

- `app/Controller`
- `app/Service`
- `app/Route`
- `app/Action`
- `app/Model`
- `app/Middleware`
- `app/Provider`

### 常见错误

本地 Windows 下这些通常都能跑：

- `app/controller`
- `app/route`
- `app/action`

但线上 Linux 下会出问题，因为命名空间和目录大小写必须严格一致。

### 重点检查

如果报错类似：

- `Controller class ... not found`
- `Class ... not found`

优先检查：

1. 文件夹大小写
2. 命名空间大小写
3. Composer autoload 是否已刷新

---

## 3. 配置不生效时怎么查

如果你改了：

- `anon.config.php`
- `.env`
- `.env.local`
- `.env.production`

但线上结果完全没变化，先不要怀疑业务代码，先查配置缓存。

### 配置缓存文件

```text
runtime/cache/config.php
```

### 处理顺序

1. 删除或清理配置缓存
2. 再重新请求
3. 还不生效再去看代码逻辑

常用命令：

```bash
php anon config:clear
```

### 补充说明

配置被缓存后，当前运行时会直接读取缓存文件，而不是重新解析 `.env*` 和 `anon.config.php`。

所以你看到“文件改了但运行结果没变”，很多时候根因不是代码，而是缓存。

---

## 4. 路由不生效时怎么查

如果你改了：

- `app/Route/*.php`
- 路由中间件
- 控制器方法名

但线上仍然走旧路由，先查路由缓存。

### 路由缓存文件

```text
runtime/cache/routes.php
```

### 常用命令

```bash
php anon route:clear
php anon route:cache
```

### 特别注意

如果项目正在使用路由缓存：

- 新增路由文件不会自动生效
- 修改控制器方法映射也不会自动生效

必须先清旧缓存，再重新构建。

---

## 5. Env 读取问题

如果某个第三方 API Key、本地路径、运行开关本地正常，线上却像没读取到，优先检查环境变量读取方式。

### 推荐写法

```php
use Anon\Core\Facade\Env;

$apiKey = trim((string) Env::get('THIRD_PARTY_API_KEY', ''));
```

### 不推荐写法

```php
$apiKey = getenv('THIRD_PARTY_API_KEY');
```

### 为什么

线上环境可能出现：

- `putenv()` 被禁用
- 变量只存在于 `$_ENV`
- 变量只存在于 `$_SERVER`
- 系统环境变量覆盖了 `.env*`

框架已经把这些读取兼容收敛到了 `Env::get()`。

---

## 6. framework/core 和 runtime vendor 的区别

很多人会改错地方。

### 两种常见位置

- 框架源码：`framework/core`
- 项目实际运行副本：`project/.../vendor/yuinijika/anon-core`

### 关键点

你改了 `framework/core`，不代表当前项目就会立刻使用这份代码。

如果项目真正执行的是 vendor 里的 `anon-core`，那你要么：

- 重新发布依赖
- 要么同步修改当前运行时的 vendor 副本

### 排查顺序

1. 先确认线上项目实际加载的是哪一份代码
2. 再决定改 app、改 framework/core，还是同步改 vendor

---

## 7. Response 格式检查

如果线上接口结构不对，优先检查是否有旧代码或旧文档思路混进来了。

### 当前正确格式

成功：

```json
{
  "success": true,
  "code": 200,
  "message": "OK",
  "data": {}
}
```

失败：

```json
{
  "success": false,
  "code": 400,
  "message": "Bad Request",
  "error_code": "BAD_REQUEST"
}
```

### 常见回退错误

- `code` 返回 `"OK"`
- 错误码直接覆盖 `code`
- 控制器返回结构不统一

---

## 8. 第三方接口排查

如果对接第三方接口时线上异常，本地正常，优先检查这些：

1. API Key 是否真的读到了
2. `Env::get()` 是否替代了裸 `getenv()`
3. `http.ssl_verify` 是否被配置缓存固定住
4. 线上 CA / SSL 问题是否被误判成业务错误
5. 上游限流头是否需要透传

如果改了 `anon.config.php` 里的：

```php
'http' => [
    'ssl_verify' => false,
],
```

但效果没变，第一反应先去看 `runtime/cache/config.php`。

---

## 9. 本地正常线上异常的默认排查顺序

强烈建议按这个顺序排：

1. 目录大小写
2. `composer dump-autoload`
3. `runtime/cache/config.php`
4. `runtime/cache/routes.php`
5. `Env::get()` / 环境变量读取链
6. `php-fpm` / opcache
7. 当前实际运行的是不是 vendor 副本

这是目前最省时间的一套默认顺序。

---

## 10. 发布交付时建议附带的提醒

如果你在给别人交付 Anon 项目修改，建议把这些也一起说清楚：

- 是否需要执行 `composer dump-autoload`
- 是否需要清配置缓存
- 是否需要清路由缓存
- 是否需要重建缓存
- 是否需要重启 `php-fpm`
- 是否涉及 Linux 目录大小写
- 是否改的是 `framework/core` 还是实际运行中的 `vendor/anon-core`

---

## 相关文档

- [当前约定总览](./current-conventions.md)
- [路由系统](./router.md)
- [请求与响应](./request-response.md)
- [项目配置](./configuration.md)
- [环境变量](./env.md)
- [身份认证](./auth.md)
- [命令行控制台](./console.md)

如果你是准备上线或者正在查线上问题，这篇应该优先看。
