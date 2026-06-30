# 当前约定总览

这篇文档不是重复介绍每个模块怎么用，而是把 **Anon Framework Next 当前真正生效的开发约定** 收拢到一处。

如果你只想先知道“现在到底应该怎么写”，先看这篇，再按需展开看其他细分文档。

---

## 1. 目录与命名空间

当前推荐统一使用大写目录名，并让目录名与命名空间保持一致：

- `app/Controller`
- `app/Service`
- `app/Route`
- `app/Action`
- `app/Model`
- `app/Middleware`
- `app/Provider`

对应命名空间也应保持一致，例如：

```php
namespace Anon\Controller;
namespace Anon\Service;
namespace Anon\Route;
namespace Anon\Action;
```

### 为什么必须这样

Windows 本地文件系统通常对大小写不敏感，所以你把目录写成 `app/controller` 本地也许还能跑。

但 Linux 线上环境是大小写敏感的：

- `Anon\Controller\UserController` 对应的就是 `app/Controller/UserController.php`
- 不是 `app/controller/UserController.php`

所以这不是代码风格问题，而是生产兼容性问题。

---

## 2. 响应格式

当前统一响应格式以 REST API 为准。

### 成功响应

```json
{
  "success": true,
  "code": 200,
  "message": "OK",
  "data": {}
}
```

### 失败响应

```json
{
  "success": false,
  "code": 400,
  "message": "Bad Request",
  "error_code": "BAD_REQUEST"
}
```

### 当前约定

- `code` 必须是数字型 HTTP 状态码
- 业务错误码放进 `error_code`
- 成功时如果需要业务语义，可额外使用 `business_code`
- 不再使用旧示例里的 `"code": "OK"`

推荐统一使用：

```php
return Response::success($data, 'OK', 200);
```

```php
return Response::error('缺少参数', 400, null, 'BAD_REQUEST');
```

---

## 3. Request 读取方式

请以当前真实实现为准，不要凭印象调用不存在的方法。

当前常用读取方式：

- 查询参数：`$request->input('type', null)`
- 路由参数：`$request->route('id')`
- 请求头：`$request->header('Authorization', '')`
- Cookie：`$request->cookie('access_token')`
- 文件：`$request->file('file')`

不要使用当前运行时并不存在的方法，例如：

```php
$request->query('type');
```

如果某段旧代码或旧文档里还在用 `query()`，按现在的实现应改成 `input()`。

---

## 4. Env 与 Config

当前推荐分工：

- `anon.config.php`：放结构化配置
- `.env*`：放敏感值和环境差异值

### 读取规则

业务代码中：

- 配置统一走 `Config::get()`
- 环境变量统一走 `Env::get()`

示例：

```php
use Anon\Core\Facade\Config;
use Anon\Core\Facade\Env;

$sslVerify = (bool) Config::get('http.ssl_verify', true);
$apiKey = trim((string) Env::get('THIRD_PARTY_API_KEY', ''));
```

### 为什么不推荐直接 `getenv()`

因为线上环境可能会遇到：

- `putenv()` 被禁用
- 变量只存在于 `$_ENV`
- 变量只存在于 `$_SERVER`

框架已经把环境变量读取链做到了 `Env::get()` 里，直接裸用 `getenv()` 容易出现“本地正常，线上取不到”的问题。

---

## 5. JWT 与认证

当前硬约束：

- `JWT_SECRET` 只能来自 `.env*` 或部署平台环境变量
- 不要把 `JWT_SECRET` 写进 `anon.config.php`

如果要做浏览器登录，优先复用现有 `Auth` 能力：

- `Auth::issueTokenPair()`
- `Auth::setTokenPairCookies()`
- `Auth::forgetTokenPairCookies()`
- `Auth::refreshTokens()`

如果是第三方登录场景，但对方没有正式登录 API，不要默认承诺可以直接复用对方站点 Cookie。跨域 Cookie 在浏览器上下文本身就有限制。

---

## 6. 第三方 API 接入

当前推荐结构：

1. `Controller` 只读参数和组织响应
2. `Service` 负责第三方请求、签名、错误归一化
3. 出口统一走 `Response::success()` / `Response::error()`

不要把这些逻辑直接塞进控制器：

- 上游 URL 拼接
- 授权头构造
- SSL 选项处理
- 上游错误映射
- 限流头解析

这些都更适合放在 `Service`。

---

## 7. 配置缓存与路由缓存

当前最常见的误判之一就是：**代码没问题，但缓存没清。**

### 配置缓存

配置缓存文件：

```text
runtime/cache/config.php
```

如果你修改了：

- `anon.config.php`
- `.env*`

但运行结果没变，优先先看配置缓存。

常用命令：

```bash
php anon config:clear
php anon config:cache
```

### 路由缓存

路由缓存文件：

```text
runtime/cache/routes.php
```

如果你修改了：

- `app/Route/*.php`
- 控制器类名
- 控制器方法名

但路由还是旧的，优先看路由缓存。

常用命令：

```bash
php anon route:clear
php anon route:cache
```

---

## 8. framework/core 与运行时 vendor 的关系

当前排查框架问题时一定要分清楚：

- 你改的是 `framework/core`
- 还是项目实际运行的 `vendor/yuinijika/anon-core`

很多时候：

- 改了 `framework/core`
- 但项目实际执行的是 `project/.../vendor/yuinijika/anon-core`

那线上效果当然不会变。

所以要先判断 **当前生效副本** 是哪个，再决定：

- 只改 app
- 改 framework/core
- 同步改 runtime vendor

---

## 9. 什么时候该改 app，什么时候该改 core

默认原则：

1. 能在 `app` 解决，就先在 `app` 解决
2. 只有确认是框架通用缺陷，才改 `framework/core`

### 更像 app 问题的场景

- 新增业务接口
- 第三方 API 代理
- 登录逻辑
- 业务验证
- 资源格式整理

### 更像 core 问题的场景

- `Request` / `Response` 基础能力缺失
- `Env` 在生产环境不兼容
- 路由加载或生成器写死了错误目录
- ORM / QueryBuilder 是框架级逻辑错误
- HTTP Client 配置读取时机不对

---

## 10. 本地正常，线上异常时先查什么

推荐按这个顺序排：

1. 目录大小写
2. `Env::get()` / `getenv()` 使用方式
3. 配置缓存
4. 路由缓存
5. `composer dump-autoload`
6. `php-fpm` / opcache 是否还是旧代码
7. 系统环境变量是否覆盖了 `.env*`

这是目前最常用、也最省时间的排查顺序。

---

## 11. 文档阅读方式

现在建议这样看文档：

1. 先看这篇“当前约定总览”
2. 再按功能去查专题文档
3. 如果遇到旧示例，优先按当前约定修正理解

当前已经重点纠偏过的内容包括：

- 大写目录与 Linux 大小写敏感
- 数字型响应 `code`
- `Env::get()` 替代裸 `getenv()`
- `JWT_SECRET` 只走 `.env*`
- 生成器产物目录的大小写
- Provider / Middleware / Resource 示例命名空间

---

## 12. 部署交付清单

如果这次改动涉及配置、路由、自动加载或框架底层，交付时建议至少提醒这些：

- `composer dump-autoload`
- `php anon config:clear`
- `php anon route:clear`
- 必要时重新构建缓存
- 重启 `php-fpm` 或刷新 opcache
- 确认 Linux 目录大小写与命名空间一致

---

## 相关文档

- [部署检查清单](./deployment-checklist.md)
- [路由系统](./router.md)
- [请求与响应](./request-response.md)
- [项目配置](./configuration.md)
- [环境变量](./env.md)
- [身份认证](./auth.md)
- [服务提供者](./service-provider.md)
- [中间件](./middleware.md)
- [数据库操作](./database.md)
- [缓存系统](./cache.md)
- [文件存储](./storage.md)
- [会话管理](./session.md)
- [异步任务队列](./queue.md)

如果你只准备记住一套规则，就记这篇里的内容。
