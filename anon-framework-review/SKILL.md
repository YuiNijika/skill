---
name: "anon-framework-review"
description: "Reviews Anon Framework Next apps and core changes for bugs, regressions, Linux compatibility, env/config mistakes, and response contract drift. Invoke when auditing Anon code or docs."
---

# Anon Framework Review

Use this skill when reviewing Anon app code, framework/core changes, runtime vendor patches, or even VitePress docs for correctness and compatibility.

The goal is not style policing. The goal is to catch bugs, regressions, misleading examples, production incompatibilities, and framework-contract drift.

## Review Priority

Check these in order:

1. correctness
2. production compatibility
3. framework contract consistency
4. deployment/cache follow-up
5. docs/example accuracy

## Highest-Value Findings

Always look for these first:

- Linux path case mismatches under `app/Controller`, `app/Service`, `app/Route`, `app/Action`
- direct `getenv()` use in business/runtime-sensitive code
- config changes that will be hidden by `runtime/cache/config.php`
- route changes hidden by `runtime/cache/routes.php`
- response payloads that regress `code` back to string values
- non-existent framework methods used in examples or code
- changes made only in `framework/core` while runtime actually uses vendored `anon-core`
- auth secret misuse such as moving `JWT_SECRET` into config

## App Review Checklist

For `app/**` changes, check:

- route lives in uppercase `app/Route`
- controller stays thin
- service owns third-party IO and error normalization
- request validation is done via `FormRequest` or `Validator`
- response uses `Response::success()` / `Response::error()`
- `code` remains numeric HTTP status code
- business failures use `error_code`
- `Env::get()` / `Config::get()` are used appropriately

## Core Review Checklist

For `framework/core/**` or `vendor/yuinijika/anon-core/**` changes, check:

- does this truly belong to framework core
- is the change backward-compatible
- does it preserve Linux-safe PSR-4 behavior
- does it preserve env fallback behavior
- does it preserve auth and response contracts
- does the live runtime copy also need the same patch

## Docs Review Checklist

When reviewing VitePress docs, check for these stale patterns:

- `app/controller` instead of `app/Controller`
- `app/route` instead of `app/Route`
- `app/action` instead of `app/Action`
- `app/provider` instead of `app/Provider`
- `"code": "OK"` instead of numeric status code
- direct `getenv()` examples without clarifying `Env::get()`
- old namespace samples like `App\Providers` when the current skeleton uses `Anon\Provider`
- examples using APIs that do not exist in current runtime

Docs should reflect what actually works in production, not just what happened to work on Windows or in older builds.

## Review Playbooks

### 1. 500 Error Review

Look for:

- missing method in runtime class
- wrong namespace/path case
- vendor/core mismatch
- stale cache
- docs-driven wrong API usage

### 2. Local OK / Online Fail Review

Look for:

- path case
- `putenv()` restrictions
- env override by system variables
- stale autoload
- stale config/route cache
- opcache / php-fpm stale runtime

### 3. Third-Party Integration Review

Look for:

- secrets leaking in responses
- upstream error handling living in controller instead of service
- SSL verify behavior not reading current config
- cookie or auth claims that are impossible cross-origin

### 4. Response Contract Review

Look for:

- numeric `code`
- stable `message`
- `error_code` on business failures
- no arbitrary payload drift across controllers

## Output Style

When reporting findings:

- list findings first
- order by severity
- cite exact file and reason
- mention deployment/runtime risk where relevant
- keep summary short

If no findings are present, explicitly say so and note any residual risk or testing gap.

## Example Triggers

Invoke this skill when the user asks things like:

- "帮我审一下 Anon 项目"
- "看看这次改动有没有坑"
- "Review 一下 framework/core"
- "检查文档有没有写错"
- "线上为什么和本地不一样，帮我审一下"
