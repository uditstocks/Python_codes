# Python Interview Questions & Answers

A focused study guide with three sections — **Google**, **Microsoft**, and **Infosys** — 10 questions each. The questions are grouped to reflect what each company typically emphasises: Google leans on internals, complexity, and algorithms; Microsoft on OOP, design, and language mechanics; Infosys on fundamentals and clear conceptual understanding.

> Tip: In real interviews, *say the trade-off out loud*. Stating complexity, edge cases, and "why this approach" matters as much as the code itself.

---

## Section 1 — Google (Internals, Complexity & Algorithms)

### 1. Explain the GIL. How does it affect threading vs multiprocessing?
The **Global Interpreter Lock** is a mutex in CPython that allows only one thread to execute Python bytecode at a time. Consequences:

- **CPU-bound work** sees no speed-up from threads — they can't run bytecode in parallel. Use `multiprocessing` (each process has its own interpreter and GIL) or a C extension like NumPy that releases the GIL.
- **I/O-bound work** *does* benefit from threads, because the GIL is released during blocking I/O (network, disk).

Mention that the free-threaded build (PEP 703, experimental from Python 3.13) makes the GIL optional — a strong signal you follow the language's evolution.

### 2. What are the time/space complexities of `list`, `dict`, and `set` operations?
| Operation | list | dict | set |
|---|---|---|---|
| Index / key access | O(1) | O(1) avg | — |
| `append` | O(1) amortised | — | — |
| Insert/delete at position | O(n) | O(1) avg | O(1) avg |
| Membership (`in`) | O(n) | O(1) avg | O(1) avg |

Be ready to explain *amortised* O(1) for `append`: the underlying dynamic array doubles capacity occasionally, so the cost averages out to constant time. `dict`/`set` worst case is O(n) under pathological hash collisions.

### 3. Find the *k* largest elements in a large stream of numbers.
Use a **min-heap of size k** (`heapq`):

```python
import heapq

def k_largest(stream, k):
    heap = []
    for x in stream:
        heapq.heappush(heap, x)
        if len(heap) > k:
            heapq.heappop(heap)   # drop the smallest
    return sorted(heap, reverse=True)
```

`O(n log k)` time, `O(k)` space — far better than sorting everything (`O(n log n)`) when `k` is small. In practice you'd just call `heapq.nlargest(k, stream)`.

### 4. Explain generators. How do `yield` and `return` differ?
A generator produces values **lazily**, pausing at each `yield` and resuming on the next request, keeping its state in between. `yield` suspends and yields a value; `return` ends the generator (raising `StopIteration`). Generators shine for streaming large files, building pipelines, and infinite sequences because they hold only one item in memory at a time.

```python
def read_lines(path):
    with open(path) as f:
        for line in f:          # one line in memory at a time
            yield line.strip()
```

### 5. Difference between `is` and `==` (and the integer-caching gotcha).
`==` compares **values**; `is` compares **identity** (same object in memory). CPython caches small integers (−5 to 256) and interns some strings, so:

```python
a = 256; b = 256
a is b      # True  (cached)
a = 257; b = 257
a is b      # False (separate objects) — but a == b is True
```

Rule: use `==` for value comparison; reserve `is` for singletons like `None`, `True`, `False`.

### 6. Implement an LRU cache.
Production answer: `functools.lru_cache`. But interviewers want the mechanics — `OrderedDict` gives O(1) get/put:

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity):
        self.cache = OrderedDict()
        self.capacity = capacity

    def get(self, key):
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)      # mark as recently used
        return self.cache[key]

    def put(self, key, value):
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)  # evict least recently used
```

### 7. What are decorators? Write one that times a function.
A decorator wraps a function to extend its behaviour without changing its body — built on closures. Use `functools.wraps` to preserve the wrapped function's name/docstring.

```python
import time, functools

def timed(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = fn(*args, **kwargs)
        print(f"{fn.__name__} took {time.perf_counter() - start:.4f}s")
        return result
    return wrapper
```

### 8. How does Python manage memory?
Two mechanisms working together:

1. **Reference counting** (primary): every object tracks how many references point to it; it's freed when the count hits zero.
2. **Cyclic garbage collector** (`gc` module): reference counting can't free reference *cycles* (A → B → A), so a generational GC periodically detects and collects them.

Mention that `__del__` finalisers and cycles interact subtly, and that memory for small objects is managed via CPython's pymalloc allocator.

### 9. Mutable vs immutable types — and the mutable-default-argument bug.
**Immutable:** `int`, `float`, `str`, `tuple`, `frozenset`, `bytes`. **Mutable:** `list`, `dict`, `set`. The classic bug: default arguments are evaluated **once**, at function definition, so a mutable default is shared across calls.

```python
def bad(item, bucket=[]):     # bucket created once!
    bucket.append(item)
    return bucket

bad(1); bad(2)   # -> [1, 2], not [2]

def good(item, bucket=None):  # the fix
    bucket = [] if bucket is None else bucket
    bucket.append(item)
    return bucket
```

### 10. Detect a cycle in a linked list.
**Floyd's tortoise-and-hare**: advance one pointer by 1 and another by 2; if they ever meet, there's a cycle. `O(n)` time, `O(1)` space.

```python
def has_cycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow is fast:
            return True
    return False
```

---

## Section 2 — Microsoft (OOP, Design & Language Mechanics)

### 1. Explain the four pillars of OOP with Python examples.
- **Encapsulation** — bundling data with methods; Python signals "private" with a leading underscore and uses name-mangling (`__x` → `_Class__x`), though nothing is truly private ("we're all adults").
- **Abstraction** — hiding implementation behind a clean interface (e.g. abstract base classes).
- **Inheritance** — a subclass reuses/extends a base class.
- **Polymorphism** — the same call works on different types (e.g. `len()` on a list or a string; overriding methods).

### 2. `@staticmethod` vs `@classmethod` vs instance method.
- **Instance method** — takes `self`; operates on an instance.
- **Classmethod** — takes `cls`; great for **alternative constructors / factories**.
- **Staticmethod** — takes neither; a utility logically grouped under the class.

```python
class Date:
    def __init__(self, y, m, d):
        self.y, self.m, self.d = y, m, d

    @classmethod
    def from_string(cls, s):          # factory
        y, m, d = map(int, s.split("-"))
        return cls(y, m, d)

    @staticmethod
    def is_leap(year):                # utility
        return year % 4 == 0 and (year % 100 != 0 or year % 400 == 0)
```

### 3. Explain MRO and the diamond problem.
Python resolves method lookup using **C3 linearisation**, exposed via `Cls.__mro__`. `super()` follows this order, enabling cooperative multiple inheritance and resolving the "diamond" (D inherits from B and C, both from A) deterministically — A's method runs once, in a well-defined order.

```python
class A: ...
class B(A): ...
class C(A): ...
class D(B, C): ...
print(D.__mro__)   # D -> B -> C -> A -> object
```

### 4. What are `__slots__` and when do you use them?
By default each instance stores attributes in a per-instance `__dict__`. Declaring `__slots__` replaces that dict with a fixed, array-like layout — **less memory and faster attribute access** — at the cost of dynamic attribute assignment. Worthwhile when you create millions of small objects.

```python
class Point:
    __slots__ = ("x", "y")
    def __init__(self, x, y):
        self.x, self.y = x, y
```

### 5. Explain context managers and the `with` statement.
A context manager guarantees setup/teardown via the `__enter__`/`__exit__` protocol — ideal for files, locks, and connections. Two ways to write one:

```python
# Class-based
class Resource:
    def __enter__(self):
        print("open"); return self
    def __exit__(self, exc_type, exc, tb):
        print("close")   # runs even if an exception is raised

# Generator-based
from contextlib import contextmanager
@contextmanager
def resource():
    print("open")
    try:
        yield "handle"
    finally:
        print("close")
```

### 6. Deep copy vs shallow copy.
A **shallow** copy duplicates the outer container but shares the nested objects; a **deep** copy recursively duplicates everything. The gotcha:

```python
import copy
original = [[1, 2], [3, 4]]
shallow = copy.copy(original)
shallow[0].append(99)        # also mutates original[0]!
deep = copy.deepcopy(original)  # fully independent
```

### 7. Explain `*args` and `**kwargs`.
`*args` collects extra **positional** arguments into a tuple; `**kwargs` collects extra **keyword** arguments into a dict. They also work for *unpacking* when calling. Argument order is: positional, `*args`, keyword-only, `**kwargs`.

```python
def f(a, *args, **kwargs):
    print(a, args, kwargs)

f(1, 2, 3, x=10)   # 1 (2, 3) {'x': 10}
nums = [1, 2, 3]
f(*nums, x=10)     # unpacking
```

### 8. Data classes vs named tuples.
- `@dataclass` auto-generates `__init__`, `__repr__`, `__eq__`, etc.; **mutable by default**, can be `frozen=True`; supports defaults and type hints.
- `namedtuple` is an **immutable**, tuple-based, lightweight record.

Use a dataclass when you need mutability, methods, or many fields; a namedtuple when you want a tiny immutable record that behaves like a tuple. (Mention `pydantic`/`attrs` for validation-heavy cases.)

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Point:
    x: int
    y: int = 0
```

### 9. Exception handling: `try/except/else/finally` and custom exceptions.
- `try` runs the risky code; `except` handles specific errors; `else` runs **only if no exception** occurred; `finally` **always** runs (cleanup).
- Custom exceptions subclass `Exception`. Use `raise ... from ...` to preserve the original cause (exception chaining).

```python
class ValidationError(Exception):
    pass

try:
    risky()
except ValueError as e:
    raise ValidationError("bad input") from e
else:
    print("succeeded")
finally:
    cleanup()
```

### 10. What is duck typing, and how do ABCs / Protocols relate?
**Duck typing** — "if it walks like a duck and quacks like a duck, it's a duck": Python cares about an object's *behaviour*, not its declared type. **Abstract Base Classes** (`abc`) let you *enforce* an interface at runtime, and **Protocols** (PEP 544, `typing.Protocol`) give *static* structural typing — type checkers verify an object has the right methods without explicit inheritance.

---

## Section 3 — Infosys (Fundamentals & Core Concepts)

### 1. What is the difference between a list and a tuple?
| | list | tuple |
|---|---|---|
| Mutable | Yes | No |
| Syntax | `[1, 2]` | `(1, 2)` |
| Hashable (usable as dict key) | No | Yes (if elements are) |
| Performance | Slightly slower | Slightly faster, less memory |

Use a list when contents change; a tuple for fixed records or when you need a hashable key.

### 2. What are mutable and immutable types? Give examples.
- **Immutable** (can't change after creation): `int`, `float`, `bool`, `str`, `tuple`, `frozenset`, `bytes`.
- **Mutable** (can change in place): `list`, `dict`, `set`, `bytearray`.

Reassigning a variable creates a new object; *mutating* changes the existing one. This matters when objects are passed to functions or shared.

### 3. What are the key features of Python? Is it compiled or interpreted?
Python is **high-level, dynamically typed, interpreted** (compiled to bytecode, then run by the CPython virtual machine), and multi-paradigm (procedural, OOP, functional). It's prized for **readability**, a **huge standard library + ecosystem**, automatic memory management, and portability across platforms.

### 4. What is PEP 8?
PEP 8 is Python's official **style guide**: 4-space indentation, `snake_case` for functions/variables, `CapWords` for classes, ~79-char line length, and import ordering. Tools that enforce it: `black` (auto-formatter), `flake8`/`pylint` (linters). Following it makes code consistent and reviewable.

### 5. Explain variable scope (LEGB) and the `global`/`nonlocal` keywords.
Python resolves names using **LEGB**: **L**ocal → **E**nclosing → **G**lobal → **B**uilt-in. `global x` lets a function rebind a module-level variable; `nonlocal x` lets a nested function rebind a variable in the enclosing (non-global) scope.

```python
def counter():
    count = 0
    def increment():
        nonlocal count
        count += 1
        return count
    return increment
```

### 6. What are list and dictionary comprehensions?
Concise, readable ways to build collections in one expression:

```python
squares = [n * n for n in range(5)]                 # [0, 1, 4, 9, 16]
evens   = [n for n in range(10) if n % 2 == 0]      # with a condition
squares_map = {n: n * n for n in range(5)}          # dict comprehension
```

They're usually faster and clearer than an equivalent `for` loop — but keep them readable (avoid deeply nested ones).

### 7. Difference between `append()` and `extend()`.
- `append(x)` adds `x` as a **single element** (if `x` is a list, it's nested).
- `extend(iterable)` iterates over the argument and adds **each element**.

```python
a = [1, 2]
a.append([3, 4])   # [1, 2, [3, 4]]
b = [1, 2]
b.extend([3, 4])   # [1, 2, 3, 4]
```

### 8. How do you handle exceptions in Python?
Wrap risky code in `try`/`except`, catching specific exceptions; add `finally` for cleanup that must always run. Catch precise exception types rather than a bare `except:`.

```python
try:
    value = int(user_input)
except ValueError:
    print("Please enter a valid number")
except ZeroDivisionError:
    print("Cannot divide by zero")
finally:
    print("Done")
```

### 9. What is the difference between a module and a package? Explain `if __name__ == "__main__"`.
- A **module** is a single `.py` file you can `import`.
- A **package** is a directory of modules (traditionally containing an `__init__.py`).

`if __name__ == "__main__":` ensures the block runs only when the file is executed directly, not when it's imported — useful for putting test/CLI code in a module without it firing on import.

### 10. What are lambda functions, and how do `map`, `filter`, and `reduce` work?
A `lambda` is a small **anonymous** function written inline, limited to a single expression.

```python
square = lambda x: x * x

nums = [1, 2, 3, 4]
list(map(lambda x: x * 2, nums))         # [2, 4, 6, 8]
list(filter(lambda x: x % 2 == 0, nums)) # [2, 4]

from functools import reduce
reduce(lambda a, b: a + b, nums)         # 10 (sum)
```

Use lambdas for short throwaway functions (e.g. a `key=` argument in `sorted`); prefer a named `def` when logic gets complex.

---

### How to use this guide
- Practise saying answers **out loud** — clarity of explanation is scored heavily.
- For coding questions, always state **time and space complexity** and discuss **edge cases**.
- Tie answers to real experience where you can ("I used a generator to stream a 10 GB log file…").
- Know *why*, not just *what* — interviewers probe follow-ups.

Good luck!
