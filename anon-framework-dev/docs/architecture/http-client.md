# HTTP 客户端

Anon Framework Next 提供了 HTTP 客户端封装。底层基于 PHP 的 `cURL` 扩展，使得向其他服务器或第三方 API（如微信支付、第三方登录等）发送对外 HTTP 请求变得轻而易举。

你可以通过 `Anon\Core\Facade\Http` 门面来快速发起请求。

---

## 发起请求

### GET 请求

使用 `Http::get` 方法可以发起一个 GET 请求。第二个参数为 Query 字符串数组，会自动拼接到 URL 后面。

```php
use Anon\Core\Facade\Http;

$response = Http::get('https://api.github.com/users/yuinijika', [
    'per_page' => 10
]);

// 请求的实际 URL 为: https://api.github.com/users/yuinijika?per_page=10
```

### POST 请求

使用 `Http::post` 方法可以发起 POST 请求。默认情况下，如果传入了数组，框架会自动将其转换为 `JSON` 格式，并自动加上 `Content-Type: application/json` 请求头。

```php
use Anon\Core\Facade\Http;

$response = Http::post('https://api.example.com/login', [
    'username' => 'admin',
    'password' => 'secret123'
]);
```

### PUT / DELETE 请求

类似于 GET 和 POST，框架也提供了对应的快捷方法：

```php
use Anon\Core\Facade\Http;

// PUT 请求
Http::put('https://api.example.com/users/1', ['name' => 'New Name']);

// DELETE 请求
Http::delete('https://api.example.com/users/1');
```

---

## 响应结构

每次调用 `Http` 门面的方法，都会返回一个包含响应结果的数组。结构如下：

```php
[
    'status'  => 200,          // HTTP 状态码
    'headers' => [             // 解析后的响应头数组
        'Content-Type' => 'application/json',
        'Server'       => 'nginx'
    ],
    'body'    => '{"id":1}',   // 原始响应体字符串
    'json'    => ['id' => 1]   // 尝试自动将 body JSON Decode 后的数组（如果不是 JSON 则为 null）
]
```

### 获取返回数据

通过访问数组下标可以快速获取所需的数据：

```php
$response = Http::get('https://api.example.com/data');

if ($response['status'] === 200) {
    // 直接获取解析后的 JSON 数据
    $data = $response['json'];
    var_dump($data);
}
```

---

## 自定义请求头

如果需要在请求中携带自定义的 Header（例如 `Authorization` Token），可以将其作为最后一个参数传入（`GET` 为第三个参数，`POST/PUT/DELETE` 为第三个参数）：

```php
use Anon\Core\Facade\Http;

$headers = [
    'Authorization' => 'Bearer your_token_here',
    'X-Custom-Header' => 'Value'
];

$response = Http::post('https://api.example.com/protected', [
    'key' => 'value'
], $headers);
```

---

## 文件上传

如果你在 POST 请求的数组数据中传入了 `\CURLFile` 对象，HTTP 客户端会自动将请求类型切换为 `multipart/form-data` 来实现文件上传。

```php
use Anon\Core\Facade\Http;

$response = Http::post('https://api.example.com/upload', [
    'description' => 'Profile Picture',
    'file' => new \CURLFile('/path/to/local/image.jpg', 'image/jpeg', 'avatar.jpg')
]);
```

---

## 异常处理

如果 `cURL` 扩展没有加载，或者请求过程中发生了底层网络错误（如 DNS 解析失败、连接超时），客户端会抛出 `\Exception` 异常。你可以使用 `try...catch` 进行捕获：

```php
use Anon\Core\Facade\Http;

try {
    $response = Http::get('https://invalid-domain.local');
} catch (\Exception $e) {
    echo "请求失败：" . $e->getMessage();
}
```