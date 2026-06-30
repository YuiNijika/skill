# 事件系统 (Event)

Anon Framework Next 提供了一个简单轻量的事件观察者（发布/订阅）机制。事件系统允许你监听和触发应用程序中发生的动作，非常适合对应用解耦。

## 基础使用

你可以通过 `Anon\Core\Facade\Event` 门面类来注册监听器或触发事件。

### 注册监听器 (Listen)

你可以监听一个字符串名称的事件：

```php
use Anon\Core\Facade\Event;
use Anon\Core\Facade\Log;

// 使用闭包注册监听器
Event::listen('user.login', function ($user) {
    Log::info("用户已登录：" . $user['name']);
});
```

除了闭包，你还可以使用类的指定方法作为监听器：

```php
// 使用 "Class@method" 字符串形式
Event::listen('user.login', 'App\Listeners\UserLoginListener@handle');

// 或者如果方法名就是 handle，可以直接简写类名
Event::listen('user.login', 'App\Listeners\UserLoginListener');

// 使用数组形式
Event::listen('user.login', [\App\Listeners\UserLoginListener::class, 'handle']);
```

> **注意：** 使用类作为监听器时，事件系统会自动通过 **依赖注入容器** (`Container`) 实例化该类。

### 触发事件 (Dispatch)

你可以传递数据载荷给监听器，触发事件后会返回所有监听器的执行结果数组：

```php
// 触发基于字符串的事件
$responses = Event::dispatch('user.login', ['name' => 'Anon Admin']);
```

### 停止事件传播

如果你的某个监听器在执行完毕后，希望停止当前事件继续向其他监听器传播，只需在监听器中返回严格的 `false` 即可：

```php
Event::listen('user.login', function ($user) {
    // 权限校验失败，拦截事件
    return false;
});
```

## 对象事件

除了基于字符串的事件，你还可以直接分发一个事件对象，系统会自动将其类名作为事件名：

```php
namespace App\Events;

class UserRegistered
{
    public $user;
    
    public function __construct($user)
    {
        $this->user = $user;
    }
}
```

触发对象事件时，无需传递第二个 `$payload` 参数，系统会自动将事件对象本身作为载荷传递给监听器：

```php
// 触发事件
Event::dispatch(new \App\Events\UserRegistered($user));

// 监听对象事件
Event::listen(\App\Events\UserRegistered::class, function ($event) {
    echo $event->user['name'];
});
```
