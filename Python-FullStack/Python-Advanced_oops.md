# 🎓 ADVANCED PYTHON OOP — THE COMPLETE DEEP GUIDE
### Every Concept, Every Counter-Question, Every Detail

---

# 📌 MENTAL MODEL BEFORE STARTING

Every concept has ONE job:
- **Class** → Blueprint
- **Object** → Thing built from blueprint
- **Encapsulation** → Hide messy internals
- **Inheritance** → Don't repeat yourself
- **Polymorphism** → Same action, different behavior
- **Abstraction** → Show only what's needed
- **Closure** → Function that remembers its environment
- **Decorator** → Wrapper that adds behavior
- **Metaclass** → Class that controls how classes are created

---

# 📖 MODULE 1: CLOSURES — The Foundation of Everything Advanced

---

## 🧠 What is a Closure — Build it From Scratch

Before decorators, before functools, before anything advanced — you must deeply understand closures. Everything else builds on this.

```python
def make_counter():
    count = 0              # lives in make_counter's scope

    def increment():       # inner function
        nonlocal count     # use count from outer scope
        count += 1
        return count

    return increment       # return function, not result

counter1 = make_counter()
counter2 = make_counter()  # completely independent

print(counter1())    # 1
print(counter1())    # 2
print(counter1())    # 3
print(counter2())    # 1  ← own separate count
print(counter1())    # 4  ← unaffected
```

**What just happened?**
- `make_counter` ran and finished
- But `count` didn't die — it was "captured" by `increment`
- `counter1` IS the `increment` function, carrying `count=0` in its memory
- This captured environment is called the **closure**

---

## 🔬 The Three Conditions for a Closure

For a closure to exist, you need ALL THREE:

```
1. A nested function (function inside function)
2. The inner function references a variable from outer scope
3. The outer function RETURNS the inner function
```

```python
# Condition check:
def outer():
    x = 10                  # ✅ outer scope variable

    def inner():
        print(x)            # ✅ inner references outer variable

    return inner            # ✅ outer returns inner

fn = outer()
fn()    # 10 — x is still alive inside fn's closure
```

---

## 🧪 Inspecting Closures — What Python Stores

```python
def make_adder(n):
    def add(x):
        return x + n        # n is captured
    return add

add5 = make_adder(5)

# Inspect the closure
print(add5.__closure__)                          # (<cell at 0x...>,)
print(add5.__closure__[0].cell_contents)         # 5  ← the captured value
print(add5.__code__.co_freevars)                 # ('n',) ← captured variable names
```

---

## ⚠️ The Classic Closure Trap — Loop Variable Capture

```python
# WRONG — all functions capture the SAME variable i
functions = []
for i in range(5):
    def fn():
        return i           # captures i by reference, not by value
    functions.append(fn)

print([f() for f in functions])   # [4, 4, 4, 4, 4] ← all see final i=4!

# RIGHT — capture by value using default argument
functions = []
for i in range(5):
    def fn(x=i):           # x=i captures current value of i
        return x
    functions.append(fn)

print([f() for f in functions])   # [0, 1, 2, 3, 4] ✅

# ALSO RIGHT — use a closure factory
functions = []
for i in range(5):
    def make_fn(x):
        def fn():
            return x       # x is a new variable each call
        return fn
    functions.append(make_fn(i))

print([f() for f in functions])   # [0, 1, 2, 3, 4] ✅
```

> **Interview answer for "What is a closure?"**
>
> *"A closure is a function that remembers the variables from its enclosing scope even after that scope has finished executing. Three things are needed: a nested function, the inner function referencing an outer variable, and the outer function returning the inner one. Python stores captured variables in the function's `__closure__` attribute. I use closures when I need stateful functions without creating a full class — like counters, adders, or configuration factories. A classic gotcha is loop variable capture — all closures in a loop capture the same variable by reference, so they all see the final value."*

---

## 🏭 Real-World Closure Use Cases

```python
# 1. Configuration factory
def make_api_caller(base_url, api_key):
    def call(endpoint, params=None):
        url = f"{base_url}/{endpoint}"
        headers = {"Authorization": f"Bearer {api_key}"}
        return f"Calling {url} with {headers}"
    return call

deloitte_api = make_api_caller("https://api.deloitte.com", "secret_key")
print(deloitte_api("users"))
print(deloitte_api("reports"))

# 2. Validator factory
def make_validator(min_val, max_val):
    def validate(value):
        return min_val <= value <= max_val
    return validate

age_validator = make_validator(18, 100)
salary_validator = make_validator(0, 10000000)

print(age_validator(25))       # True
print(age_validator(15))       # False
print(salary_validator(50000)) # True
```

---

## `nonlocal` vs `global` — The Difference

```python
x = "global"

def outer():
    x = "outer"

    def inner():
        nonlocal x          # refers to outer's x, NOT global x
        x = "modified by inner"
        print(x)

    inner()
    print(x)   # modified by inner

outer()
print(x)       # global — unchanged

# global keyword
count = 0
def increment():
    global count            # refers to module-level count
    count += 1

increment()
print(count)   # 1
```

> *"`nonlocal` goes one scope up — to the immediately enclosing function. `global` goes all the way to module scope. In clean code I avoid both — `nonlocal` can be justified in closures, but `global` is almost always a design smell."*

---

# 📖 MODULE 2: DECORATORS — Complete Deep Dive

---

## 🧠 Why Decorators Exist — The Problem They Solve

```python
# Scenario: You have 10 API functions and need to add logging to all of them

def get_users():
    print("LOG: get_users called")      # repeated in every function
    return ["Raj", "Priya"]

def get_orders():
    print("LOG: get_orders called")     # same log line, duplicated
    return [101, 102, 103]

def get_reports():
    print("LOG: get_reports called")    # and again...
    return ["Q1", "Q2"]

# This violates DRY. What if log format changes? Edit 10 functions.
```

```python
# WITH decorators — one place, applied everywhere

def log(func):
    def wrapper(*args, **kwargs):
        print(f"LOG: {func.__name__} called")
        return func(*args, **kwargs)
    return wrapper

@log
def get_users(): return ["Raj", "Priya"]

@log
def get_orders(): return [101, 102, 103]

@log
def get_reports(): return ["Q1", "Q2"]
# Change log format once — all 10 functions updated instantly
```

---

## 🔬 Decorator Anatomy — Every Line Explained

```python
import functools

def my_decorator(func):          # 1. takes a function as argument
    
    @functools.wraps(func)       # 2. preserves original function metadata
    def wrapper(*args, **kwargs):# 3. *args, **kwargs = accept any arguments
        
        # 4. Code runs BEFORE the original function
        print("Before")
        
        result = func(*args, **kwargs)  # 5. call the ORIGINAL function
        
        # 6. Code runs AFTER
        print("After")
        
        return result            # 7. return original result
    
    return wrapper               # 8. return the wrapper (not its result!)
```

**Why `@functools.wraps(func)`?**

```python
def bad_decorator(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

def good_decorator(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@bad_decorator
def greet(name):
    """Says hello to someone"""
    return f"Hello {name}"

@good_decorator
def greet2(name):
    """Says hello to someone"""
    return f"Hello {name}"

print(greet.__name__)    # wrapper     ← WRONG, lost original name
print(greet.__doc__)     # None        ← WRONG, lost docstring
print(greet2.__name__)   # greet2     ← correct
print(greet2.__doc__)    # Says hello to someone ← correct
```

> *"I always use `@functools.wraps` in decorators. Without it, the wrapper function replaces the original's metadata — `__name__`, `__doc__`, `__module__` all become the wrapper's. This breaks documentation, debugging, and any code that inspects function names. `functools.wraps` copies all that metadata from the original function onto the wrapper."*

---

## 🎯 Decorator with Arguments — The Three-Level Pattern

```python
# Regular decorator:   @decorator
# With arguments:      @decorator(arg)   ← needs one extra layer

def repeat(times):                    # LEVEL 1 — accepts decorator arguments
    def decorator(func):              # LEVEL 2 — accepts the function
        @functools.wraps(func)
        def wrapper(*args, **kwargs): # LEVEL 3 — the actual wrapper
            for _ in range(times):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(times=3)
def say_hello(name):
    print(f"Hello {name}!")

say_hello("Raj")
# Hello Raj!
# Hello Raj!
# Hello Raj!

# What Python does internally:
# say_hello = repeat(times=3)(say_hello)
# repeat(3) returns decorator
# decorator(say_hello) returns wrapper
# say_hello = wrapper
```

---

## 🏭 Production Decorators You Must Know

### Timing Decorator

```python
import time
import functools

def timer(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        end = time.perf_counter()
        print(f"{func.__name__} took {end - start:.4f}s")
        return result
    return wrapper

@timer
def process_large_dataset(n):
    return sum(range(n))

process_large_dataset(10_000_000)  # process_large_dataset took 0.3421s
```

### Retry Decorator

```python
import time
import functools

def retry(max_attempts=3, delay=1, exceptions=(Exception,)):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    last_exception = e
                    print(f"Attempt {attempt}/{max_attempts} failed: {e}")
                    if attempt < max_attempts:
                        time.sleep(delay)
            raise last_exception
        return wrapper
    return decorator

@retry(max_attempts=3, delay=2, exceptions=(ConnectionError,))
def call_external_api(endpoint):
    import random
    if random.random() < 0.7:
        raise ConnectionError("Timeout")
    return f"Data from {endpoint}"
```

### Cache / Memoize Decorator

```python
import functools

def memoize(func):
    cache = {}
    @functools.wraps(func)
    def wrapper(*args):
        if args not in cache:
            cache[args] = func(*args)
        return cache[args]
    return wrapper

@memoize
def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

# Python's built-in version:
from functools import lru_cache

@lru_cache(maxsize=128)
def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n-1) + fibonacci(n-2)
```

### Class-Based Decorator

```python
class RateLimit:
    """Limits function to N calls per minute"""
    def __init__(self, max_calls):
        self.max_calls = max_calls
        self.calls = []

    def __call__(self, func):           # makes instance callable as decorator
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            now = time.time()
            self.calls = [t for t in self.calls if now - t < 60]
            if len(self.calls) >= self.max_calls:
                raise Exception(f"Rate limit: max {self.max_calls} calls/minute")
            self.calls.append(now)
            return func(*args, **kwargs)
        return wrapper

@RateLimit(max_calls=5)
def api_endpoint():
    return "Response"
```

---

## Stacking Decorators — Order Matters

```python
def decorator_A(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        print("A before")
        result = func(*args, **kwargs)
        print("A after")
        return result
    return wrapper

def decorator_B(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        print("B before")
        result = func(*args, **kwargs)
        print("B after")
        return result
    return wrapper

@decorator_A          # applied SECOND (outer)
@decorator_B          # applied FIRST (inner)
def my_func():
    print("Function runs")

my_func()
# A before    ← A is outermost, runs first
# B before
# Function runs
# B after
# A after     ← A closes last

# Equivalent to: my_func = decorator_A(decorator_B(my_func))
```

> *"Decorators stack from bottom to top in application, but execute from top to bottom. The bottom decorator is applied first, making it the innermost wrapper. When the decorated function is called, execution flows from the outermost (top) decorator inward. I always reason about stacking by thinking: closest to the function = innermost = runs last before the function."*

---

# 📖 MODULE 3: MRO & C3 LINEARIZATION — Deeply Explained

---

## 🧠 Why MRO Exists — The Diamond Problem

```
        Animal
       /      \
    Dog        Cat
       \      /
        Hybrid
```

```python
class Animal:
    def speak(self):
        return "Animal speaks"

class Dog(Animal):
    def speak(self):
        return "Woof"

class Cat(Animal):
    def speak(self):
        return "Meow"

class Hybrid(Dog, Cat):    # inherits from both
    pass

h = Hybrid()
print(h.speak())    # Woof — but HOW does Python decide?
```

Without a rule, this is ambiguous. Python uses **C3 Linearization** to resolve it.

---

## 🔢 C3 Linearization — The Algorithm

**The formula:**
```
L(C) = C + merge(L(Parent1), L(Parent2), [Parent1, Parent2])
```

**Plain English version of the algorithm:**

```
1. Start with the class itself
2. Look at the first element of each parent's MRO list
3. Pick the first element that does NOT appear in the TAIL (non-first positions) of any other list
4. Add it to result, remove it from all lists
5. Repeat until all lists are empty
```

**Step-by-step example:**

```python
class A: pass
class B(A): pass
class C(A): pass
class D(B, C): pass

# Computing L(D):
# L(A) = [A, object]
# L(B) = [B, A, object]
# L(C) = [C, A, object]
# L(D) = D + merge([B, A, object], [C, A, object], [B, C])

# Step 1: First elements = B, C, B
#         B appears in tail of [B, C]? NO (B is first, not tail)
#         Pick B → result = [D, B]
#         Remove B from all lists → merge([A, object], [C, A, object], [C])

# Step 2: First elements = A, C, C
#         A appears in tail of [C, A, object]? YES (A is in tail)
#         Skip A. Try C. C appears in tail of any list? NO
#         Pick C → result = [D, B, C]
#         Remove C → merge([A, object], [A, object], [])

# Step 3: First elements = A, A
#         A appears in tail? NO
#         Pick A → result = [D, B, C, A]
#         Remove A → merge([object], [object], [])

# Step 4: Pick object → result = [D, B, C, A, object]

print(D.__mro__)
# (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)
```

---

## 🎯 How to Explain C3 in Interview — The Confident Answer

When asked about C3 linearization:

> *"C3 linearization is the algorithm Python uses to compute Method Resolution Order in multiple inheritance. The key rule is: take the leftmost class that doesn't appear in the tail of any other inheritance list. 'Tail' means everything except the first element. This ensures three things: a class always appears before its parents, the left-to-right order of parents is preserved, and no class appears twice. You can inspect the result with `ClassName.__mro__`. I don't compute it by hand in production — I check `__mro__` — but understanding the algorithm helps me design inheritance hierarchies that won't create ambiguity."*

**If they ask a counter-question: "What happens if C3 can't resolve?"**

```python
class A: pass
class B(A): pass
class C(A): pass

# This would create an inconsistent hierarchy:
# class D(A, B): pass   # TypeError: Cannot create a consistent MRO
# Because A comes before B in D's parents,
# but B should come before A (since B inherits A)

# Python catches this and raises TypeError at class definition
```

> *"If C3 can't find a consistent ordering — usually because the parent order contradicts the inheritance hierarchy — Python raises a `TypeError` at class definition time. It fails fast and loudly, which is good. The fix is to reorder the parents or rethink the inheritance structure."*

---

## super() Follows MRO — Critical Understanding

```python
class A:
    def method(self):
        print("A")
        super().method()    # calls next in MRO, not necessarily object

class B(A):
    def method(self):
        print("B")
        super().method()

class C(A):
    def method(self):
        print("C")
        super().method()

class D(B, C):
    def method(self):
        print("D")
        super().method()

D().method()
# D  → B  → C  → A
# MRO: D → B → C → A → object

# super() in B doesn't call A — it calls C (next in D's MRO)!
# This is cooperative multiple inheritance
```

> *"`super()` doesn't mean 'call my parent' — it means 'call the next class in the MRO.' This enables cooperative multiple inheritance where every class in the chain participates. For this to work, every class must call `super()`, otherwise the chain breaks. This is why Mixins should always call `super()`."*

---

# 📖 MODULE 4: LISKOV SUBSTITUTION PRINCIPLE — Truly Understood

---

## 🧠 The Core Idea in Plain English

LSP says: **"Anywhere you use a parent class, you should be able to swap in a child class and everything still works."**

More formally: if S is a subtype of T, then objects of type T may be replaced with objects of type S without altering the correctness of the program.

---

## ✅ LSP Satisfied — Good Example

```python
class Bird:
    def __init__(self, name):
        self.name = name

    def describe(self):
        return f"I am {self.name}"

class Sparrow(Bird):
    def describe(self):
        return f"I am {self.name}, a small bird"

class Eagle(Bird):
    def describe(self):
        return f"I am {self.name}, a large bird"

def print_bird_info(bird: Bird):    # expects Bird
    print(bird.describe())          # works with ANY Bird subclass

# All substitutions work correctly:
print_bird_info(Bird("Generic"))
print_bird_info(Sparrow("Jack"))
print_bird_info(Eagle("Eddie"))
```

---

## ❌ LSP Violated — The Classic Example

```python
class Bird:
    def fly(self):
        return "Flying!"

class Penguin(Bird):
    def fly(self):
        raise NotImplementedError("Penguins can't fly!")  # LSP VIOLATION

def make_bird_fly(bird: Bird):
    return bird.fly()   # caller expects this to always work

make_bird_fly(Bird())      # works
make_bird_fly(Penguin())   # CRASHES — LSP violated
```

**The fix — redesign the hierarchy:**

```python
from abc import ABC, abstractmethod

class Animal(ABC):
    def __init__(self, name):
        self.name = name

    @abstractmethod
    def move(self):
        pass

class FlyingBird(Animal):
    def move(self):
        return f"{self.name} is flying"

    def fly(self):
        return f"{self.name} soars through the air"

class NonFlyingBird(Animal):
    def move(self):
        return f"{self.name} is walking"

    def swim(self):
        return f"{self.name} swims gracefully"

class Eagle(FlyingBird): pass
class Sparrow(FlyingBird): pass
class Penguin(NonFlyingBird): pass
class Ostrich(NonFlyingBird): pass

def make_animal_move(animal: Animal):
    return animal.move()   # now works for ALL animals

make_animal_move(Eagle("Eddie"))    # Eddie is flying
make_animal_move(Penguin("Pete"))   # Pete is walking
```

---

## 🔍 LSP — The Four Rules (What Must Not Change in a Subclass)

```python
class DataProcessor:
    def process(self, data: list) -> dict:
        """
        Input: must be a list
        Output: must be a dict
        No exceptions thrown for valid input
        """
        return {"processed": len(data)}

# RULE 1: Subclass must accept AT LEAST what parent accepts
# (don't strengthen preconditions)
class BadProcessor(DataProcessor):
    def process(self, data: list) -> dict:
        if len(data) < 10:                   # VIOLATION — more restrictive
            raise ValueError("Need 10+ items")
        return {"processed": len(data)}

# RULE 2: Subclass must return AT MOST what parent promises
# (don't weaken postconditions)
class AnotherBadProcessor(DataProcessor):
    def process(self, data: list):           # VIOLATION — removed return type dict
        return "done"                        # parent promised dict, got string

# RULE 3: Don't throw new exceptions parent doesn't throw
class YetAnotherBad(DataProcessor):
    def process(self, data: list) -> dict:
        raise RuntimeError("Unexpected!")    # VIOLATION — parent never threw this

# CORRECT subclass
class GoodProcessor(DataProcessor):
    def process(self, data: list) -> dict:
        result = super().process(data)
        result["extra"] = "additional info"  # adds more, doesn't change contract
        return result
```

> **Complete interview answer for LSP:**
>
> *"Liskov Substitution Principle says a subclass should be substitutable for its parent without breaking the program. Concretely, four things must hold: subclasses can't strengthen preconditions — they must accept everything the parent accepts; subclasses can't weaken postconditions — they must return what the parent promises; subclasses shouldn't throw new exceptions the parent doesn't; and subclasses must maintain the parent's invariants. The classic violation is Penguin extending Bird with a `fly()` method — Penguin can't fly, so it breaks any code that calls `bird.fly()` on a Bird reference. The fix is better abstraction — separate flying and non-flying birds. I think about LSP when designing inheritance: ask 'would a caller using the parent break if given this child?' If yes, my hierarchy is wrong."*

---

# 📖 MODULE 5: METACLASSES — The Most Advanced OOP Topic

---

## 🧠 The Core Concept — Classes Are Objects Too

In Python, everything is an object. Including classes.

```python
class Dog:
    pass

# Dog is an object of type 'type'
print(type(Dog))         # <class 'type'>
print(type(42))          # <class 'int'>
print(type("hello"))     # <class 'str'>

# type IS the metaclass — the class of all classes
print(type(int))         # <class 'type'>
print(type(str))         # <class 'type'>
print(type(type))        # <class 'type'>  ← type is its own metaclass
```

A metaclass is simply: **the class of a class.**

```
Normal world:  instance = Class(...)         ← Class creates instances
Meta world:    Class    = Metaclass(...)     ← Metaclass creates Classes
```

---

## 🏗️ How Classes Are Created — Under the Hood

```python
# When you write this:
class MyClass:
    x = 10
    def greet(self): return "Hello"

# Python internally does:
MyClass = type("MyClass", (), {"x": 10, "greet": lambda self: "Hello"})
# type(name, bases, namespace_dict)

print(MyClass.x)             # 10
obj = MyClass()
print(obj.greet())           # Hello
```

---

## 🔧 Creating a Custom Metaclass

```python
class MyMeta(type):                          # inherit from type
    
    def __new__(mcs, name, bases, namespace):
        # mcs     = the metaclass itself (MyMeta)
        # name    = name of class being created ("MyClass")
        # bases   = tuple of parent classes
        # namespace = dict of class attributes and methods
        
        print(f"Creating class: {name}")
        print(f"  Parents: {bases}")
        print(f"  Attributes: {list(namespace.keys())}")
        
        # You can MODIFY the class before it's created
        namespace['created_by'] = 'MyMeta'
        
        return super().__new__(mcs, name, bases, namespace)
    
    def __init__(cls, name, bases, namespace):
        print(f"Initializing class: {name}")
        super().__init__(name, bases, namespace)

class MyClass(metaclass=MyMeta):         # use MyMeta as metaclass
    x = 10
    def greet(self): return "Hello"

# Creating class: MyClass
# Initializing class: MyClass

print(MyClass.created_by)    # MyMeta — we injected this!
```

---

## 🎯 Real-World Metaclass Use Cases

### Use Case 1: Enforce Method Implementation (like ABC but custom)

```python
class InterfaceMeta(type):
    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        
        # Check if this is a concrete class (has parents beyond object)
        if bases:
            required = ['save', 'load', 'delete']
            for method in required:
                if method not in namespace:
                    raise TypeError(
                        f"Class {name} must implement '{method}' method"
                    )
        return cls

class Repository(metaclass=InterfaceMeta):
    pass   # base class — no check

class UserRepository(Repository):
    def save(self): return "saved"
    def load(self): return "loaded"
    def delete(self): return "deleted"
    # All required methods present — works!

# class BrokenRepo(Repository):
#     def save(self): return "saved"
#     TypeError: Class BrokenRepo must implement 'load' method
```

### Use Case 2: Singleton via Metaclass

```python
class SingletonMeta(type):
    _instances = {}

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class Database(metaclass=SingletonMeta):
    def __init__(self, url):
        self.url = url
        print(f"Connecting to {url}")

db1 = Database("postgresql://localhost/mydb")
db2 = Database("postgresql://localhost/mydb")

print(db1 is db2)    # True — same instance
# "Connecting to..." printed only ONCE
```

### Use Case 3: Auto-Register Subclasses

```python
class PluginMeta(type):
    registry = {}

    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        if bases:   # don't register the base Plugin class itself
            mcs.registry[name] = cls
            print(f"Registered plugin: {name}")
        return cls

class Plugin(metaclass=PluginMeta):
    def run(self): pass

class PDFPlugin(Plugin):
    def run(self): return "Processing PDF"

class ExcelPlugin(Plugin):
    def run(self): return "Processing Excel"

class CSVPlugin(Plugin):
    def run(self): return "Processing CSV"

print(PluginMeta.registry)
# {'PDFPlugin': <class 'PDFPlugin'>, 'ExcelPlugin': ..., 'CSVPlugin': ...}

# Use registry to run any plugin by name
def run_plugin(name):
    plugin_cls = PluginMeta.registry.get(name)
    if plugin_cls:
        return plugin_cls().run()

print(run_plugin("PDFPlugin"))   # Processing PDF
```

### Use Case 4: Auto-Add Logging to All Methods

```python
import functools

class LogAllMeta(type):
    def __new__(mcs, name, bases, namespace):
        for attr_name, attr_value in namespace.items():
            if callable(attr_value) and not attr_name.startswith('_'):
                namespace[attr_name] = mcs.log_call(attr_value)
        return super().__new__(mcs, name, bases, namespace)

    @staticmethod
    def log_call(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            print(f"[LOG] Calling: {func.__name__}")
            result = func(*args, **kwargs)
            print(f"[LOG] Done: {func.__name__}")
            return result
        return wrapper

class Service(metaclass=LogAllMeta):
    def process(self):
        return "processing"

    def save(self):
        return "saving"

s = Service()
s.process()
# [LOG] Calling: process
# [LOG] Done: process
```

---

## `__new__` vs `__init__` vs `__init_subclass__` — All Three

```python
class MyClass:

    def __new__(cls, *args, **kwargs):
        print(f"1. __new__ called — creating instance of {cls}")
        instance = super().__new__(cls)   # actually creates the object
        return instance                    # MUST return the instance

    def __init__(self, name):
        print(f"2. __init__ called — setting up {name}")
        self.name = name

    def __init_subclass__(cls, **kwargs):
        # Called when THIS class is subclassed
        super().__init_subclass__(**kwargs)
        print(f"3. __init_subclass__ called — {cls.__name__} just subclassed MyClass")

class Child(MyClass):    # triggers __init_subclass__
    pass
# 3. __init_subclass__ called — Child just subclassed MyClass

obj = MyClass("Raj")
# 1. __new__ called — creating instance
# 2. __init__ called — setting up Raj
```

**Use cases:**
- `__new__` → Singleton, flyweight pattern, subclassing immutables (`int`, `str`, `tuple`)
- `__init__` → Normal initialization (99% of cases)
- `__init_subclass__` → Lightweight alternative to metaclass for registering subclasses

---

## `__class_getitem__` — For Generics Support

```python
class TypedList:
    def __class_getitem__(cls, item):
        # Called when you write TypedList[int]
        print(f"Creating TypedList of type {item}")
        return cls

# Enables this syntax:
x: TypedList[int] = TypedList()
```

---

> **Interview answer for metaclasses:**
>
> *"A metaclass is the class of a class — it controls how classes are created, just like a class controls how objects are created. The default metaclass is `type`. I create custom metaclasses by inheriting from `type` and overriding `__new__` or `__init__`. I use metaclasses for framework-level concerns: enforcing interfaces, auto-registering subclasses, auto-injecting logging, or implementing Singleton. They're powerful but heavy — in most application code I'd reach for `__init_subclass__`, class decorators, or ABC before metaclasses. ORMs like Django use metaclasses — `Model` classes automatically register with the ORM because of metaclass magic."*

---

# 📖 MODULE 6: SPECIAL/DUNDER METHODS — Complete Reference

---

## 🧠 The Full Picture

Dunder methods (double underscore) are Python's protocol system. They define how objects behave with built-in operations.

```python
class SmartList:
    """A list with smart behavior — demonstrates all major dunders"""

    def __init__(self, items=None):
        self._items = items or []

    # ── REPRESENTATION ─────────────────────────────────────
    def __str__(self):
        return f"SmartList({self._items})"

    def __repr__(self):
        return f"SmartList(items={self._items!r})"

    # ── CONTAINER PROTOCOL ──────────────────────────────────
    def __len__(self):
        return len(self._items)             # len(smart_list)

    def __getitem__(self, index):
        return self._items[index]           # smart_list[0]

    def __setitem__(self, index, value):
        self._items[index] = value          # smart_list[0] = "new"

    def __delitem__(self, index):
        del self._items[index]              # del smart_list[0]

    def __contains__(self, item):
        return item in self._items          # "x" in smart_list

    def __iter__(self):
        return iter(self._items)            # for x in smart_list

    def __reversed__(self):
        return reversed(self._items)        # reversed(smart_list)

    # ── ARITHMETIC ──────────────────────────────────────────
    def __add__(self, other):
        return SmartList(self._items + other._items)   # sl1 + sl2

    def __mul__(self, n):
        return SmartList(self._items * n)              # sl * 3

    # ── COMPARISON ──────────────────────────────────────────
    def __eq__(self, other):
        return self._items == other._items  # sl1 == sl2

    def __lt__(self, other):
        return len(self) < len(other)       # sl1 < sl2 (enables sorting!)

    # ── CALLABLE ────────────────────────────────────────────
    def __call__(self, filter_fn):
        return SmartList([x for x in self._items if filter_fn(x)])

    # ── CONTEXT MANAGER ─────────────────────────────────────
    def __enter__(self):
        print("Entering SmartList context")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        print("Exiting SmartList context")
        return False   # don't suppress exceptions

    # ── ATTRIBUTE ACCESS ────────────────────────────────────
    def __getattr__(self, name):
        # Called when normal attribute lookup fails
        raise AttributeError(f"SmartList has no attribute '{name}'")


sl = SmartList([1, 2, 3, 4, 5])

print(len(sl))           # 5
print(sl[0])             # 1
print(3 in sl)           # True
print(sl * 2)            # SmartList([1,2,3,4,5,1,2,3,4,5])

evens = sl(lambda x: x % 2 == 0)
print(evens)             # SmartList([2, 4])

with sl as s:
    print(s[0])          # 1
```

---

## `__getattr__` vs `__getattribute__` — Critical Difference

```python
class MyClass:

    def __getattr__(self, name):
        # Called ONLY when attribute NOT found normally
        # Good for dynamic attributes, proxies
        print(f"__getattr__ called for: {name}")
        return f"default_{name}"

    def __getattribute__(self, name):
        # Called for EVERY attribute access, even existing ones
        # Very dangerous to override — can cause infinite recursion
        print(f"__getattribute__ called for: {name}")
        return super().__getattribute__(name)  # MUST call super

obj = MyClass()
obj.x = 10

print(obj.x)      # __getattribute__ called → 10
print(obj.y)      # __getattribute__ called → not found → __getattr__ called → default_y
```

> *"`__getattr__` is the fallback — only called when normal lookup fails. Safe to override for proxy patterns or dynamic attributes. `__getattribute__` intercepts EVERY attribute access including existing ones — very rarely overridden because it's easy to cause infinite recursion (accessing `self.anything` inside it triggers itself). If I override it, I always use `super().__getattribute__()` to do the actual lookup."*

---

## Context Managers — `__enter__` and `__exit__`

```python
class DatabaseTransaction:
    def __init__(self, db_url):
        self.db_url = db_url
        self.connection = None

    def __enter__(self):
        print(f"Opening connection to {self.db_url}")
        self.connection = f"connection_to_{self.db_url}"
        return self.connection      # what 'as' variable receives

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type:
            print(f"Exception occurred: {exc_val}. Rolling back.")
        else:
            print("Success. Committing transaction.")
        print("Closing connection")
        self.connection = None
        return False   # False = don't suppress the exception

with DatabaseTransaction("postgresql://localhost/mydb") as conn:
    print(f"Using {conn}")
    # do database operations

# Opening connection to postgresql://...
# Using connection_to_postgresql://...
# Success. Committing.
# Closing connection
```

**contextlib alternative — simpler for simple cases:**

```python
from contextlib import contextmanager

@contextmanager
def db_transaction(url):
    conn = f"connection_to_{url}"
    print("Opening connection")
    try:
        yield conn          # what 'as' receives
        print("Committing")
    except Exception as e:
        print(f"Rolling back: {e}")
        raise
    finally:
        print("Closing connection")

with db_transaction("postgresql://localhost/mydb") as conn:
    print(f"Using {conn}")
```

---

## `__slots__` — Memory Optimization

```python
class RegularPoint:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    # Has __dict__ — can add arbitrary attributes

class SlottedPoint:
    __slots__ = ['x', 'y']    # fixed set of attributes
    def __init__(self, x, y):
        self.x = x
        self.y = y
    # NO __dict__ — cannot add arbitrary attributes

import sys
rp = RegularPoint(1, 2)
sp = SlottedPoint(1, 2)

print(sys.getsizeof(rp.__dict__))  # ~232 bytes
# sp has no __dict__ — saves memory

# sp.z = 10    # AttributeError — slots enforced
```

> *"`__slots__` replaces the instance `__dict__` with a fixed set of descriptors. This saves memory — significant when creating millions of objects — and also speeds up attribute access. The tradeoff: no dynamic attributes, no `__weakref__` by default, and more complex with multiple inheritance. I use `__slots__` in data-heavy applications like game engines, scientific computing, or high-throughput APIs."*

---

# 📖 MODULE 7: SOLID PRINCIPLES — All Five

---

## S — Single Responsibility

```python
# WRONG — one class doing too much
class Employee:
    def calculate_pay(self): pass
    def save_to_database(self): pass    # database concern
    def generate_report(self): pass     # reporting concern
    def send_email(self): pass          # communication concern

# RIGHT — each class has one reason to change
class Employee:
    def __init__(self, name, salary):
        self.name = name
        self.salary = salary

class PayrollCalculator:
    def calculate(self, employee): pass

class EmployeeRepository:
    def save(self, employee): pass

class ReportGenerator:
    def generate(self, employee): pass
```

## O — Open/Closed Principle

```python
# WRONG — must modify class to add new shape
class AreaCalculator:
    def calculate(self, shape):
        if shape.type == "circle":
            return 3.14 * shape.radius ** 2
        elif shape.type == "rectangle":           # add new type? edit this class
            return shape.width * shape.height

# RIGHT — open for extension, closed for modification
class Shape(ABC):
    @abstractmethod
    def area(self): pass

class Circle(Shape):
    def area(self): return 3.14 * self.radius ** 2

class Rectangle(Shape):
    def area(self): return self.width * self.height

class Triangle(Shape):                             # new shape — no modification needed
    def area(self): return 0.5 * self.base * self.height

def calculate_total_area(shapes):
    return sum(s.area() for s in shapes)          # works with any shape
```

## I — Interface Segregation

```python
# WRONG — fat interface forces irrelevant implementations
class Worker(ABC):
    @abstractmethod
    def work(self): pass

    @abstractmethod
    def eat(self): pass       # robots don't eat!

    @abstractmethod
    def sleep(self): pass     # robots don't sleep!

class Robot(Worker):
    def work(self): return "working"
    def eat(self): raise NotImplementedError   # forced to implement irrelevant method
    def sleep(self): raise NotImplementedError

# RIGHT — segregated interfaces
class Workable(ABC):
    @abstractmethod
    def work(self): pass

class Eatable(ABC):
    @abstractmethod
    def eat(self): pass

class Human(Workable, Eatable):
    def work(self): return "working"
    def eat(self): return "eating"

class Robot(Workable):          # only what's relevant
    def work(self): return "working"
```

## D — Dependency Inversion

```python
# WRONG — high-level depends on low-level
class EmailService:
    def send(self, msg): return f"Email: {msg}"

class UserNotifier:
    def __init__(self):
        self.service = EmailService()    # hard-coded dependency!

    def notify(self, msg):
        self.service.send(msg)

# RIGHT — depend on abstraction
class NotificationService(ABC):
    @abstractmethod
    def send(self, message): pass

class EmailService(NotificationService):
    def send(self, message): return f"Email: {message}"

class SMSService(NotificationService):
    def send(self, message): return f"SMS: {message}"

class UserNotifier:
    def __init__(self, service: NotificationService):   # inject dependency
        self.service = service

    def notify(self, msg):
        return self.service.send(msg)

notifier = UserNotifier(EmailService())    # easy to swap
notifier = UserNotifier(SMSService())      # zero code change in UserNotifier
```

---

# 📖 MODULE 8: ADVANCED PATTERNS

---

## Descriptor Protocol — How Properties Work Internally

```python
class Validator:
    """A descriptor that validates values"""

    def __set_name__(self, owner, name):
        self.name = name
        self.private_name = f"_{name}"

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self        # accessed from class, not instance
        return getattr(obj, self.private_name, None)

    def __set__(self, obj, value):
        self.validate(value)
        setattr(obj, self.private_name, value)

    def validate(self, value):
        pass   # override in subclasses

class PositiveNumber(Validator):
    def validate(self, value):
        if not isinstance(value, (int, float)) or value <= 0:
            raise ValueError(f"{self.name} must be a positive number, got {value}")

class Employee:
    salary = PositiveNumber()    # descriptor
    age = PositiveNumber()       # descriptor

    def __init__(self, name, salary, age):
        self.name = name
        self.salary = salary     # triggers PositiveNumber.__set__
        self.age = age

e = Employee("Raj", 50000, 30)
# e.salary = -1000    # ValueError: salary must be a positive number
```

---

## Abstract Factory Pattern

```python
from abc import ABC, abstractmethod

# Abstract products
class Button(ABC):
    @abstractmethod
    def render(self): pass

class TextInput(ABC):
    @abstractmethod
    def render(self): pass

# Concrete products — Web
class WebButton(Button):
    def render(self): return "<button>Click</button>"

class WebTextInput(TextInput):
    def render(self): return "<input type='text'/>"

# Concrete products — Mobile
class MobileButton(Button):
    def render(self): return "[ BUTTON ]"

class MobileTextInput(TextInput):
    def render(self): return "[ TEXT INPUT ]"

# Abstract factory
class UIFactory(ABC):
    @abstractmethod
    def create_button(self) -> Button: pass

    @abstractmethod
    def create_input(self) -> TextInput: pass

# Concrete factories
class WebFactory(UIFactory):
    def create_button(self): return WebButton()
    def create_input(self): return WebTextInput()

class MobileFactory(UIFactory):
    def create_button(self): return MobileButton()
    def create_input(self): return MobileTextInput()

def render_form(factory: UIFactory):
    btn = factory.create_button()
    inp = factory.create_input()
    return f"{btn.render()} + {inp.render()}"

print(render_form(WebFactory()))
print(render_form(MobileFactory()))
```

---

# 📋 MASTER INTERVIEW Q&A — Advanced

**Q: What is the difference between a decorator and a metaclass?**

> *"Both modify behavior, but at different levels. A decorator modifies a function or class after it's created — it's applied at decoration time. A metaclass modifies how classes are created — it intercepts class creation itself. Decorators are simpler and more explicit. Metaclasses are more powerful but harder to reason about. Rule of thumb: if I can solve it with a decorator or `__init_subclass__`, I do that. Metaclasses are for framework-level concerns where I need to control class creation across an entire hierarchy."*

**Q: When would you use `__slots__`?**

> *"When I'm creating large numbers of small objects and memory is a concern. `__slots__` removes the per-instance `__dict__`, which saves 50-200 bytes per instance. At a million objects, that's significant. I also use it when I want to restrict what attributes can be set — it prevents accidental attribute creation. The tradeoffs are: no dynamic attributes, more complex multiple inheritance, and no `__weakref__` unless explicitly added to slots."*

**Q: Explain the Descriptor Protocol.**

> *"Descriptors are objects that define `__get__`, `__set__`, or `__delete__`. When you access an attribute on an object, Python checks if the class has a descriptor for that name. If it does, Python delegates to the descriptor's methods. This is how `@property`, `@classmethod`, `@staticmethod`, and Django model fields all work internally. I use custom descriptors for reusable validation logic — define the validator once as a descriptor, apply it to multiple class attributes."*

**Q: What is the difference between `__new__` and `__init__`?**

> *"`__new__` creates the instance — it's called first, receives the class as first argument, and must return the new instance. `__init__` initializes the instance — it's called second, receives the already-created instance as `self`, and sets up attributes. I override `__new__` for Singleton, for subclassing immutable types like `int` or `tuple` where `__init__` is too late to set the value, or in metaclasses. For 99% of cases, only `__init__` needs overriding."*

---

# 🗺️ STUDY ROADMAP — 4 Weeks

| Week | Focus | Goal |
|------|-------|------|
| 1 | Closures + Decorators | Build 3 real decorators from scratch |
| 2 | OOP Pillars + MRO + LSP | Explain each out loud without notes |
| 3 | Metaclasses + Dunders + SOLID | Build a mini-framework using metaclasses |
| 4 | Mock Interviews | Answer all Q&A timed — 45-90 seconds each |

---

# 💡 THE GOLDEN INTERVIEW FORMULA

Every technical answer = **What + Why + Example + Production**

- **What** — one sentence definition
- **Why** — what problem it solves
- **Example** — concrete code scenario
- **Production** — how you've used or would use it at work

Never just define. Always connect to real impact.
