# 辅助函数 (Support)

Anon Framework 提供了 `Anon\Core\Support` 命名空间下的各种辅助工具类，帮助开发者快速完成字符串处理、数据转换等常用操作。

## 字符串处理 (`Str`)

`Str` 助手类位于 `Anon\Core\Support\Str`，提供了生成随机字符串、UUID，以及命名风格转换等静态方法。

### 生成 UUID

使用 `Str::uuid()` 方法可以快速生成一个标准的 UUID v4（RFC 4122）。底层优先使用安全的 `random_bytes()` 函数，若环境不支持则拥有随机回退机制。

```php
use Anon\Core\Support\Str;

$uuid = Str::uuid();
// 输出类似： "d3b07384-d9a4-4f41-b91c-7f8d6f5195b2"
```

### 生成随机字符串

使用 `Str::random()` 可以生成指定长度的安全随机字符串，默认长度为 16 位。

```php
use Anon\Core\Support\Str;

$random16 = Str::random();
// 输出类似： "kL9xY2vPqA1mN5zT"

$random32 = Str::random(32);
// 输出类似： "m9LpXq2vN5zT1kL9xY2vPqA1mN5zT1kL"
```

### 命名风格转换

你可以方便地在“驼峰命名 (Camel Case)”和“蛇形命名 (Snake Case)”之间进行互相转换。

**转为蛇形命名：**
```php
$snake = Str::snake('CamelCaseString');
// 输出: "camel_case_string"
```

**转为驼峰命名：**
```php
$camel = Str::camel('snake_case_string');
// 输出: "snakeCaseString"
```

### 前后缀判断

`Str` 还提供了语义化的前缀和后缀判断方法，支持传入单个字符串或数组进行匹配。

**判断开头：**
```php
$isHttp = Str::startsWith('https://anon.dev', ['http://', 'https://']);
// 返回 true
```

**判断结尾：**
```php
$isImage = Str::endsWith('avatar.png', ['.png', '.jpg', '.jpeg']);
// 返回 true
```
