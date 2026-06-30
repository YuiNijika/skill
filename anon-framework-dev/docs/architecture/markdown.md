# Markdown 解析

Anon Framework Next 引入了 `erusev/parsedown` 库，提供 Markdown 渲染支持。

## 基础用法

### 1. 文本转换

实例化 `Parsedown` 将 Markdown 文本解析为 HTML。

```php
use Parsedown;

$markdown = "### 欢迎使用 Anon\n\n这是一款**现代化的** PHP 框架。";

$parsedown = new Parsedown();
$html = $parsedown->text($markdown);

// 输出结果:
// <h3>欢迎使用 Anon</h3>
// <p>这是一款<strong>现代化的</strong> PHP 框架。</p>
```

### 2. 安全模式

在处理用户输入时，开启安全模式以过滤危险 HTML 标签。

```php
use Parsedown;

$maliciousInput = "[点击我](javascript:alert('XSS')) 或者输入 <script>alert(1)</script>";

$parsedown = new Parsedown();
// 开启安全模式
$parsedown->setSafeMode(true);

$html = $parsedown->text($maliciousInput);
```

### 3. 行内文本解析

解析行内样式，不使用 `<p>` 标签包裹：

```php
use Parsedown;

$parsedown = new Parsedown();
$html = $parsedown->line('只解析**加粗**和[链接](https://example.com)');
```

## 在 Response 中使用

将 Markdown 解析为 HTML 后返回：

```php
use Anon\Core\Http\Request;
use Anon\Core\Http\Response;
use Parsedown;

Route::post('/api/article/preview', function (Request $request) {
    $markdownContent = $request->input('content', '');
    
    $parsedown = new Parsedown();
    $parsedown->setSafeMode(true);
    
    return Response::success([
        'html' => $parsedown->text($markdownContent)
    ]);
});
```
