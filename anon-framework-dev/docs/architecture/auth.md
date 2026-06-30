# 身份认证 (Auth)

Anon Framework Next 提供了基于 JWT (JSON Web Token) 的无状态身份认证模块，非常适合 API 开发。

## 配置

默认骨架建议只保留最小认证配置，在 `.env*` 中保存密钥：

```php
return [
    'auth' => [
        // 无需在此配置 jwt_secret，框架会自动从 .env 中的 JWT_SECRET 读取
    ],
];
```

这已经足够支撑最基础的 JWT 登录与鉴权。

如果你需要多 guard、Cookie 闭环、双令牌、会话跟踪等高级能力，再按需追加配置即可。

### 可选高级配置

下面是一份更完整的认证配置示例，用于多 guard、Cookie、Refresh Token 等场景：

```php
return [
    'auth' => [
        'default_guard' => 'api',
        'token_sources' => ['header', 'cookie'],
        'header_name' => 'Authorization',
        'header_prefix' => 'Bearer',
        'cookie_enabled' => true,
        'cookie_name' => 'access_token',
        'cookie_path' => '/',
        'cookie_domain' => '',
        'cookie_secure' => false,
        'cookie_httponly' => true,
        'cookie_samesite' => 'Lax',
        'refresh_enabled' => true,
        'refresh_ttl' => 604800,
        'refresh_token_sources' => ['cookie', 'header'],
        'refresh_header_name' => 'X-Refresh-Token',
        'refresh_header_prefix' => 'Bearer',
        'refresh_cookie_enabled' => true,
        'refresh_cookie_name' => 'refresh_token',
        'refresh_cookie_path' => '/',
        'refresh_cookie_domain' => '',
        'refresh_cookie_secure' => false,
        'refresh_cookie_httponly' => true,
        'refresh_cookie_samesite' => 'Lax',
        'refresh_query_key' => 'refresh_token',
        'session_track_enabled' => true,
        'session_prefix' => 'auth:session:',
        'session_index_prefix' => 'auth:session:index:',
        'query_key' => 'access_token',
        'blacklist_prefix' => 'auth:blacklist:',
        'subject_key' => 'id',
        'guards' => [
            'api' => [
                'ttl' => 7200,
            ],
            'admin' => [
                'ttl' => 7200,
                'token_sources' => ['header', 'cookie'],
                'cookie_enabled' => true,
                'cookie_name' => 'admin_access_token',
                'refresh_enabled' => true,
                'refresh_ttl' => 604800,
                'refresh_cookie_name' => 'admin_refresh_token',
            ],
        ],
    ],
];
```

## 生成 Token

你可以通过 `Anon\Core\Facade\Auth` 门面为用户生成 Token：

```php
use Anon\Core\Facade\Auth;

// 传入用户数组或对象，默认有效期 7200 秒
$token = Auth::login(['id' => 1, 'name' => 'admin']);
```

你也可以显式指定 Guard 并附加角色、权限等声明：

```php
$token = Auth::login(
    [
        'id' => 1,
        'roles' => ['admin'],
        'permissions' => ['user.read', 'user.update'],
    ],
    7200,
    'admin'
);
```

## 验证与获取用户信息

客户端需要在 HTTP 请求头中携带 Token：
`Authorization: Bearer <your_token>`

服务端获取并验证：

```php
use Anon\Core\Facade\Auth;

// 检查当前请求是否已认证
if (Auth::check()) {
    // 返回解析后的 JWT payload 数组
    $user = Auth::user(); 
}
```

## Guard

框架当前支持基于 Guard 的多密钥认证。你可以按不同客户端、后台、开放接口定义不同 Guard：

```php
use Anon\Core\Facade\Auth;

$admin = Auth::guard('admin')->user();
$adminId = Auth::guard('admin')->id();
```

如果你不显式指定 Guard，则会使用 `auth.default_guard`。

Guard 配置还可以覆盖全局认证行为，例如：

- `token_sources`
- `header_name`
- `header_prefix`
- `cookie_enabled`
- `cookie_name`
- `cookie_path`
- `cookie_domain`
- `cookie_secure`
- `cookie_httponly`
- `cookie_samesite`
- `refresh_enabled`
- `refresh_secret`
- `refresh_ttl`
- `refresh_token_sources`
- `refresh_header_name`
- `refresh_header_prefix`
- `refresh_cookie_enabled`
- `refresh_cookie_name`
- `refresh_cookie_path`
- `refresh_cookie_domain`
- `refresh_cookie_secure`
- `refresh_cookie_httponly`
- `refresh_cookie_samesite`
- `refresh_query_key`
- `session_track_enabled`
- `session_prefix`
- `session_index_prefix`
- `query_key`
- `subject_key`

这意味着你可以为后台管理端单独设置从 Cookie 读取 Token，而普通 API 仍使用 `Authorization: Bearer ...`。

## Token 来源解析

当前 Auth 模块支持按配置顺序从以下位置解析 Token：

- `header`
- `cookie`
- `query`

例如：

```php
'auth' => [
    'token_sources' => ['header', 'cookie'],
    'cookie_name' => 'access_token',
]
```

此时框架会优先读取请求头中的 Bearer Token，如果没有，再回退到 Cookie 中的 `access_token`。

你也可以直接获取当前请求中解析到的原始 Token：

```php
$token = Auth::token();
```

## Cookie 闭环

当你希望浏览器端走更完整的 Cookie 认证流程时，可以开启 `cookie_enabled`，并把 `cookie` 放进 `token_sources`：

```php
'auth' => [
    'token_sources' => ['header', 'cookie'],
    'cookie_enabled' => true,
    'cookie_name' => 'access_token',
    'cookie_path' => '/',
    'cookie_httponly' => true,
    'cookie_samesite' => 'Lax',
]
```

此时常见行为如下：

- 登录成功后，把 JWT 写入响应 Cookie
- 后续请求即使不手动带 `Authorization`，也可以从 Cookie 中解析 Token
- `refresh()` 返回新 Token 时，同步刷新响应 Cookie
- `logout()` 时，把当前 Token 拉黑并清理响应 Cookie

如果你有多个 Guard，建议为不同 Guard 配置不同的 `cookie_name`，避免浏览器端互相覆盖。

对于双令牌场景，也建议同时为不同 Guard 配置不同的 `refresh_cookie_name`。

## 双令牌模式

当前骨架默认启用 `access_token + refresh_token` 双令牌模式：

- `access_token` 用于访问业务接口，默认有效期较短
- `refresh_token` 仅用于续期，默认有效期较长
- 两者都会带上 `guard` 和 `typ` 声明，避免 access / refresh 混用
- `refresh_token` 默认优先从 Cookie 读取，也支持通过 `X-Refresh-Token: Bearer <token>` 传入

你可以直接签发一组 Token：

```php
$tokens = Auth::issueTokenPair(['id' => 1, 'name' => 'admin']);
```

返回结果示例：

```php
[
    'token' => '...',
    'access_token' => '...',
    'refresh_token' => '...',
    'token_type' => 'Bearer',
    'expires_in' => 7200,
    'refresh_expires_in' => 604800,
    'guard' => 'api',
]
```

## 会话跟踪

虽然 JWT 本身是无状态的，但当前实现已经额外把登录会话同步记录到缓存中：

- 每次登录或刷新都会生成或续用一个 `sid`
- access token 和 refresh token 都会带上同一个 `sid`
- 服务端会把该 `sid` 对应的会话信息写入缓存
- 后续鉴权时不仅校验签名和黑名单，也会校验对应会话是否仍存在

这带来两个直接收益：

- 删除某个会话记录后，对应 access / refresh token 会立即失效
- 可以实现“查看我的设备”、“踢掉某台设备”、“保留当前设备并踢掉其他设备”

典型会话信息包括：

- `session_id`
- `guard`
- `subject`
- `created_at`
- `last_seen_at`
- `last_ip`
- `user_agent`
- `access_expires_at`
- `refresh_expires_at`

## 刷新 Token

当客户端需要延长会话时，可以基于当前请求中的 `refresh_token` 生成新的 Token 对：

```php
$tokens = Auth::refreshTokens();

// 也可以指定 guard、access ttl 和 refresh ttl
$adminTokens = Auth::refreshTokens('admin', 3600, 1209600);
```

刷新后，旧的 `refresh_token` 会立即进入黑名单；如果当前请求还带着旧的 `access_token`，它也会一并失效。

如果当前 Guard 已启用 Cookie，新生成的 access / refresh Token 也会同步回写到响应 Cookie。

## 退出登录

```php
Auth::logout();
```

退出后，当前 Token 会立即进入黑名单；在同一次请求内继续调用 `Auth::check()` 时，也会重新按黑名单规则校验，不会继续沿用旧结果。

如果当前 Guard 已启用 Cookie，退出时也会同步清理对应 Cookie。

## 角色与权限检查

JWT Payload 中可直接声明 `role`、`roles`、`permissions` 字段，随后通过 `Auth` 进行检查：

```php
if (Auth::hasRole(['admin', 'editor'])) {
    // ...
}

if (Auth::hasPermission('user.update')) {
    // ...
}
```

也可以直接抛出标准 HTTP 异常：

```php
Auth::authorizeRole('admin');
Auth::authorizePermission(['user.read', 'user.update']);
```

## 退出登录与 Token 黑名单

调用 `logout()` 时，框架会将当前请求中命中的 access / refresh Token 的 `jti` 写入缓存黑名单，直到它们自然过期：

```php
Auth::logout();
```

这意味着：

- 已退出的旧 Token 无法继续访问受保护接口
- `refresh()` 生成新 Token 时，也会自动让旧 Token 失效

## 中间件

框架内置了三类常用鉴权中间件：

- `Authenticate`：只检查当前请求是否已登录
- `RequireRole`：检查角色
- `RequirePermission`：检查权限

### 登录校验

```php
use Anon\Core\Http\Middleware\Authenticate;

Route::get('/profile', function () {
    return ['ok' => true];
})->middleware(Authenticate::class);
```

### 指定 Guard

```php
Route::get('/admin/profile', function () {
    return ['ok' => true];
})->middleware(Authenticate::class . ':admin');
```

### 角色校验

`RequireRole` 使用 `|` 分隔多个角色：

```php
use Anon\Core\Http\Middleware\RequireRole;

Route::get('/admin/users', function () {
    return ['ok' => true];
})->middleware(RequireRole::class . ':admin|super-admin,admin');
```

上面的示例中：

- 第一个参数 `admin|super-admin` 是允许的角色列表
- 第二个参数 `admin` 是 Guard 名称

### 权限校验

```php
use Anon\Core\Http\Middleware\RequirePermission;

Route::get('/admin/user/update', function () {
    return ['ok' => true];
})->middleware(RequirePermission::class . ':user.update|user.manage,admin');
```

## 骨架示例接口

当前 `skeleton` 已内置一组认证示例接口，便于直接联调：

```bash
# 普通登录
POST /auth/login

# admin guard 登录
POST /auth/admin/login

# 获取当前用户
GET /auth/user

# 刷新 token 对
POST /auth/refresh

# 退出登录
POST /auth/logout

# 查看当前用户会话列表
GET /auth/sessions

# 吊销指定会话
POST /auth/sessions/revoke

# 吊销其他会话
POST /auth/sessions/revoke-others

# admin guard 认证
GET /auth/admin/panel

# admin 角色 + 权限校验
GET /auth/admin/ability
```

默认骨架配置下：

- `POST /auth/login` 会返回 `access_token + refresh_token`，并写入 `access_token` / `refresh_token` Cookie
- `POST /auth/refresh` 会基于 refresh token 签发新的 token 对，并同步刷新两枚 Cookie
- `POST /auth/logout` 会拉黑当前 access / refresh token，并清理对应 Cookie
- `GET /auth/sessions` 会返回当前会话和全部活跃会话列表
- `POST /auth/sessions/revoke` 可按 `session_id` 吊销指定会话
- `POST /auth/sessions/revoke-others` 会保留当前设备并吊销其他设备
- `POST /auth/admin/login` 会把 admin guard 的 token 对写入 `admin_access_token` / `admin_refresh_token`

控制器写法示例：

```php
public function authLogin(): Response
{
    $tokens = Auth::issueTokenPair(['id' => 1, 'name' => 'admin']);
    $response = Response::success($tokens, 'Login Success');

    return Auth::setTokenPairCookies($response, $tokens);
}

public function authRefresh(): Response
{
    $tokens = Auth::refreshTokens();
    if ($tokens === null) {
        return Response::error('Unauthorized', 401);
    }

    return Auth::setTokenPairCookies(Response::success($tokens, 'Refresh Success'), $tokens);
}

public function authLogout(): Response
{
    if (!Auth::logout()) {
        return Response::error('Unauthorized', 401);
    }

    return Auth::forgetTokenPairCookies(Response::success(null, 'Logout Success'));
}
```

吊销指定会话示例：

```php
public function authRevokeSession(Request $request): Response
{
    $sessionId = trim((string) $request->input('session_id', ''));
    if ($sessionId === '') {
        return Response::error('session_id is required', 400);
    }

    if (!Auth::revokeSession($sessionId)) {
        return Response::error('Session Not Found', 404);
    }

    return Response::success(['session_id' => $sessionId], 'Session Revoked');
}
```

## 高级用法：多端 Token 隔离

如果你的系统比较复杂，比如既有普通用户（User）登录，又有后台管理员（Admin）登录，或者第三方 API 客户端授权，你可以直接使用底层的 `JWTUtil` 工具类，并传入独立密钥，或者在 `auth.guards` 中声明多个 Guard。

### 1. 配置独立密钥
在 `.env*` 中定义多个 Secret：
```env
JWT_SECRET=change_this_to_a_random_secret
JWT_ADMIN_SECRET=change_this_to_a_random_admin_secret
JWT_API_SECRET=change_this_to_a_random_api_secret
```

这些 Secret 只应存在于 `.env*` 或部署平台环境变量中，不应写回 `anon.config.php`，也不应在仓库里保留可直接使用的默认密钥。

### 2. 签发专属 Token
```php
use Anon\Core\Auth\JWTUtil;
use Anon\Core\Facade\Env;

// 给 API 客户端生成一个有效期 1 年的专属 Token
$payload = [
    'sub'  => 'client_id_1001',
    'role' => 'api_client',
    'iat'  => time(),
    'exp'  => time() + 31536000 
];

$apiSecret = Env::get('JWT_API_SECRET');
$apiToken = JWTUtil::encode($payload, $apiSecret);
```

### 3. 验证专属 Token
你可以编写一个专门的 API 授权中间件来解析它。由于签名时使用的是专属密钥，拿普通用户的 Token 尝试访问此接口时，底层 `hash_hmac` 会直接拒绝并抛出异常，从而实现了极度安全的权限隔离。

```php
use Anon\Core\Auth\JWTUtil;
use Anon\Core\Facade\Env;
use Anon\Core\Exception\Http;
use Anon\Core\Http\Request;

class ApiAuthMiddleware
{
    public function handle(Request $request, \Closure $next)
    {
        $token = $request->bearerToken();
        $apiSecret = Env::get('JWT_API_SECRET');

        try {
            // 必须使用专用密钥才能解码成功
            $payload = JWTUtil::decode($token, $apiSecret);
            
            // 可以将解析后的身份信息存入 request 供下游控制器使用
            $request->apiClient = $payload;
            
        } catch (\Exception $e) {
            throw new Http(401, 'Invalid API Token');
        }

        return $next($request);
    }
}
```
