---
name: "yuinijika-coding-style"
description: "YuiNijika's coding conventions across PHP, C++, TypeScript, Python, and Rust, aligned with Anon Framework coding standards. Invoke when generating or reviewing code in YuiNijika's style."
---

# Coding Style — YuiNijika

Aligned with the [Anon Framework Coding Standards](https://anon.miomoe.cn/guide/coding-standards.html) and real-world development experience. Language-agnostic core principles, with per-language conventions following each ecosystem's community norms.

---

## Universal Principles

These apply across all languages, but when contributing to an existing project, **match the project's conventions first** — these rules are the fallback.

### Formatting

- Indentation: **4 spaces**, tabs forbidden
- Line endings: **LF (Unix)**, CRLF forbidden
- Encoding: **UTF-8 without BOM**
- One trailing newline at end of file

### Scripting & Automation

For shell tasks and automation scripts, prefer **Python** over PowerShell (`.ps1`). PowerShell has chronic encoding issues and garbage syntax — every terminal command tends to take multiple retries, burning tokens for no reason. Use Python for any script that needs to run reliably on the first try. Avoid running PowerShell commands in the terminal unless they serve a clear, necessary purpose — if Python can do it, use Python.

### After Making Changes

Do **not** run dev/build commands or package manager installs after editing code. The development environment is already set up — these commands waste tokens and are meaningless busywork. Make the code change, verify with linter if needed, and stop.

### Prefer Direct Actions Over Scripts

For routine tasks (file manipulation, search, text processing), do it yourself rather than spawning a script. Running a script for minor work risks introducing errors that then cost more tokens to debug and fix. If the model can read, edit, or search directly — that's always the first choice. Scripts are for bulk automation that genuinely can't be done by hand, not for convenience.

### UI Text & Page Content

User-facing text — page banners, taglines, descriptions — should only describe what the user sees and experiences. Write for the actual scenario, not for explaining how it was built.

### UI Component Libraries

- Use the library's CLI to add components, never write them by hand.
- Never edit files inside the library's directory.

### Naming Quick Reference

| Element | PHP | C++ | TypeScript | Python | Rust |
|---|---|---|---|---|---|
| Class / Interface / Struct | `PascalCase` | `PascalCase` | `PascalCase` | `PascalCase` | `PascalCase` |
| Method / Function | `camelCase` | `camelCase` | `camelCase` | `snake_case` | `snake_case` |
| Variable / Property | `camelCase` | `snake_case` | `camelCase` | `snake_case` | `snake_case` |
| Constant | `UPPER_SNAKE` | `UPPER_SNAKE` | `UPPER_SNAKE` | `UPPER_SNAKE` | `UPPER_SNAKE` |
| File name | `PascalCase` | `snake_case` | `kebab-case` | `snake_case` | `snake_case` |
| Boolean prefix | `is`/`has`/`should` | `is_`/`has_`/`should_` | `is`/`has`/`should` | `is_`/`has_`/`should_` | `is_`/`has_`/`should_` |

Python and Rust use `snake_case` for methods/variables per strong community convention — do not force `camelCase` there.

### Directory-Scoped Naming

When a directory already scopes the content, do not repeat the directory name in the file or class. The directory is the namespace.

```
// Good — directory provides scope, file name is clean
Test/AI              → class AI
components/Button    → export Button
models/User          → class User

// Bad — redundant prefix
Test/TestAI          → class TestAI
components/ButtonComponent → export ButtonComponent
models/UserModel     → class UserModel
```

When importing, alias as needed for clarity at the call site: `import { AI as TestAI } from './Test/AI'`.

### Type Declarations

All languages prefer explicit types:

- **PHP**: properties, parameters, and return types must be declared. `strict_types=1`. Union types `int|string`, nullable `?string`.
- **C++**: prefer `auto` with explicit initialization. Parameters and return values use concrete types, not `void*`.
- **TypeScript**: `strict: true`. Ban `any` unless there is a documented reason.
- **Python**: type hints (PEP 484) on all public functions. Run mypy.
- **Rust**: type-enforced by the compiler. Use `Option<T>` and `Result<T, E>`, not raw `unwrap()`.

### Early Return

Universal. Handle edge cases and errors first, keep the main logic at the outermost indentation level.

```php
// Good
if (!$user) {
    throw new Exception('User not found');
}
// Main logic...

// Bad
if ($user) {
    if ($user->isActive) {
        // Main logic...
    }
}
```

### No Magic Numbers

Named constants or enums only:

```php
class User { public const STATUS_BANNED = 2; }
```

```cpp
enum class UserStatus : uint8_t { Banned = 2 };
```

```typescript
enum UserStatus { BANNED = 2 }
```

```python
class UserStatus(IntEnum): BANNED = 2
```

```rust
enum UserStatus { Banned = 2 }
```

### Single Responsibility (SRP)

One function does one thing. If it is getting long, split it into private/protected helpers. Applies to all languages equally.

---

## Comment Style

Core philosophy: **code documents "what", comments document "why"**.

### Do Not Write These

```php
// If status is 1
if ($status === 1) { ... }

// Increment counter
$count++;

// Loop through items
for (...) { ... }

// Return result
return $result;
```

The code already says this. These comments are noise.

### Write These Instead

```php
// Production may disable putenv(), fall back to $_ENV
if (function_exists('getenv')) { ... }

// Onion-model exception passthrough — must throw to outer Handler
catch (\Throwable $e) { throw $e; }

// Prevent CPU-burning infinite loop, sleep 3s on failure
sleep(3);
```

Same principle in every language:

```cpp
// /proc/self/exe is more reliable than argv[0] on Linux
auto path = std::filesystem::read_symlink("/proc/self/exe");
```

```typescript
// Chrome's 5-minute WebSocket idle timeout is undocumented, discovered empirically
const HEARTBEAT_INTERVAL = 4.5 * 60 * 1000;
```

```python
# requests timeout is (connect, read) tuple; single value applies to both
response = requests.get(url, timeout=(5, 30))
```

### DocBlock / Documentation Comments

Write these only when:
- The method has complex business logic or edge cases
- Parameters are `mixed` / `array` / `object` and need structural explanation
- The method throws specific exceptions
- It is a public API / library interface

```php
/**
 * Pop and execute a job from the queue (blocking mode).
 *
 * @throws Exception When Redis extension is not loaded
 */
public function pop(?string $queue = null, int $timeout = 3): ?Job { ... }
```

Python uses docstrings, TypeScript uses JSDoc, Rust uses `///`.

---

## PHP & Anon Framework

### Framework Conventions

- Use `declare(strict_types=1);` at the top of every new file
- Pure PHP files omit the closing `?>`
- Directory-semantic naming: `Auth/Manager.php`, not `Auth/AuthManager.php`
- View / template files: `kebab-case.php` (e.g. `error-500.php`)
- PSR-4 uppercase directories for Linux compatibility: `app/Controller`, `app/Service`, `app/Route`

### Env & Config

```php
// Forbidden
$val = getenv('KEY');

// Correct
$val = trim((string) Env::get('KEY', ''));
```

`JWT_SECRET` must live in `.env` only — never in config files.

### Response Contract

```json
{ "success": true,  "code": 200, "message": "OK",      "data": {} }
{ "success": false, "code": 400, "message": "...",     "error_code": "BAD_REQUEST" }
```

`code` is **always** a numeric HTTP status code. **Never** a string.

### Controller / Service Split

Controller is thin: receive params → call Service → return Response.
Service owns business logic and third-party IO.

### Production Checklist

- `composer dump-autoload`
- Clear config / route cache
- Restart php-fpm / reload opcache
- Verify Linux directory casing matches namespace
- Verify cookie domain / secure / httponly / samesite

---

## C++

### Style

Follow the project's existing style. Prefer C++17/20 features:

```cpp
// Prefer structured bindings
auto [name, age] = getPerson();

// Prefer range-for
for (const auto& item : items) { ... }

// Prefer smart pointers
auto ptr = std::make_unique<Foo>();

// Prefer enum class
enum class State : uint8_t { Idle, Running, Done };
```

### Headers

- Use `#pragma once`
- Include order: own header → project headers → third-party → standard library
- No `using namespace std;` in headers

### Error Handling

- Without exceptions: use `std::optional` / `std::expected` (C++23) or `std::variant`
- With exceptions: throw by value, catch by const reference

---

## TypeScript

### Configuration

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true
  }
}
```

### Style

- Prefer `interface` for object shapes, `type` for unions / intersections
- Functions with 3+ parameters: use object parameters with destructuring
- Async: `Promise<T>` + `async/await`, never mix with callbacks
- File naming: `kebab-case.ts` for modules, `PascalCase.tsx` for React components

```typescript
interface FetchOptions {
  url: string;
  method?: 'GET' | 'POST';
  timeout?: number;  // ms, default 5000
}

async function fetchData({ url, method = 'GET', timeout = 5000 }: FetchOptions): Promise<Data> { ... }
```

---

## Python

### Style

PEP 8 compliant. Format with Ruff or Black.

```python
def fetch_user(user_id: int, *, include_deleted: bool = False) -> User | None:
    """Fetch a user by ID, excluding deleted by default.

    Args:
        user_id: User ID
        include_deleted: Whether to include deleted users

    Returns:
        User object or None
    """
    ...
```

- Type hints: all public functions must have them
- Strings: prefer f-strings
- Paths: prefer `pathlib.Path` over `os.path`
- Dependencies: `requirements.txt` or `pyproject.toml`

---

## Rust

### Style

`rustfmt` defaults, `clippy` zero warnings.

```rust
/// Load configuration from file, returning defaults on failure.
pub fn load_config(path: &Path) -> Result<Config, ConfigError> {
    let content = std::fs::read_to_string(path)?;
    toml::from_str(&content).map_err(ConfigError::from)
}
```

- Prefer `&str` / `&[T]` parameters, avoid unnecessary `.clone()`
- Error handling: `thiserror` for libraries, `anyhow` for applications
- `unwrap()` / `expect()` only where failure is logically impossible, with a comment explaining why
- Module file naming: `snake_case.rs`