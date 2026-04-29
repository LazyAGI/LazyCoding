# B. Coding Standards

> These rules apply continuously during coding. The AI review tool checks all of these items —
> following them upfront reduces review round-trips.

---

## B.1 Minimal Defensive Programming

**Core principle (最小防御原则): only add protection at real trust boundaries — not for
conditions already guaranteed by the caller or the type system. Every guard must justify its
existence with a concrete failure scenario.**

- Do not add `isinstance` checks, null guards, or broad `try...except Exception` blocks,
  especially for conditions already guaranteed by documentation.
- Exception catches must be specific: use `OSError`, `ValueError`, etc. Be aware of inheritance:
  `UnicodeDecodeError` is a subclass of `OSError` — do not list both (flake8 B014).
- When a contract is violated, fail fast with a clear error rather than silently swallowing
  the exception and continuing.
- Do not write dead code in `except` blocks (e.g. `if some_var:` when that variable is
  necessarily empty on the exception path).
- A silent `except: pass` must at minimum log `LOG.debug(...)` so the error is observable.

**Legitimate exceptions** — the following always require protection:
- User input (SQL injection, path traversal, XSS)
- External API responses (structure is untrusted)
- Concurrent shared state (race conditions)
- Resource lifecycle (files / connections / locks must be released on all paths)

**Examples:**

```python
# Bad — unnecessary guard; parse_config() is documented to return a dict
def load_settings(path):
    config = parse_config(path)
    if not isinstance(config, dict):
        return {}
    return config

# Good — trust the contract, fail fast if violated
def load_settings(path):
    return parse_config(path)
```

```python
# Bad — broad except swallows everything
try:
    result = call_api(url)
except Exception:
    pass

# Good — specific exception, with logging
try:
    result = call_api(url)
except ConnectionError:
    LOG.warning("API unreachable: %s", url)
    result = None
```

---

## B.2 Concurrency Safety

- Objects shared across threads (tool sets, caches, counters) must be constructed independently
  inside each thread's closure — do not share mutable state across threads.
- When using `concurrent.futures`, alias `TimeoutError` as `FuturesTimeoutError` to avoid
  confusion with the built-in `TimeoutError`.
- The `timeout` argument to `as_completed(futs, timeout=N)` should reflect the expected
  duration of a **single task** — do not multiply by the number of workers; workers run in
  parallel, not serially.
- Class-level shared data structures should be initialized at class definition time, not lazily,
  to avoid TOCTOU races.
- Keep lock granularity small; never perform I/O or LLM calls while holding a lock.

**Examples:**

```python
# Bad — shared mutable list across threads
results = []
def worker(item):
    results.append(process(item))  # race condition

# Good — each thread returns its result; aggregate after join
def worker(item):
    return process(item)

with ThreadPoolExecutor() as pool:
    results = list(pool.map(worker, items))
```

```python
# Bad — timeout multiplied by worker count
futs = [pool.submit(task, x) for x in batch]
for f in as_completed(futs, timeout=30 * len(batch)):  # wrong
    ...

# Good — timeout reflects single-task duration
for f in as_completed(futs, timeout=30):
    ...
```

---

## B.3 Function and Interface Design

- **Single responsibility**: each function does one thing. Extract helper functions when
  cyclomatic complexity exceeds the project limit.
- **Semantically distinct return values**: use a sentinel object (`_SKIP = object()`) to
  distinguish "skip this item" from "return an empty value" — do not use `return None` for both.
- **Avoid repeated calls**: do not call the same function twice in the same conditional
  (e.g. `ckpt.get('key') is not None` followed by `ckpt.get('key') or []`); cache to a variable.
- **Default arguments**: should reflect the most general call site, not be hardcoded to one
  specific caller's name.
- **Magic numbers**: extract as named constants with a comment explaining the design intent.
- **Interface changes**: when modifying a public function signature, update all call sites
  atomically — never leave the codebase in a state where only some callers are updated.

**Examples:**

```python
# Bad — repeated call to the same function
if ckpt.get('key') is not None:
    value = ckpt.get('key') or []  # called twice

# Good — cache to a variable
value = ckpt.get('key')
if value is not None:
    value = value or []
```

```python
# Bad — magic number without explanation
if retry_count > 3:
    raise RetryExhausted()

# Good — named constant with intent
MAX_RETRIES = 3  # balances latency vs. reliability for LLM calls
if retry_count > MAX_RETRIES:
    raise RetryExhausted()
```

---

## B.4 Naming and Semantics

- Method names should be self-explanatory and concise. Do not redundantly include the class
  name in the method name (`ClassA.get_classA_instance()` → `ClassA.get_instance()`).
- Sibling classes (same base class or same role) should use consistent parameter names for
  analogous methods.
- When overloading dunder methods (`__or__`, `__getitem__`, etc.), the semantics must match
  mainstream conventions (bash pipe `|`, Python slice `[]`); misleading semantics must be flagged.
- Variable names should reflect content, not type (`active_users` is better than `user_list`).

**Examples:**

```python
# Bad — class name repeated in method name
class DocumentParser:
    def get_document_parser_instance(cls): ...

# Good
class DocumentParser:
    @classmethod
    def get_instance(cls): ...
```

```python
# Bad — name reflects type, not content
user_list = get_active_users()
str_val = record.name

# Good — name reflects content
active_users = get_active_users()
username = record.name
```

---

## B.5 Code Cleanliness

- **Remove dead code**: functions / constants / mapping entries with no callers, branches
  commented "reserved for future use" with no actual usage.
- **Remove redundant exception types** (flake8 B014): do not list a subclass alongside its
  parent in the same `except` tuple.
- **Comments explain "why", not "what"**: the code itself should make the "what" obvious.
- **Avoid duplication**: the same logic appearing twice should be extracted. When extracting,
  do not over-abstract to the point of reducing readability.

**Examples:**

```python
# Bad — dead code left "for future use"
FEATURE_FLAGS = {
    'dark_mode': True,
    'beta_export': False,   # reserved for future use — no caller
}

# Good — remove until actually needed; version control preserves history
FEATURE_FLAGS = {
    'dark_mode': True,
}
```

```python
# Bad — comment explains "what"
x = x + 1  # increment x by 1

# Good — comment explains "why"
x = x + 1  # offset for 1-based line numbers in GitHub API
```

---

## B.6 Code Conciseness and Readability

- **Early return**: use early returns to reduce `if-else` nesting depth. When a condition is met,
  return immediately instead of wrapping the remaining logic in an `else` block.

  ```python
  # Bad — deep nesting
  def process(user):
      if user is not None:
          if user.is_active:
              if user.has_permission('edit'):
                  do_edit(user)

  # Good — early return
  def process(user):
      if user is None:
          return
      if not user.is_active:
          return
      if not user.has_permission('edit'):
          return
      do_edit(user)
  ```

- **Merge similar branches**: when two branches of an `if-else` share most of their logic,
  merge them into a single path.

  ```python
  # Before
  if a:
      xxx(a)
  else:
      xxx(geta())

  # After
  if not a:
      a = geta()
  xxx(a)
  ```

- **Prefer generators over lists**: when iterating over large data, use generator expressions
  instead of list comprehensions to reduce memory usage — unless the result needs to be
  indexed or reused.

  ```python
  # Bad — builds entire list in memory just to iterate
  total = sum([compute(x) for x in large_dataset])

  # Good — generator, no intermediate list
  total = sum(compute(x) for x in large_dataset)
  ```

- **Keep functions short**: aim for ≤20 lines per function. When a function grows beyond this,
  extract logical steps into well-named helper functions.

- **Minimize parameter count**: functions with more than 3-4 parameters should consider grouping
  related parameters into a dataclass, TypedDict, or config object.

  ```python
  # Bad — too many positional parameters
  def send_email(to, cc, bcc, subject, body, reply_to, priority):
      ...

  # Good — group into a config object
  @dataclass
  class EmailMessage:
      to: str
      subject: str
      body: str
      cc: str = ""
      bcc: str = ""
      reply_to: str = ""
      priority: str = "normal"

  def send_email(msg: EmailMessage):
      ...
  ```

- **Simplify conditional expressions**: prefer `if x:` over `if x is not None:` when truthiness
  is sufficient. Avoid verbose boolean comparisons like `if flag == True:`.

  ```python
  # Bad — verbose
  if name is not None and len(name) > 0:
      ...
  if flag == True:
      ...

  # Good — concise
  if name:
      ...
  if flag:
      ...
  ```

---

## B.7 Architecture and Design

- **Single abstraction principle**: if the system already has one implementation of a concept,
  adding a second requires introducing a shared base class / Protocol / ABC that both
  implementations conform to.
- **Justify every change**: every modification must be able to answer "why does this need to
  change here?" Avoid unnecessary refactoring, over-engineering, or changes that violate
  framework conventions.
- **Module coupling**: when adding a feature, assess whether it increases coupling between
  modules. Prefer interfaces / events / dependency injection over direct references.
- **Testability**: new code should be easy to test. Avoid global state, hardcoded dependencies,
  and external calls that cannot be mocked.
- **Performance**: do not sacrifice readability for performance without profiling data to justify
  it. Choose appropriate data structures (`set` for membership, `dict` for mapping,
  `deque` for queues).

**Examples:**

```python
# Bad — second implementation without shared abstraction
class FileExporter:
    def export(self, data): ...

class S3Exporter:
    def upload(self, data): ...  # different method name, no shared interface

# Good — shared Protocol
class Exporter(Protocol):
    def export(self, data) -> None: ...

class FileExporter:
    def export(self, data): ...

class S3Exporter:
    def export(self, data): ...
```

```python
# Bad — membership check on a list (O(n))
if user_id in [u.id for u in all_users]:
    ...

# Good — use a set (O(1))
active_ids = {u.id for u in all_users}
if user_id in active_ids:
    ...
```

---

## B.8 Error Handling and Observability

- Error messages must include enough context for the caller to locate the problem (file name,
  line number, argument values).
- Distinguish between expected errors (bad user input, network timeout — use `LOG.warning`)
  and programming errors (assertion failures, type errors — use `LOG.error` or `raise`).
- Do not swallow an exception at a low level and then print a vague message at a high level.
  Either handle and log at the point of failure, or propagate upward and let the caller decide.
- Critical operations (LLM calls, file writes, network requests) must have timeout protection
  with a clear log message on timeout.

**Examples:**

```python
# Bad — vague error message
raise ValueError("Invalid input")

# Good — actionable context
raise ValueError(
    f"Invalid chunk_size={chunk_size} for file '{path}': "
    f"must be between 1 and {MAX_CHUNK_SIZE}"
)
```

```python
# Bad — swallow at low level, vague message at high level
def fetch_data(url):
    try:
        return requests.get(url).json()
    except Exception:
        return None  # caller sees None with no idea why

# Good — log at point of failure, let caller decide
def fetch_data(url):
    try:
        resp = requests.get(url, timeout=10)
        resp.raise_for_status()
        return resp.json()
    except requests.Timeout:
        LOG.warning("Timeout fetching %s", url)
        raise
```

---

## B.9 Security

- **SQL injection**: use parameterized queries; never concatenate SQL strings.
- **Path traversal**: normalize user-supplied paths with `os.path.realpath` and verify they
  are within the allowed directory before use.
- **Hardcoded secrets**: keys, tokens, and passwords must not appear in code; use environment
  variables or config files.
- **Unsafe deserialization**: do not use `pickle`, `eval`, or `exec` on untrusted input.
- **Dependencies**: when adding a new dependency, check whether it is truly required; optional
  dependencies should be imported inside `try/except ImportError`.

**Examples:**

```python
# Bad — SQL injection via string concatenation
query = f"SELECT * FROM users WHERE name = '{username}'"
cursor.execute(query)

# Good — parameterized query
cursor.execute("SELECT * FROM users WHERE name = %s", (username,))
```

```python
# Bad — path traversal: user controls the path
def read_doc(filename):
    return open(f"/data/docs/{filename}").read()  # ../../etc/passwd

# Good — normalize and verify
def read_doc(filename):
    base = "/data/docs"
    full = os.path.realpath(os.path.join(base, filename))
    if not full.startswith(base):
        raise ValueError(f"Path traversal blocked: {filename}")
    return open(full).read()
```
