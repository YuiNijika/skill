---
name: "anon-framework-core-dev"
description: "Builds and fixes Anon Framework Next core internals. Invoke when modifying framework/core behavior, base HTTP/Auth/Env/ORM systems, generators, or cross-project compatibility logic."
---

# Anon Framework Core Dev

Use this skill only when the task clearly belongs to `framework/core` or the runtime `anon-core` package itself.

This skill is stricter than the app-level skill. Core changes can affect every Anon project, so default to minimal, compatibility-first fixes.

## When To Invoke

Invoke this skill when working on:

- `framework/core/src/**`
- runtime `vendor/yuinijika/anon-core/src/**`
- framework-level Request / Response / Auth / Env / Config behavior
- route loading or CLI generators
- ORM / query builder / migration base behavior
- cache, queue, session, storage base implementations
- OpenAPI generator or Action framework internals

Do not invoke this skill for normal app business APIs unless the root cause is proven to be inside the framework.

## Core Principles

- fix root causes, not app-specific symptoms
- preserve backward compatibility whenever possible
- prefer additive compatibility over breaking rewrites
- explain exactly why the issue belongs to framework core
- if the project runs against vendored `anon-core`, remember the effective runtime copy may need the same fix

## Decision Gate

Before changing core, confirm at least one of these is true:

1. the same issue would affect multiple Anon apps
2. app-level workaround would be duplicated or unsafe
3. a base class / facade / container capability is missing or wrong
4. PSR-4, env loading, response shape, auth parsing, or ORM semantics are broken at framework level

If none are true, push the change back to `app`.

## High-Risk Areas

Treat these as sensitive:

- `Support/Env.php`
- `Http/Response.php`
- `Http/Request.php`
- `Http/Client.php`
- `Auth/Manager.php`
- `Auth/JWTUtil.php`
- `Foundation/App.php`
- `Routing/Router.php`
- `Database/QueryBuilder.php`
- `Database/Model.php`
- `Database/ModelQueryBuilder.php`
- `Console/Commands/Make/*`

For these files, always think about Linux compatibility, runtime cache, and old project behavior.

## Known Project-Wide Constraints

### 1. Directory Case

PSR-4 directories must match namespace case exactly on Linux.

Prefer uppercase app directories:

- `Controller`
- `Service`
- `Route`
- `Action`
- `Model`
- `Middleware`
- `Provider`

If changing route loaders or generators, prefer uppercase first and remain compatible with legacy lowercase layouts.

### 2. Environment Loading

Framework code must not depend on raw `getenv()` behavior.

Reason:

- production may disable `putenv()`
- framework must fallback to `$_ENV` / `$_SERVER`
- direct `getenv()` can pass locally and fail online

For framework-level env access, the environment subsystem must remain robust in restricted hosting environments.

### 3. Response Contract

Current normalized API contract:

- `success` is boolean
- `code` is numeric HTTP status code
- `message` is string
- `error_code` is for business errors
- `business_code` is optional for successful business semantics

Do not regress `code` back to string values like `"OK"`.

### 4. Config Behavior

Framework config may be cached in `runtime/cache/config.php`.

When changing config-sensitive systems:

- ensure runtime lookup behavior is correct
- do not assume config file changes are immediately visible if cache exists
- mention cache clearing in handoff when relevant

### 5. Auth Secret Rule

`JWT_SECRET` must come from `.env`, not config.

Do not move JWT secrets into config examples or framework defaults.

## Core Change Playbooks

### 1. Fix Env / Config Compatibility

Use this flow:

1. confirm whether the bug is env read, env write, or config cache behavior
2. preserve fallback behavior across `getenv`, `$_ENV`, `$_SERVER`
3. do not assume `putenv()` exists
4. avoid introducing app-specific special cases into core
5. document deployment follow-up if config cache is involved

### 2. Fix Response / Request Base API

Use this flow:

1. verify the missing method or wrong return shape in actual runtime code
2. prefer adding compatible helper methods over changing existing call contracts
3. preserve existing method aliases when possible
4. keep JSON envelope consistent

Examples:

- adding `withHeaders()` to `Response`
- avoiding non-existent request helpers in docs or generated code
- keeping `Response::success()` and `Response::error()` behavior stable

### 3. Fix Route Loading / Generators

Use this flow:

1. inspect actual directory assumptions
2. compare them against PSR-4 namespace expectations
3. support uppercase-first resolution
4. retain compatibility for old lowercase structures if feasible

Examples:

- `Foundation/App.php` route loading
- `make:controller`
- `make:action`
- `make:provider`
- `make:middleware`
- `make:model`

### 4. Fix ORM / Query Builder

Preserve these known correctness rules:

- validate `join` operators
- align `insertAll()` by first-row keys
- preserve soft-delete scope in `update`, `delete`, `aggregate`, `exists`, `cursor`
- allow non-auto-increment / UUID primary keys to save correctly
- avoid treating zero-change updates as hard failures

When changing query behavior, think about silent regressions across all apps.

### 5. Fix HTTP Client

Rules:

- SSL verify should be resolved dynamically at request time
- `sslVerify(false)` must really disable peer and host verification
- client return shape should remain:
  - `status`
  - `headers`
  - `body`
  - `json`
- error behavior should remain predictable for callers

### 6. Fix Auth Manager

Rules:

- keep guard support intact
- keep cookie-based auth support intact
- keep refresh-token and session-tracking behavior intact
- avoid changing default token parsing order unless necessary
- never break `JWT_SECRET` env-only rule

## Effective Runtime Rule

When a project is currently running against vendored `anon-core`, remember:

- editing `framework/core` alone may not change the live app
- the effective runtime copy is often under `project/.../vendor/yuinijika/anon-core`
- if the user needs immediate runtime recovery, mirror the fix where the app actually executes

Always state which copy is authoritative for the current task.

## Review Checklist

Before finalizing a core change, check:

- does this really belong in core
- is the fix additive and backward-compatible
- could this break old projects silently
- does Linux case-sensitivity still work
- does restricted hosting without `putenv()` still work
- does current response/auth/config behavior remain stable
- if needed, was the runtime vendor copy also updated

## Core Templates

### 1. Uppercase-First Directory Resolver

```php
protected function resolveAppDirectory(array $candidates): string
{
    foreach ($candidates as $directory) {
        $path = APP_PATH . DIRECTORY_SEPARATOR . $directory;

        if (is_dir($path)) {
            return $path;
        }
    }

    return APP_PATH . DIRECTORY_SEPARATOR . $candidates[0];
}
```

### 2. Dynamic Config Read

```php
$sslVerify = $this->sslVerify ?? (bool) \Anon\Core\Facade\Config::get('http.ssl_verify', true);
```

### 3. Env Fallback Read

```php
if (isset($this->data[$key])) {
    return $this->data[$key];
}

if (function_exists('getenv')) {
    $systemEnv = getenv($key);
    if ($systemEnv !== false) {
        return $this->parseValue($systemEnv);
    }
}

if (isset($_ENV[$key])) {
    return $this->parseValue($_ENV[$key]);
}

if (isset($_SERVER[$key])) {
    return $this->parseValue($_SERVER[$key]);
}
```

### 4. Numeric Response Code

```php
$payload = [
    'success' => false,
    'code' => $statusCode,
    'message' => $message,
];
```

## Anti-Patterns

Avoid these in core:

- app-specific business logic in framework classes
- raw `getenv()` dependence in production-critical paths
- returning string codes in API envelopes
- forcing lowercase PSR-4 paths
- making generator output diverge from namespace conventions
- changing auth secret source away from `.env`
- fixing only `framework/core` when the actual runtime copy is vendor

## Handoff Requirements

When handing off a core change, include:

- why it is a framework issue
- which files were changed
- whether runtime vendor copy was also changed
- whether cache or autoload should be refreshed
- whether Linux case / opcache / php-fpm restart matters

## Example Triggers

Invoke this skill when the user asks things like:

- "这个问题是 framework 本身的问题，直接修 core"
- "Response 这里少了基础方法"
- "Env 在生产环境不兼容"
- "route loader / make 命令路径写死了"
- "ORM 这里是框架级 bug"
- "anon-core vendor 和 framework/core 一起修"
