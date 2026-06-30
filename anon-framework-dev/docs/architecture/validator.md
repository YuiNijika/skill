# 数据验证器 (Validator)

Anon Framework Next 提供了一个数据验证器，用于验证用户提交的数据（如表单提交或 API 请求）。
## 基础使用

你可以通过 `Anon\Core\Facade\Validator` 门面类创建一个验证器实例。

### 验证数据

以下是一个常见的用户注册验证示例：

```php
use Anon\Core\Facade\Validator;
use Anon\Core\Http\Request;
use Anon\Core\Http\Response;

Route::post('/register', function (Request $request) {
    // 获取提交的数据
    $data = $request->post();
    
    // 创建验证器
    $validator = Validator::make($data, [
        'username' => 'required|max:20',
        'email'    => 'required|email',
        'age'      => 'numeric|min:18'
    ], [
        // 自定义错误提示（可选）
        'username.required' => '必须填写用户名',
        'username.max'      => '用户名长度不能超过20',
        'email.required'    => '必须填写邮箱',
        'email.email'       => '邮箱格式不合法',
        'age.numeric'       => '年龄必须是数字',
        'age.min'           => '未满18岁禁止访问'
    ]);

    // 检查是否验证失败
    if ($validator->fails()) {
        // 返回第一个错误信息
        return Response::error($validator->firstError(), 400, $validator->errors());
    }

    // 验证通过，继续处理业务逻辑...
    return Response::success($data, '注册成功');
});
```

## 可用验证规则

框架内置了以下常用验证规则：

| 规则 | 说明 | 示例 |
| --- | --- | --- |
| `required` | 字段不能为空（null、空字符串、空数组） | `required` |
| `email` | 必须是一个合法的电子邮件地址 | `email` |
| `max:value` | 最大值。对于字符串是最大长度；对于数字是最大数值；对于数组是最大元素个数。 | `max:255` |
| `min:value` | 最小值。同上。 | `min:6` |
| `numeric` | 必须是数字或数字字符串 | `numeric` |
| `integer` | 必须是整数 | `integer` |
| `in:foo,bar` | 字段值必须在给定的列表中 | `in:admin,user,guest` |

## 错误处理

验证失败后，你可以通过以下方法获取错误信息：

```php
// 获取所有字段的所有错误信息数组
$errors = $validator->errors();

// 获取第一条错误信息（字符串），适合作为 API 的 errorMessage 返回
$first = $validator->firstError();
```
