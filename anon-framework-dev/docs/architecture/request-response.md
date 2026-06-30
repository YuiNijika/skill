# 请求与响应

Anon Framework Next 是一个 RESTful API 框架。底层接管了 PHP 原生的 `$_GET`, `$_POST` 等操作，封装为 `Request` 和 `Response` 对象。

---

## Request 请求对象

`Anon\Core\Http\Request` 类负责捕获当前 HTTP 请求的环境信息，提供获取输入数据的 API。在路由闭包或控制器方法中，可以通过参数注入的方式获取 `$request` 实例。

### 捕获输入数据

使用 `$request->input()` 方法获取前端参数。该方法会自动从 `GET`、`POST` 以及 `application/json` Body 中提取数据。

```php
use Anon\Core\Http\Request;

Route::post('/user/update', function (Request $request) {
    // 获取 username 参数
    $username = $request->input('username');
    
    // 支持设置默认值
    $age = $request->input('age', 18);
    
    return ['username' => $username, 'age' => $age];
});
```

#### 数据安全过滤

`input()` 方法提供第三个参数 `$filter`，在获取数据的同时进行过滤与转换。

支持的过滤器：`int`, `float`, `string`, `bool`, `email`, `url`, `xss`。

```php
Route::post('/comment', function (Request $request) {
    // 转换为整型
    $postId = $request->input('post_id', 0, 'int');
    
    // 转义 HTML 实体标签
    $content = $request->input('content', '', 'xss');
    
    // 对整个数组递归进行 xss 过滤
    $allSafeData = $request->input(null, null, 'xss');
});
```

### 获取请求基础信息

获取请求的类型、URI 以及请求头。

```php
Route::get('/test', function (Request $request) {
    $method = $request->method();
    $uri = $request->uri();
    
    // 访问原始 HTTP Header 数组
    $headers = $request->header;
    
    // 获取当前站点与 URL 信息
    $host = $request->host();         // 例如: 127.0.0.1:8000
    $baseUrl = $request->baseUrl();   // 例如: http://127.0.0.1:8000
    $fullUrl = $request->fullUrl();   // 例如: http://127.0.0.1:8000/test?from=docs
    
    return compact('method', 'uri', 'baseUrl', 'fullUrl');
});
```

### 获取请求类型与格式

```php
Route::post('/api/data', function (Request $request) {
    // 检查是否为指定请求方法
    if ($request->isMethod('post')) {
        // ...
    }

    // 判断是否是 AJAX 请求 (检查 X-Requested-With)
    if ($request->isAjax()) {
        // ...
    }

    // 判断当前请求提交的内容是否是 JSON 格式
    if ($request->isJson()) {
        // ...
    }

    // 判断客户端是否期望返回 JSON 响应 (检查 Accept 头或 AJAX)
    if ($request->wantsJson()) {
        // ...
    }
});
```

### 路径与模式匹配

你可以使用 `is` 方法来验证请求的路径是否匹配给定的模式，支持 `*` 通配符。

```php
// 请求路径: /admin/users/1

$request->is('admin/*'); // true
$request->is('admin/users/*'); // true
$request->is('api/*'); // false

// 获取纯净路径 (不含 Query)
$path = $request->path(); // "/admin/users/1"

// 获取查询字符串 (Query String)
$qs = $request->queryString(); 
```

### 获取客户端 IP 地址

使用 `ip` 方法获取客户端 IP 地址。该方法会提取 `X-Forwarded-For` 代理层的真实 IP，进行格式校验，并将 IPv6 本地回环地址（`::1`）转化为 IPv4 `127.0.0.1`。

```php
Route::get('/ip', function (Request $request) {
    $clientIp = $request->ip();
    
    return ['ip' => $clientIp];
});
```

### 获取路由参数

在路由定义中使用的动态参数（如 `/user/{id}`），可以通过 `route` 方法获取：

```php
// 获取单个参数
$id = $request->route('id');

// 获取所有参数
$params = $request->route();
```

### 获取请求头与 Cookie

```php
// 获取请求头 (不区分大小写)
$token = $request->header('Authorization');

// 获取 Bearer Token
$token = $request->bearerToken();

// 获取 Cookie
$sessionId = $request->cookie('PHPSESSID');
```

### 获取上传文件

使用 `file` 方法获取上传的文件。返回 `Anon\Core\Http\UploadedFile` 实例：

```php
// 获取上传文件
$file = $request->file('avatar');

if ($file && $file->isValid()) {
    $name = $file->getClientOriginalName();
    $extension = $file->getClientOriginalExtension();
    
    // 如果不传路径，将默认保存到 anon.config.php 中的 upload.path (默认为 run/storage)
    $path = $file->move();
    
    // 你也可以指定特定目录
    // $path = $file->move(BASE_PATH . '/run/storage/avatars');
    
    echo "文件已保存至: " . $path;
}
```

检查请求中是否包含有效文件：

```php
if ($request->hasFile('avatar')) {
    // 文件存在且上传有效
}
```

---

## Response 响应对象

`Response` 负责把接口结果送回客户端。默认会按 JSON 输出，并自动带上 `Content-Type: application/json; charset=utf-8`。

### 直接返回数组或对象

控制器或路由里直接返回数组、对象也可以，路由器会帮你包成成功响应。

```php
Route::get('/list', function () {
    return [
        ['id' => 1, 'name' => 'Alice'],
        ['id' => 2, 'name' => 'Bob']
    ];
});
```

### 推荐的标准响应

多数 API 建议直接用 `Response::success()` 和 `Response::error()`。这样前端拿到的结构会一直稳定。

#### 成功响应
```php
use Anon\Core\Http\Response;

Route::get('/user/profile', function () {
    $user = ['id' => 100, 'name' => 'Anon'];

    return Response::success($user);
});
```

返回结构大概是：

```json
{
  "success": true,
  "code": 200,
  "message": "OK",
  "data": {
    "id": 100,
    "name": "Anon"
  }
}
```

如果这个接口有更明确的业务语义，也可以自己给消息、状态码和业务码：

```php
return Response::success($user, 'Created', 201, 'USER_CREATED');
```

这里的 `code` 推荐始终保持为数字型 HTTP 状态码；像 `USER_CREATED` 这类业务语义应放到 `business_code` 或错误时的 `error_code` 中，而不是覆盖 `code` 字段本身。

#### 失败响应
```php
use Anon\Core\Http\Response;

Route::post('/user/pay', function (Request $request) {
    $amount = $request->input('amount');

    if ($amount < 0) {
        return Response::error('金额不能为负数', 400, ['amount' => ['金额不能为负数']], 'BAD_REQUEST');
    }

    return Response::success(null, '支付成功');
});
```

返回结构大概是：

```json
{
  "success": false,
  "code": "BAD_REQUEST",
  "message": "金额不能为负数",
  "errors": {
    "amount": ["金额不能为负数"]
  }
}
```

框架抛出的常见异常也会走同一套结构，比如 `UNAUTHORIZED`、`FORBIDDEN`、`NOT_FOUND`、`VALIDATION_FAILED`、`TOO_MANY_REQUESTS`、`INTERNAL_ERROR`。异常响应里会带 `trace_id`，方便前端报错时和服务端日志对上。

### 原始 JSON 输出

如果某个接口就是想原样输出，不需要标准 envelope，可以显式使用 `Response::json()`。

```php
return Response::json(['status' => 'ok'], 201);
```

### 自定义 HTTP 状态码与头信息

链式调用方法设置状态码和头信息：

```php
use Anon\Core\Http\Response;

Route::get('/custom', function () {
    return Response::json(['status' => 'ok'])
        ->setStatusCode(201)
        ->setHeader('X-Custom-Header', 'AnonFramework');
});
```
