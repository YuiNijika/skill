# Widget 小部件

在构建复杂的 API 或后端服务时，常常会遇到需要在不同控制器或接口中复用同一块业务逻辑的场景（例如获取导航菜单数据、计算通用统计面板数据等）。

Anon Framework Next 提供了轻量级的 `Widget` 组件，允许你注册并调用这些可复用的代码块。

---

## 注册 Widget

你可以使用闭包或独立的类来注册一个 Widget。通常，Widget 的注册应该在 `app/hook.php` 的 `app_init` 阶段或专门的服务提供者中进行。

### 1. 使用闭包注册

```php
use Anon\Core\Support\Widget;
use Anon\Core\Facade\Hook;

Hook::add('app_init', function () {
    Widget::register('site_stats', function ($type = 'daily') {
        // 执行复杂的统计逻辑
        return [
            'type' => $type,
            'users' => 1024,
            'orders' => 256
        ];
    });
});
```

### 2. 使用类注册

对于逻辑较复杂的 Widget，建议使用类来进行封装。类必须包含一个 `render` 方法。

```php
namespace App\Widget;

class UserMenuWidget
{
    public function render(int $roleId)
    {
        if ($roleId === 1) {
            return ['Dashboard', 'Users', 'Settings'];
        }
        return ['Profile', 'Orders'];
    }
}
```

然后进行注册：

```php
use Anon\Core\Support\Widget;
use App\Widget\UserMenuWidget;

Widget::register('user_menu', UserMenuWidget::class);
```

> **提示**：如果 Widget 是以类名注册的，框架会在调用时通过 IoC 容器自动实例化它，这意味着你的 Widget 类的构造函数也支持自动依赖注入。

---

## 调用 Widget

在任何控制器或业务代码中，你可以通过 `Widget::call` 快速执行并获取 Widget 的返回值。

```php
namespace Anon\Controller;

use Anon\Core\Http\Response;
use Anon\Core\Support\Widget;

class DashboardController
{
    public function index(): Response
    {
        // 调用闭包注册的 Widget
        $stats = Widget::call('site_stats', ['monthly']);
        
        // 调用类注册的 Widget
        $menu = Widget::call('user_menu', [1]);

        return Response::json([
            'stats' => $stats,
            'menu' => $menu
        ]);
    }
}
```
