# 🎓 PYTHON OOP — COMPLETE GUIDE: BASICS TO ADVANCED
### Every Concept, Every Counter-Question, Every Detail — Interview Ready

---

# 📌 THE GOLDEN MENTAL MODEL

Before learning anything — burn this into your brain:

**OOP is just a way to organize code by modeling real-world things.**

| Concept | Job | Real-World Analogy |
|---------|-----|--------------------|
| Class | Blueprint | Architectural plan for a house |
| Object | Actual thing | The house built from the plan |
| Encapsulation | Hide internals | Capsule pill — you use it, don't see inside |
| Inheritance | Reuse from parent | Child inherits traits from parent |
| Polymorphism | Same action, different behavior | "Speak" → dog barks, cat meows |
| Abstraction | Show only what's needed | TV remote — you press buttons, don't see circuits |
| Closure | Function remembers its environment | A backpack carrying variables |
| Decorator | Wrapper adding behavior | Security guard at a door |
| Metaclass | Class that creates classes | Factory that builds factories |

---

# ═══════════════════════════════════════
# 🟢 PART 1: BASICS
# ═══════════════════════════════════════

---

# 📖 MODULE 1: CLASSES & OBJECTS

---

## 🧠 Why Do Classes Exist?

Imagine building a hospital system for 500 patients. Without classes:

```python
# WITHOUT classes — nightmare
patient1_name = "Raj"
patient1_age = 35
patient1_blood = "A+"

patient2_name = "Priya"
patient2_age = 28
patient2_blood = "B+"
# For 500 patients → completely unmanageable
```

With classes:

```python
# WITH classes — clean, scalable
class Patient:
    def __init__(self, name, age, blood_group):
        self.name = name
        self.age = age
        self.blood_group = blood_group

    def get_info(self):
        return f"Patient: {self.name}, Age: {self.age}, Blood: {self.blood_group}"

p1 = Patient("Raj", 35, "A+")
p2 = Patient("Priya", 28, "B+")
print(p1.get_info())
# Patient: Raj, Age: 35, Blood: A+
```

> **Classes exist because as systems grow, you need a way to bundle related data and behavior into reusable, manageable units.**

---

## 🔬 Anatomy of a Class — Every Part Explained

```python
class BankAccount:

    # ── CLASS VARIABLE ──────────────────────────────────────
    # Shared across ALL instances — one copy in memory
    bank_name = "Deloitte Bank"
    total_accounts = 0

    # ── CONSTRUCTOR ─────────────────────────────────────────
    # Runs automatically when you do BankAccount(...)
    # self = the specific object being created RIGHT NOW
    def __init__(self, owner, balance=0):

        # ── INSTANCE VARIABLES ──────────────────────────────
        # Unique to each object — each account has its own
        self.owner = owner
        self.balance = balance
        BankAccount.total_accounts += 1    # update shared counter

    # ── INSTANCE METHOD ─────────────────────────────────────
    # Needs self — works on THIS specific account
    def deposit(self, amount):
        self.balance += amount
        return f"Deposited ₹{amount}. New balance: ₹{self.balance}"

    def withdraw(self, amount):
        if amount > self.balance:
            return "Insufficient funds"
        self.balance -= amount
        return f"Withdrawn ₹{amount}. Remaining: ₹{self.balance}"

    # ── STRING REPRESENTATION ───────────────────────────────
    def __str__(self):      # called by print()
        return f"Account[{self.owner}] Balance: ₹{self.balance}"

    def __repr__(self):     # called in console/debugging
        return f"BankAccount(owner='{self.owner}', balance={self.balance})"


# Creating objects
acc1 = BankAccount("Raj", 10000)
acc2 = BankAccount("Priya", 25000)

print(acc1.deposit(5000))          # Deposited ₹5000. New balance: ₹15000
print(acc2.withdraw(30000))        # Insufficient funds
print(BankAccount.total_accounts)  # 2
print(acc1)                        # Account[Raj] Balance: ₹15000
```

---

## 🎯 Class Variable vs Instance Variable — Most Commonly Confused

```python
class Employee:
    company = "Deloitte"      # class variable — ONE copy, shared by all

    def __init__(self, name):
        self.name = name      # instance variable — each object has its own

e1 = Employee("Raj")
e2 = Employee("Priya")

print(e1.company)    # Deloitte
print(e2.company)    # Deloitte

# Change class variable — affects ALL
Employee.company = "Google"
print(e1.company)    # Google
print(e2.company)    # Google

# ⚠️ TRAP — doing this creates an INSTANCE variable, not changing class variable
e1.company = "Microsoft"
print(e1.company)        # Microsoft  ← instance variable SHADOWS class variable
print(e2.company)        # Google     ← class variable unchanged
print(Employee.company)  # Google     ← class variable unchanged
```

> **Interview answer:**
> *"Class variables are shared across all instances — one copy in memory. Instance variables belong to each object individually. A subtle trap: assigning to a class variable via an instance like `obj.class_var = value` creates a new instance variable that shadows the class variable for that object — it doesn't change the class variable itself."*

---

## 🔑 `self` — Deeply Understood

```python
class Dog:
    def __init__(self, name):
        self.name = name

    def bark(self):
        return f"{self.name} says Woof!"

d = Dog("Bruno")
d.bark()

# Python INTERNALLY translates d.bark() to:
Dog.bark(d)    # d is passed as self automatically
```

`self` is NOT a keyword. You can name it anything (but don't):

```python
class Weird:
    def __init__(this, name):   # valid
        this.name = name

    def greet(me):              # also valid
        return f"I am {me.name}"
```

> *"`self` is a reference to the current instance. Python automatically passes the object as the first argument when you call a method. Through `self`, the method accesses and modifies that specific object's data."*

---

## ⚡ `__str__` vs `__repr__`

```python
class Product:
    def __init__(self, name, price):
        self.name = name
        self.price = price

    def __str__(self):
        # Human-friendly — for end users
        return f"{self.name} costs ₹{self.price}"

    def __repr__(self):
        # Developer-friendly — for debugging, ideally recreates the object
        return f"Product(name='{self.name}', price={self.price})"

p = Product("Laptop", 75000)
print(p)          # calls __str__  → Laptop costs ₹75000
print(repr(p))    # calls __repr__ → Product(name='Laptop', price=75000)

products = [p]
print(products)   # uses __repr__ → [Product(name='Laptop', price=75000)]
```

> *"`__str__` is for display — what you show end users. `__repr__` is for debugging — should ideally be enough to recreate the object. If `__str__` isn't defined, Python falls back to `__repr__`."*

---

# 📖 MODULE 2: ENCAPSULATION — Deep Dive

---

## 🧠 The Real Purpose — Beyond Just "Hiding"

Encapsulation has two jobs:
1. Bundle data and methods that operate on it together
2. Control access to protect data integrity

```python
# WITHOUT encapsulation — dangerous
class Temperature:
    def __init__(self):
        self.celsius = 0

temp = Temperature()
temp.celsius = -300   # physically impossible! Nothing stops this
```

```python
# WITH encapsulation — protected
class Temperature:
    def __init__(self):
        self.__celsius = 0    # private

    def set_celsius(self, value):
        if value < -273.15:   # absolute zero check
            raise ValueError("Temperature below absolute zero is impossible!")
        self.__celsius = value

    def get_celsius(self):
        return self.__celsius

    def get_fahrenheit(self):
        return (self.__celsius * 9/5) + 32

temp = Temperature()
temp.set_celsius(25)
print(temp.get_celsius())     # 25
print(temp.get_fahrenheit())  # 77.0
# temp.set_celsius(-300)      # ValueError — protected!
```

---

## 🔐 Access Levels — Visualized

```
Public    → self.name        → Anyone can access freely
Protected → self._name       → Convention: "internal use, don't touch"
Private   → self.__name      → Python enforces: name-mangled to _ClassName__name
```

```python
class HospitalRecord:
    def __init__(self, patient, diagnosis, ssn):
        self.patient = patient          # public
        self._diagnosis = diagnosis     # protected
        self.__ssn = ssn               # private

    def get_ssn_last4(self):
        return f"***-**-{str(self.__ssn)[-4:]}"

record = HospitalRecord("Raj", "Diabetes", 123456789)

print(record.patient)               # Raj — works fine
print(record._diagnosis)            # Diabetes — works but frowned upon
# print(record.__ssn)               # AttributeError
print(record._HospitalRecord__ssn)  # 123456789 — name mangling trick
print(record.get_ssn_last4())       # ***-**-6789 — clean controlled access
```

---

## 🏠 Properties — The Pythonic Way

```python
class Circle:
    def __init__(self, radius):
        self._radius = radius

    @property
    def radius(self):               # getter — access like attribute
        return self._radius

    @radius.setter
    def radius(self, value):        # setter — with validation
        if value < 0:
            raise ValueError("Radius cannot be negative")
        self._radius = value

    @property
    def area(self):                 # computed property — read-only
        return 3.14159 * self._radius ** 2

c = Circle(5)
print(c.radius)      # 5   ← looks like attribute access, is actually method
print(c.area)        # 78.53975
c.radius = 10        # calls setter with validation
# c.radius = -1      # ValueError!
```

> *"I prefer `@property` over explicit getters/setters because it keeps the interface clean — callers access it like an attribute, but internally I have full control with validation. It's the Pythonic way to implement encapsulation."*

---

## 🎯 Encapsulation Interview Questions

**Q: "Why make an attribute private?"**
> *"To protect data integrity. If an attribute must satisfy conditions — age must be positive, temperature can't go below absolute zero — making it private and using a controlled setter prevents invalid states. It also future-proofs the code — I can change the internal implementation without breaking external code."*

**Q: "Can you access private attributes in Python?"**
> *"Technically yes — Python uses name mangling, not true access control. `__attr` becomes `_ClassName__attr`, still accessible but deliberately ugly. Python trusts developers with convention — 'we're all consenting adults here' is the Python philosophy."*

**Q: "Difference between `_name` and `__name`?"**
> *"Single underscore is a convention — 'protected, please don't touch.' Double underscore triggers actual name mangling — Python renames it to `_ClassName__name` to prevent accidental access in subclasses. Double underscore is especially useful when you genuinely don't want subclasses accidentally overriding an attribute."*

---

# 📖 MODULE 3: INHERITANCE — Deep Dive

---

## 🧠 The Core Purpose — DRY (Don't Repeat Yourself)

```python
# WITHOUT inheritance — repeated code
class Customer:
    def __init__(self, name, email):
        self.name = name
        self.email = email
    def send_email(self, msg):          # duplicated in Admin
        print(f"Sending to {self.email}: {msg}")

class Admin:
    def __init__(self, name, email):    # same as Customer!
        self.name = name
        self.email = email
    def send_email(self, msg):          # same as Customer!
        print(f"Sending to {self.email}: {msg}")
    def delete_user(self, user):
        print(f"{user} deleted")
```

```python
# WITH inheritance — clean
class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email

    def send_email(self, msg):
        print(f"Sending to {self.email}: {msg}")

class Customer(User):
    def __init__(self, name, email, loyalty_points=0):
        super().__init__(name, email)   # reuse parent's init
        self.loyalty_points = loyalty_points

    def add_points(self, points):
        self.loyalty_points += points
        return f"{self.name} now has {self.loyalty_points} points"

class Admin(User):
    def __init__(self, name, email, access_level):
        super().__init__(name, email)
        self.access_level = access_level

    def delete_user(self, user):
        if self.access_level >= 3:
            return f"Admin {self.name} deleted {user}"
        return "Insufficient access"

c = Customer("Raj", "raj@gmail.com")
a = Admin("Priya", "priya@deloitte.com", access_level=5)

c.send_email("Your order is shipped!")    # inherited from User
a.send_email("System maintenance alert")  # inherited from User
print(c.add_points(100))
print(a.delete_user("Raj"))
```

---

## 🔗 `super()` — Every Scenario

```python
class Vehicle:
    def __init__(self, brand, speed):
        self.brand = brand
        self.speed = speed

    def describe(self):
        return f"{self.brand} at {self.speed}km/h"

class Car(Vehicle):
    def __init__(self, brand, speed, doors):
        super().__init__(brand, speed)    # calls Vehicle.__init__
        self.doors = doors

    def describe(self):
        base = super().describe()         # calls Vehicle.describe()
        return f"{base}, {self.doors} doors"

class ElectricCar(Car):
    def __init__(self, brand, speed, doors, battery):
        super().__init__(brand, speed, doors)  # calls Car.__init__
        self.battery = battery

    def describe(self):
        base = super().describe()              # calls Car.describe()
        return f"{base}, {self.battery}kWh battery"

e = ElectricCar("Tesla", 250, 4, 100)
print(e.describe())
# Tesla at 250km/h, 4 doors, 100kWh battery
```

> *"`super()` gives access to the parent class. Most commonly used in `__init__` to reuse parent initialization without duplicating it. Also used in methods to extend parent behavior rather than completely replace it. In Python 3, `super()` doesn't need arguments."*

---

## Types of Inheritance — All 5

```python
# 1. SINGLE — one child, one parent
class Animal: pass
class Dog(Animal): pass

# 2. MULTILEVEL — chain
class Animal: pass
class Mammal(Animal): pass
class Dog(Mammal): pass      # Dog → Mammal → Animal

# 3. HIERARCHICAL — multiple children, one parent
class Animal: pass
class Dog(Animal): pass
class Cat(Animal): pass

# 4. MULTIPLE — one child, multiple parents
class Flyable: pass
class Swimmable: pass
class Duck(Flyable, Swimmable): pass

# 5. HYBRID — combination
class Animal: pass
class Flyable: pass
class Bird(Animal, Flyable): pass    # multiple
class Parrot(Bird): pass             # + multilevel
```

---

# 📖 MODULE 4: POLYMORPHISM — Deep Dive

---

## Type 1: Method Overriding

```python
class Notification:
    def __init__(self, message):
        self.message = message

    def send(self):
        raise NotImplementedError("Subclass must implement send()")

class EmailNotification(Notification):
    def __init__(self, message, email):
        super().__init__(message)
        self.email = email

    def send(self):
        return f"Email to {self.email}: {self.message}"

class SMSNotification(Notification):
    def __init__(self, message, phone):
        super().__init__(message)
        self.phone = phone

    def send(self):
        return f"SMS to {self.phone}: {self.message}"

class PushNotification(Notification):
    def __init__(self, message, device_id):
        super().__init__(message)
        self.device_id = device_id

    def send(self):
        return f"Push to {self.device_id}: {self.message}"

# POLYMORPHISM — same interface, different behavior
notifications = [
    EmailNotification("Order shipped!", "raj@gmail.com"),
    SMSNotification("Your OTP is 1234", "+91-9876543210"),
    PushNotification("Flash sale now!", "device_abc")
]

for notif in notifications:
    print(notif.send())   # each calls its own send()
```

---

## Type 2: Duck Typing — Python's Natural Polymorphism

```python
# "If it walks like a duck and quacks like a duck, it's a duck"
# Python doesn't care about type — only whether the method exists

class PDFReport:
    def generate(self):
        return "Generating PDF..."

class ExcelReport:
    def generate(self):
        return "Generating Excel..."

class HTMLReport:
    def generate(self):
        return "Generating HTML..."

def create_report(report):      # doesn't care what type
    return report.generate()    # just needs generate() to exist

for r in [PDFReport(), ExcelReport(), HTMLReport()]:
    print(create_report(r))
```

---

## Operator Overloading — Magic Methods

```python
class Vector:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __add__(self, other):     # v1 + v2
        return Vector(self.x + other.x, self.y + other.y)

    def __sub__(self, other):     # v1 - v2
        return Vector(self.x - other.x, self.y - other.y)

    def __mul__(self, scalar):    # v * 3
        return Vector(self.x * scalar, self.y * scalar)

    def __eq__(self, other):      # v1 == v2
        return self.x == other.x and self.y == other.y

    def __len__(self):            # len(v)
        return int((self.x**2 + self.y**2) ** 0.5)

    def __str__(self):
        return f"Vector({self.x}, {self.y})"

v1 = Vector(2, 3)
v2 = Vector(1, 4)
print(v1 + v2)    # Vector(3, 7)
print(v1 * 3)     # Vector(6, 9)
print(v1 == v2)   # False
print(len(v1))    # 3
```

**Key Magic Methods:**

```
__init__     → constructor
__str__      → print(obj)
__repr__     → repr(obj), console
__len__      → len(obj)
__add__      → obj + obj
__eq__       → obj == obj
__lt__       → obj < obj (enables sorting!)
__contains__ → item in obj
__iter__     → for item in obj
__getitem__  → obj[key]
__setitem__  → obj[key] = value
__call__     → obj()  (makes object callable like a function)
__enter__    → with obj as x:
__exit__     → end of with block
```

---

## `__call__` — Making Objects Callable

```python
class Multiplier:
    def __init__(self, factor):
        self.factor = factor

    def __call__(self, number):    # obj() — callable!
        return number * self.factor

double = Multiplier(2)
triple = Multiplier(3)

print(double(5))    # 10
print(triple(5))    # 15
# double is an object used like a function
```

---

# 📖 MODULE 5: ABSTRACTION — Deep Dive

---

## Abstract Base Classes

```python
from abc import ABC, abstractmethod

class PaymentProcessor(ABC):    # Abstract — can't instantiate

    @abstractmethod
    def authenticate(self):
        """Must verify credentials with payment provider"""
        pass

    @abstractmethod
    def process_payment(self, amount, currency):
        """Must handle actual payment transaction"""
        pass

    @abstractmethod
    def refund(self, transaction_id, amount):
        """Must handle refund logic"""
        pass

    # Non-abstract method — shared logic for all
    def format_amount(self, amount, currency):
        return f"{currency} {amount:.2f}"


class RazorpayProcessor(PaymentProcessor):
    def authenticate(self):
        return "Razorpay: API key verified"

    def process_payment(self, amount, currency):
        return f"Razorpay processing {self.format_amount(amount, currency)}"

    def refund(self, transaction_id, amount):
        return f"Razorpay refunding ₹{amount} for txn {transaction_id}"


class StripeProcessor(PaymentProcessor):
    def authenticate(self):
        return "Stripe: OAuth token verified"

    def process_payment(self, amount, currency):
        return f"Stripe processing {self.format_amount(amount, currency)}"

    def refund(self, transaction_id, amount):
        return f"Stripe refunding ${amount} for txn {transaction_id}"


# PaymentProcessor()   # TypeError — can't instantiate abstract class

def run_payment(processor: PaymentProcessor, amount, currency):
    print(processor.authenticate())
    print(processor.process_payment(amount, currency))

run_payment(RazorpayProcessor(), 1500, "INR")
run_payment(StripeProcessor(), 20, "USD")
```

> *"Abstraction lets me define a contract — a set of methods any implementation must provide. Abstract Base Classes enforce this: any class inheriting from my ABC must implement all abstract methods or Python raises a `TypeError` at instantiation. This is invaluable in large teams — I define the interface, others implement it, and I know they'll all work the same way from the outside."*

---

## staticmethod vs classmethod vs instance method

Think of a class as a restaurant:
- **Instance method** = waiter who knows which specific table they're serving (`self`)
- **Class method** = manager who changes restaurant-wide settings (`cls`)
- **Static method** = a calculator tool — useful in the restaurant, doesn't need to know any table

```python
class Restaurant:
    discount = 10

    def __init__(self, name, order):
        self.name = name
        self.order = order

    def get_bill(self):                      # instance method — needs self
        return f"{self.name}'s order: {self.order}"

    @classmethod
    def change_discount(cls, new_discount):  # class method — changes for ALL
        cls.discount = new_discount

    @staticmethod
    def is_valid_order(item):               # static — just utility, no self/cls
        return isinstance(item, str) and len(item) > 0

r = Restaurant("Raj", "Biryani")
print(r.get_bill())
Restaurant.change_discount(15)
print(Restaurant.is_valid_order("Biryani"))    # True
```

> *"Instance methods use `self` and operate on specific object data. Class methods use `cls` and operate on class-level data — I use them for factory methods or when I need to change something affecting all instances. Static methods take neither — they're utility functions that logically belong to the class but don't need access to any instance or class data."*

---

# ═══════════════════════════════════════
# 🟡 PART 2: INTERMEDIATE
# ═══════════════════════════════════════

---

# 📖 MODULE 6: GENERATORS & ITERATORS

---

## The Conveyor Belt vs Warehouse Analogy

- **List** = warehouse — store ALL items first, then process. Uses massive memory.
- **Generator** = conveyor belt — process one at a time. Only next item appears when ready.

```python
# List — loads everything into memory
def get_squares_list(n):
    result = []
    for i in range(n):
        result.append(i ** 2)
    return result

# Generator — produces one at a time using yield
def get_squares_gen(n):
    for i in range(n):
        yield i ** 2      # pauses, remembers state, resumes on next call

# For 10 million numbers:
squares_list = get_squares_list(10_000_000)   # ~400MB RAM
squares_gen  = get_squares_gen(10_000_000)    # ~100 bytes

# Generator expression (one-liner)
squares = (x**2 for x in range(10_000_000))  # lazy, memory efficient
```

**`yield` vs `return`:**
- `return` exits the function completely
- `yield` pauses it, saves state, resumes from there on next call

```python
def countdown(n):
    while n > 0:
        yield n         # pauses here, returns n to caller
        n -= 1          # resumes here on next call

for num in countdown(5):
    print(num)   # 5 4 3 2 1

# Manual control
gen = countdown(3)
print(next(gen))   # 3
print(next(gen))   # 2
print(next(gen))   # 1
# next(gen)        # StopIteration
```

> *"Generators are functions that use `yield` instead of `return`. They produce values lazily — one at a time — instead of computing everything upfront. Critical for memory efficiency with large datasets. I've used generators when reading large CSV files, streaming database records, or processing API responses in batches."*

---

## Custom Iterator — `__iter__` and `__next__`

```python
class NumberRange:
    """Custom iterator — iterate from start to end"""

    def __init__(self, start, end):
        self.start = start
        self.end = end
        self.current = start

    def __iter__(self):        # makes object iterable
        return self            # returns iterator object (itself)

    def __next__(self):        # returns next value
        if self.current > self.end:
            raise StopIteration   # signals end of iteration
        value = self.current
        self.current += 1
        return value

r = NumberRange(1, 5)
for num in r:
    print(num)   # 1 2 3 4 5

print(list(NumberRange(1, 5)))   # [1, 2, 3, 4, 5]
```

---

## ⚠️ Common Python Traps — Asked in Interviews

### Mutable Default Argument

```python
# WRONG — the list is created ONCE at function definition
def add_to_list(item, lst=[]):
    lst.append(item)
    return lst

print(add_to_list("a"))    # ['a']
print(add_to_list("b"))    # ['a', 'b']  ← NOT ['b']!
print(add_to_list("c"))    # ['a', 'b', 'c'] ← same list grows!

# RIGHT — use None as sentinel
def add_to_list(item, lst=None):
    if lst is None:
        lst = []           # new list created each call
    lst.append(item)
    return lst
```

> *"Default argument values are evaluated once when the function is defined, not each call. A mutable default like a list is shared across all calls. Fix: use `None` as default and create the mutable object inside the function body."*

### `is` vs `==`

```python
a = [1, 2, 3]
b = [1, 2, 3]
c = a

print(a == b)    # True  — same values
print(a is b)    # False — different objects in memory
print(a is c)    # True  — same object (c points to a)

# Common mistake with strings (Python caches some)
x = "hello"
y = "hello"
print(x is y)   # True  — cached (don't rely on this!)
print(x == y)   # True  — always use == for value comparison
```

### `*args` and `**kwargs`

```python
def flexible(*args, **kwargs):
    print(f"Positional: {args}")      # tuple
    print(f"Keyword: {kwargs}")       # dict

flexible(1, 2, 3, name="Raj", age=30)
# Positional: (1, 2, 3)
# Keyword: {'name': 'Raj', 'age': 30}

# Unpacking
def add(a, b, c):
    return a + b + c

numbers = [1, 2, 3]
print(add(*numbers))   # 6 — unpacks list into positional args

config = {"a": 1, "b": 2, "c": 3}
print(add(**config))   # 6 — unpacks dict into keyword args
```

---

## Method Chaining — Fluent Interface

```python
class QueryBuilder:
    def __init__(self, table):
        self.table = table
        self._conditions = []
        self._limit = None
        self._order = None

    def where(self, condition):
        self._conditions.append(condition)
        return self       # returning self enables chaining

    def order_by(self, column):
        self._order = column
        return self

    def limit(self, n):
        self._limit = n
        return self

    def build(self):
        query = f"SELECT * FROM {self.table}"
        if self._conditions:
            query += " WHERE " + " AND ".join(self._conditions)
        if self._order:
            query += f" ORDER BY {self._order}"
        if self._limit:
            query += f" LIMIT {self._limit}"
        return query

query = (QueryBuilder("employees")
         .where("department = 'IT'")
         .where("salary > 50000")
         .order_by("name")
         .limit(10)
         .build())

print(query)
# SELECT * FROM employees WHERE department = 'IT' AND salary > 50000 ORDER BY name LIMIT 10
```

---

# ═══════════════════════════════════════
# 🟠 PART 3: ADVANCED
# ═══════════════════════════════════════

---

# 📖 MODULE 7: CLOSURES — The Foundation of Everything Advanced

---

## 🧠 What is a Closure — Built From Scratch

**Step 1 — Functions are first-class objects:**

```python
def greet(name):
    return f"Hello {name}"

say_hello = greet           # assign to variable
say_hello("Raj")            # Hello Raj

functions = [greet, print]  # store in list
functions[0]("Priya")       # Hello Priya

def execute(func, arg):     # pass as argument
    return func(arg)

execute(greet, "Anita")     # Hello Anita
```

**Step 2 — Functions can return functions (closure):**

```python
def make_multiplier(n):
    def multiply(x):         # inner function
        return x * n         # n is "closed over" — remembered
    return multiply           # return the function itself (not result)

double = make_multiplier(2)  # n=2 is captured forever
triple = make_multiplier(3)  # n=3 is captured forever

print(double(5))    # 10
print(triple(5))    # 15
```

**Step 3 — The full closure:**

```python
def make_counter():
    count = 0              # lives in make_counter's scope

    def increment():
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

---

## 🔬 The Three Conditions for a Closure

```
1. A nested function (function inside function)
2. The inner function references a variable from outer scope
3. The outer function RETURNS the inner function
```

```python
# Inspect what Python captures
def make_adder(n):
    def add(x):
        return x + n
    return add

add5 = make_adder(5)
print(add5.__closure__)                        # (<cell at 0x...>,)
print(add5.__closure__[0].cell_contents)       # 5 ← the captured value
print(add5.__code__.co_freevars)               # ('n',) ← captured variable name
```

---

## ⚠️ Classic Closure Trap — Loop Variable Capture

```python
# WRONG — all functions capture the SAME variable i (by reference)
functions = []
for i in range(5):
    def fn():
        return i           # captures i by reference, not value
    functions.append(fn)

print([f() for f in functions])   # [4, 4, 4, 4, 4] ← all see final i=4!

# RIGHT 1 — capture by value using default argument
functions = []
for i in range(5):
    def fn(x=i):           # x=i captures current value of i
        return x
    functions.append(fn)

print([f() for f in functions])   # [0, 1, 2, 3, 4] ✅

# RIGHT 2 — use a closure factory
functions = []
for i in range(5):
    def make_fn(x):
        def fn():
            return x       # x is a new variable each call
        return fn
    functions.append(make_fn(i))

print([f() for f in functions])   # [0, 1, 2, 3, 4] ✅
```

---

## `nonlocal` vs `global`

```python
x = "global"

def outer():
    x = "outer"

    def inner():
        nonlocal x          # refers to outer's x (one scope up)
        x = "modified by inner"

    inner()
    print(x)   # modified by inner — nonlocal changed outer's x

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

> *"A closure is a function that remembers variables from its enclosing scope even after that scope finishes executing. Three things are needed: a nested function, the inner referencing an outer variable, and the outer returning the inner. Python stores captured variables in `__closure__`. A classic gotcha is loop variable capture — all closures in a loop capture the same variable by reference, seeing the final value."*

---

## Real-World Closure Use Cases

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
salary_validator = make_validator(0, 10_000_000)

print(age_validator(25))          # True
print(age_validator(15))          # False
print(salary_validator(500000))   # True
```

---

# 📖 MODULE 8: DECORATORS — Complete Deep Dive

---

## 🧠 Why Decorators Exist

```python
# Problem: 10 API functions all need logging
def get_users():
    print("LOG: get_users called")      # repeated in every function!
    return ["Raj", "Priya"]

def get_orders():
    print("LOG: get_orders called")     # duplicated!
    return [101, 102]
# Change log format → edit 10 functions. Violates DRY.
```

```python
# WITH decorator — one place, applied everywhere
import functools

def log(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        print(f"LOG: {func.__name__} called")
        return func(*args, **kwargs)
    return wrapper

@log
def get_users(): return ["Raj", "Priya"]

@log
def get_orders(): return [101, 102]
# Change log format once → all 10 functions updated
```

---

## 🔬 Decorator Anatomy — Every Line

```python
import functools

def my_decorator(func):           # 1. takes a function as argument

    @functools.wraps(func)        # 2. preserves original function metadata
    def wrapper(*args, **kwargs): # 3. accepts any arguments

        print("Before")           # 4. code runs BEFORE original function

        result = func(*args, **kwargs)  # 5. call original function

        print("After")            # 6. code runs AFTER

        return result             # 7. return original result

    return wrapper                # 8. return wrapper (not its result!)
```

**Why `@functools.wraps(func)`?**

```python
def bad_decorator(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@bad_decorator
def greet(name):
    """Says hello"""
    return f"Hello {name}"

print(greet.__name__)   # wrapper     ← WRONG, lost original name
print(greet.__doc__)    # None        ← WRONG, lost docstring

# @functools.wraps preserves __name__, __doc__, __module__
```

---

## Decorator with Arguments — Three-Level Pattern

```python
# Regular:       @decorator
# With args:     @decorator(arg)   ← needs one extra layer

def repeat(times):                        # LEVEL 1 — accepts decorator args
    def decorator(func):                  # LEVEL 2 — accepts the function
        @functools.wraps(func)
        def wrapper(*args, **kwargs):     # LEVEL 3 — the actual wrapper
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
```

---

## Production Decorators

### Timing

```python
import time

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
def process_data(n):
    return sum(range(n))

process_data(10_000_000)  # process_data took 0.3421s
```

### Retry

```python
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

### Cache / Memoize

```python
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
    if n <= 1: return n
    return fibonacci(n-1) + fibonacci(n-2)

# Python built-in version:
from functools import lru_cache

@lru_cache(maxsize=128)
def fibonacci(n):
    if n <= 1: return n
    return fibonacci(n-1) + fibonacci(n-2)
```

### Class-Based Decorator

```python
class RateLimit:
    """Limits function to N calls per minute"""
    def __init__(self, max_calls):
        self.max_calls = max_calls
        self.calls = []

    def __call__(self, func):         # makes instance callable as decorator
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

@decorator_A    # applied SECOND (outer)
@decorator_B    # applied FIRST (inner)
def my_func():
    print("Function runs")

my_func()
# A before    ← outermost, runs first
# B before
# Function runs
# B after
# A after     ← outermost, closes last

# Equivalent to: my_func = decorator_A(decorator_B(my_func))
```

> *"Decorators stack bottom to top in application, but execute top to bottom. The bottom decorator is innermost — runs last before the function. I always reason about stacking by thinking: closest to the function definition = innermost = runs last."*

---

# 📖 MODULE 9: MRO & C3 LINEARIZATION

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
    def speak(self): return "Animal speaks"

class Dog(Animal):
    def speak(self): return "Woof"

class Cat(Animal):
    def speak(self): return "Meow"

class Hybrid(Dog, Cat): pass

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

**Plain English rule:**
```
1. Start with the class itself
2. Look at first element of each parent's MRO list
3. Pick the first element that does NOT appear in the TAIL
   (tail = everything except the first element) of any other list
4. Add it to result, remove it from all lists
5. Repeat until all lists empty
```

**Step-by-step:**

```python
class A: pass
class B(A): pass
class C(A): pass
class D(B, C): pass

# L(A) = [A, object]
# L(B) = [B, A, object]
# L(C) = [C, A, object]
# L(D) = D + merge([B,A,obj], [C,A,obj], [B,C])

# Step 1: First elements = B, C, B
#         Is B in TAIL of any list? [B,C] → tail is [C], B not there ✅
#         Pick B → result=[D,B], remove B from all lists
#         Now: merge([A,obj], [C,A,obj], [C])

# Step 2: First elements = A, C, C
#         Is A in TAIL of any list? [C,A,obj] → tail=[A,obj], YES ❌
#         Skip A. Try C. Is C in any tail? No ✅
#         Pick C → result=[D,B,C], remove C
#         Now: merge([A,obj], [A,obj], [])

# Step 3: Pick A → result=[D,B,C,A], remove A
# Step 4: Pick object → result=[D,B,C,A,object]

print(D.__mro__)
# (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)
```

---

## How to Explain C3 in Interview

> *"C3 linearization is the algorithm Python uses to compute Method Resolution Order. The key rule: take the leftmost class that doesn't appear in the tail of any other inheritance list — 'tail' means everything except the first element. This ensures three guarantees: a class always appears before its parents, left-to-right parent order is preserved, and no class appears twice. I don't compute it manually in production — I check `ClassName.__mro__` — but understanding the algorithm helps me design inheritance hierarchies that won't create ambiguity."*

**Counter-question: "What if C3 can't resolve?"**

```python
# Inconsistent hierarchy — Python raises TypeError at class definition
# class Broken(A, B): pass  ← if this contradicts the hierarchy
# TypeError: Cannot create a consistent MRO
```

> *"If C3 finds a contradiction — usually the parent order contradicts the inheritance hierarchy — Python raises a `TypeError` at class definition time. It fails fast and loudly. The fix is to reorder parents or redesign the hierarchy."*

---

## `super()` Follows MRO — Critical

```python
class A:
    def method(self):
        print("A")
        super().method()    # next in MRO, not necessarily object

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
# D → B → C → A
# MRO: D → B → C → A → object

# super() in B doesn't call A — it calls C (next in D's MRO)!
```

> *"`super()` means 'call the next class in the MRO', not 'call my parent'. This enables cooperative multiple inheritance. For it to work, every class in the chain must call `super()`, otherwise the chain breaks."*

---

# 📖 MODULE 10: LISKOV SUBSTITUTION PRINCIPLE

---

## 🧠 Core Idea

**"Anywhere you use a parent class, you should be able to swap in a child class and everything still works."**

---

## ✅ LSP Satisfied

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

def print_bird_info(bird: Bird):
    print(bird.describe())          # works with ANY Bird subclass

print_bird_info(Sparrow("Jack"))    # works
print_bird_info(Eagle("Eddie"))     # works
```

---

## ❌ LSP Violated — Classic Example

```python
class Bird:
    def fly(self): return "Flying!"

class Penguin(Bird):
    def fly(self):
        raise NotImplementedError("Penguins can't fly!")   # VIOLATION

def make_bird_fly(bird: Bird):
    return bird.fly()   # caller expects this to always work

make_bird_fly(Penguin())   # CRASHES — LSP violated
```

**The Fix:**

```python
from abc import ABC, abstractmethod

class Animal(ABC):
    def __init__(self, name): self.name = name

    @abstractmethod
    def move(self): pass

class FlyingBird(Animal):
    def move(self): return f"{self.name} is flying"
    def fly(self): return f"{self.name} soars through the air"

class NonFlyingBird(Animal):
    def move(self): return f"{self.name} is walking"

class Eagle(FlyingBird): pass
class Penguin(NonFlyingBird): pass

def make_animal_move(animal: Animal):
    return animal.move()   # works for ALL animals now

make_animal_move(Eagle("Eddie"))    # Eddie is flying
make_animal_move(Penguin("Pete"))   # Pete is walking
```

---

## The Four Rules of LSP

```python
class DataProcessor:
    def process(self, data: list) -> dict:
        return {"processed": len(data)}

# RULE 1: Don't strengthen preconditions (accept at least what parent accepts)
class BadProcessor1(DataProcessor):
    def process(self, data: list) -> dict:
        if len(data) < 10:                    # VIOLATION — more restrictive
            raise ValueError("Need 10+ items")
        return {"processed": len(data)}

# RULE 2: Don't weaken postconditions (return at least what parent promises)
class BadProcessor2(DataProcessor):
    def process(self, data: list):            # VIOLATION — removed dict return
        return "done"

# RULE 3: Don't throw new exceptions parent doesn't throw
class BadProcessor3(DataProcessor):
    def process(self, data: list) -> dict:
        raise RuntimeError("Unexpected!")     # VIOLATION

# CORRECT — extends without breaking contract
class GoodProcessor(DataProcessor):
    def process(self, data: list) -> dict:
        result = super().process(data)
        result["extra"] = "additional info"   # adds more, doesn't change contract
        return result
```

> **Complete interview answer:**
> *"LSP says a subclass should be substitutable for its parent without breaking the program. Four things must hold: subclasses can't strengthen preconditions — must accept everything parent accepts; can't weaken postconditions — must return what parent promises; shouldn't throw new exceptions the parent doesn't; and must maintain parent invariants. Classic violation: Penguin extending Bird with `fly()` — breaks any code calling `bird.fly()` on a Bird reference. Fix: better abstraction — separate FlyingBird and NonFlyingBird. I think about LSP when designing: ask 'would a caller using the parent break if given this child?' If yes, the hierarchy is wrong."*

---

# 📖 MODULE 11: METACLASSES

---

## 🧠 Classes Are Objects Too

```python
class Dog: pass

print(type(Dog))    # <class 'type'>
print(type(42))     # <class 'int'>
print(type(type))   # <class 'type'>  ← type is its own metaclass

# A metaclass is the class of a class
# Normal: instance = Class(...)        ← Class creates instances
# Meta:   Class    = Metaclass(...)   ← Metaclass creates Classes
```

---

## How Classes Are Created Under the Hood

```python
# When you write this:
class MyClass:
    x = 10
    def greet(self): return "Hello"

# Python internally does:
MyClass = type("MyClass", (), {"x": 10, "greet": lambda self: "Hello"})
# type(name, bases, namespace_dict)
```

---

## Creating a Custom Metaclass

```python
class MyMeta(type):                          # inherit from type

    def __new__(mcs, name, bases, namespace):
        # mcs       = the metaclass (MyMeta)
        # name      = class being created ("MyClass")
        # bases     = tuple of parent classes
        # namespace = dict of class attributes/methods

        print(f"Creating class: {name}")
        namespace['created_by'] = 'MyMeta'   # inject attribute

        return super().__new__(mcs, name, bases, namespace)

    def __init__(cls, name, bases, namespace):
        super().__init__(name, bases, namespace)

class MyClass(metaclass=MyMeta):
    x = 10

print(MyClass.created_by)    # MyMeta — we injected this!
```

---

## Real-World Metaclass Use Cases

### Use Case 1: Enforce Method Implementation

```python
class InterfaceMeta(type):
    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
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
print(db1 is db2)   # True — "Connecting..." printed only ONCE
```

### Use Case 3: Auto-Register Subclasses (Plugin System)

```python
class PluginMeta(type):
    registry = {}

    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        if bases:
            mcs.registry[name] = cls
        return cls

class Plugin(metaclass=PluginMeta):
    def run(self): pass

class PDFPlugin(Plugin):
    def run(self): return "Processing PDF"

class ExcelPlugin(Plugin):
    def run(self): return "Processing Excel"

print(PluginMeta.registry)
# {'PDFPlugin': <class 'PDFPlugin'>, 'ExcelPlugin': <class 'ExcelPlugin'>}

def run_plugin(name):
    plugin_cls = PluginMeta.registry.get(name)
    if plugin_cls:
        return plugin_cls().run()

print(run_plugin("PDFPlugin"))    # Processing PDF
```

### Use Case 4: Auto-Add Logging to All Methods

```python
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
    def process(self): return "processing"
    def save(self): return "saving"

s = Service()
s.process()
# [LOG] Calling: process
# [LOG] Done: process
```

---

## `__new__` vs `__init__` vs `__init_subclass__`

```python
class MyClass:

    def __new__(cls, *args, **kwargs):
        print(f"1. __new__ — creating instance")
        instance = super().__new__(cls)   # actually creates the object
        return instance                    # MUST return instance

    def __init__(self, name):
        print(f"2. __init__ — setting up {name}")
        self.name = name

    def __init_subclass__(cls, **kwargs):
        # Called when THIS class is subclassed
        super().__init_subclass__(**kwargs)
        print(f"3. __init_subclass__ — {cls.__name__} subclassed MyClass")

class Child(MyClass): pass   # triggers __init_subclass__

obj = MyClass("Raj")
# 1. __new__ — creating instance
# 2. __init__ — setting up Raj
```

**When to use each:**
- `__new__` → Singleton, subclassing immutables (`int`, `str`, `tuple`)
- `__init__` → Normal initialization (99% of cases)
- `__init_subclass__` → Lightweight alternative to metaclass for registering subclasses

> **Interview answer for metaclasses:**
> *"A metaclass is the class of a class — it controls how classes are created, just like a class controls how objects are created. The default metaclass is `type`. I create custom metaclasses by inheriting from `type` and overriding `__new__` or `__init__`. Common uses: enforcing interfaces, auto-registering subclasses, auto-injecting logging, Singleton. They're powerful but heavy — in most application code I'd reach for `__init_subclass__`, class decorators, or ABC first. ORMs like Django use metaclasses — `Model` classes automatically register with the ORM because of metaclass magic."*

---

# 📖 MODULE 12: SPECIAL/DUNDER METHODS — Complete Reference

---

## The Full Container Protocol

```python
class SmartList:
    def __init__(self, items=None):
        self._items = items or []

    def __str__(self):
        return f"SmartList({self._items})"

    def __repr__(self):
        return f"SmartList(items={self._items!r})"

    def __len__(self):               # len(smart_list)
        return len(self._items)

    def __getitem__(self, index):    # smart_list[0]
        return self._items[index]

    def __setitem__(self, index, value):   # smart_list[0] = "new"
        self._items[index] = value

    def __delitem__(self, index):    # del smart_list[0]
        del self._items[index]

    def __contains__(self, item):    # "x" in smart_list
        return item in self._items

    def __iter__(self):              # for x in smart_list
        return iter(self._items)

    def __add__(self, other):        # sl1 + sl2
        return SmartList(self._items + other._items)

    def __eq__(self, other):         # sl1 == sl2
        return self._items == other._items

    def __lt__(self, other):         # sl1 < sl2 (enables sorting!)
        return len(self) < len(other)

    def __call__(self, filter_fn):   # sl(lambda x: x > 0)
        return SmartList([x for x in self._items if filter_fn(x)])

    def __enter__(self):             # with sl as s:
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        return False    # don't suppress exceptions
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
        # Dangerous to override — can cause infinite recursion
        print(f"__getattribute__ called for: {name}")
        return super().__getattribute__(name)   # MUST call super

obj = MyClass()
obj.x = 10

print(obj.x)    # __getattribute__ → 10 (found)
print(obj.y)    # __getattribute__ → not found → __getattr__ → default_y
```

> *"`__getattr__` is the fallback — only called when normal lookup fails. Safe to override for proxy patterns. `__getattribute__` intercepts EVERY access — easy to cause infinite recursion. Always use `super().__getattribute__()` inside it."*

---

## Context Managers — `__enter__` and `__exit__`

```python
class DatabaseTransaction:
    def __init__(self, db_url):
        self.db_url = db_url

    def __enter__(self):
        print(f"Opening connection to {self.db_url}")
        self.connection = f"conn_{self.db_url}"
        return self.connection       # what 'as' variable receives

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type:
            print(f"Exception: {exc_val}. Rolling back.")
        else:
            print("Success. Committing.")
        print("Closing connection")
        return False    # False = don't suppress exception

with DatabaseTransaction("postgresql://localhost/mydb") as conn:
    print(f"Using {conn}")

# contextlib alternative:
from contextlib import contextmanager

@contextmanager
def db_transaction(url):
    print("Opening")
    try:
        yield f"conn_{url}"    # what 'as' receives
        print("Committing")
    except Exception as e:
        print(f"Rolling back: {e}")
        raise
    finally:
        print("Closing")
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
    __slots__ = ['x', 'y']    # fixed set of attributes — no __dict__
    def __init__(self, x, y):
        self.x = x
        self.y = y

import sys
rp = RegularPoint(1, 2)
sp = SlottedPoint(1, 2)

# SlottedPoint uses ~40-50% less memory per instance
# sp.z = 10    # AttributeError — slots enforced
```

> *"`__slots__` replaces per-instance `__dict__` with fixed descriptors. Saves significant memory when creating millions of small objects. Tradeoffs: no dynamic attributes, no `__weakref__` by default, more complex with multiple inheritance."*

---

# 📖 MODULE 13: SOLID PRINCIPLES

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

---

## O — Open/Closed Principle

```python
# WRONG — must modify class to add new shape
class AreaCalculator:
    def calculate(self, shape):
        if shape.type == "circle":
            return 3.14 * shape.radius ** 2
        elif shape.type == "rectangle":     # add new type? edit this class
            return shape.width * shape.height

# RIGHT — open for extension, closed for modification
class Shape(ABC):
    @abstractmethod
    def area(self): pass

class Circle(Shape):
    def area(self): return 3.14 * self.radius ** 2

class Rectangle(Shape):
    def area(self): return self.width * self.height

class Triangle(Shape):                      # new shape — NO modification needed
    def area(self): return 0.5 * self.base * self.height

def total_area(shapes):
    return sum(s.area() for s in shapes)   # works with any shape
```

---

## L — Liskov Substitution *(covered in Module 10)*

---

## I — Interface Segregation

```python
# WRONG — fat interface forces irrelevant implementations
class Worker(ABC):
    @abstractmethod
    def work(self): pass

    @abstractmethod
    def eat(self): pass      # robots don't eat!

    @abstractmethod
    def sleep(self): pass    # robots don't sleep!

class Robot(Worker):
    def work(self): return "working"
    def eat(self): raise NotImplementedError    # forced but irrelevant
    def sleep(self): raise NotImplementedError  # forced but irrelevant

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

class Robot(Workable):      # only what's relevant
    def work(self): return "working"
```

---

## D — Dependency Inversion

```python
# WRONG — high-level depends on low-level
class UserNotifier:
    def __init__(self):
        self.service = EmailService()    # hard-coded dependency!

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
notifier = UserNotifier(SMSService())     # zero code change in UserNotifier
```

---

# 📖 MODULE 14: ADVANCED PATTERNS

---

## Descriptor Protocol — How Properties Work Internally

```python
class PositiveNumber:
    """A reusable validator descriptor"""

    def __set_name__(self, owner, name):
        self.name = name
        self.private_name = f"_{name}"

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self    # accessed from class, not instance
        return getattr(obj, self.private_name, None)

    def __set__(self, obj, value):
        if not isinstance(value, (int, float)) or value <= 0:
            raise ValueError(f"{self.name} must be a positive number, got {value}")
        setattr(obj, self.private_name, value)

class Employee:
    salary = PositiveNumber()    # descriptor — reusable on any class
    age = PositiveNumber()

    def __init__(self, name, salary, age):
        self.name = name
        self.salary = salary    # triggers PositiveNumber.__set__
        self.age = age

e = Employee("Raj", 50000, 30)
# e.salary = -1000    # ValueError: salary must be a positive number
```

---

## Composition vs Inheritance

```python
# Inheritance = "IS-A" relationship
class Animal: pass
class Dog(Animal): pass      # Dog IS-A Animal

# Composition = "HAS-A" relationship — more flexible
class Engine:
    def start(self): return "Engine started"

class GPS:
    def navigate(self, dest): return f"Navigating to {dest}"

class MusicSystem:
    def play(self, song): return f"Playing {song}"

class Car:
    def __init__(self):
        self.engine = Engine()        # HAS-A Engine
        self.gps = GPS()              # HAS-A GPS
        self.music = MusicSystem()    # HAS-A MusicSystem

    def start(self): return self.engine.start()
    def go_to(self, dest): return self.gps.navigate(dest)
    def play(self, song): return self.music.play(song)

# To change GPS behavior — swap the GPS class, don't touch Car
```

> *"Prefer composition over inheritance. Inheritance creates tight coupling — changes to parent ripple through children. Composition is more flexible — I can swap components without touching the containing class. Use inheritance for genuine 'is-a' type hierarchies with shared behavior. Use composition for 'has-a' relationships or when flexibility is needed."*

---

## Design Patterns — Factory & Singleton

**Factory Pattern:**

```python
class Animal:
    def speak(self): pass

class Dog(Animal):
    def speak(self): return "Woof!"

class Cat(Animal):
    def speak(self): return "Meow!"

class AnimalFactory:
    @staticmethod
    def create(animal_type):
        animals = {"dog": Dog, "cat": Cat}
        cls = animals.get(animal_type.lower())
        if not cls:
            raise ValueError(f"Unknown animal: {animal_type}")
        return cls()

animal = AnimalFactory.create("dog")
print(animal.speak())    # Woof!
```

**Singleton Pattern:**

```python
class DatabaseConnection:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance.connected = False
        return cls._instance    # always return same instance

db1 = DatabaseConnection()
db2 = DatabaseConnection()
print(db1 is db2)    # True — same object!
```

---

# 📋 MASTER INTERVIEW Q&A

## Basic Level

**Q: What are the four pillars of OOP?**
> *"Encapsulation — bundling data and methods, controlling access. Inheritance — child class reusing parent's code. Polymorphism — same interface, different behavior. Abstraction — hiding complexity, showing only the interface. Each solves a specific problem: encapsulation protects data integrity, inheritance prevents duplication, polymorphism enables flexibility, abstraction manages complexity."*

**Q: What is the difference between a class and an object?**
> *"A class is a blueprint — defines structure and behavior but occupies no real data memory. An object is an instance — a specific thing built from that blueprint with its own data. `Car` is the class. `my_car = Car('Toyota')` is the object."*

**Q: Can you instantiate an abstract class?**
> *"No. Python raises `TypeError`. That's the point — an abstract class is a contract saying 'any concrete subclass must implement these methods.' The moment you implement all abstract methods, that subclass can be instantiated."*

**Q: What is the difference between `is` and `==`?**
> *"`==` compares values. `is` compares identity — are they literally the same object in memory? Two lists can have the same content (`==` True) but be different objects (`is` False). Common mistake: using `is` to compare strings — Python caches some strings, making `is` seem to work, but it's unreliable and semantically wrong."*

---

## Intermediate Level

**Q: What is a closure?**
> *"A closure is a function that remembers variables from its enclosing scope even after that scope finishes executing. Three conditions: nested function, inner references outer variable, outer returns inner. Classic gotcha: loop variable capture — all closures in a loop capture the same variable by reference, seeing the final value."*

**Q: What is the difference between `@staticmethod`, `@classmethod`, and instance methods?**
> *"Instance methods take `self` — operate on specific object data. Class methods take `cls` — operate on class-level data, good for factory methods. Static methods take neither — utility functions that logically belong to the class but don't need access to instance or class data."*

**Q: Why use `@functools.wraps` in decorators?**
> *"Without it, the wrapper function replaces the original's metadata — `__name__`, `__doc__`, `__module__` all become the wrapper's. This breaks documentation, debugging, and any code inspecting function names. `functools.wraps` copies all metadata from the original onto the wrapper."*

**Q: What is the mutable default argument trap?**
> *"Default values are evaluated once at function definition, not each call. A mutable default like a list is shared across all calls. Fix: use `None` as default, create the mutable object inside the function body."*

---

## Advanced Level

**Q: What is a metaclass?**
> *"A metaclass is the class of a class — it controls how classes are created, just like a class controls how objects are created. Default metaclass is `type`. I create custom metaclasses by inheriting from `type`. Common uses: enforcing interfaces, auto-registering subclasses, Singleton. They're powerful but heavy — I prefer `__init_subclass__`, class decorators, or ABC first."*

**Q: How does Python resolve method calls in multiple inheritance?**
> *"Python uses MRO — Method Resolution Order — computed by C3 linearization. The rule: take the leftmost class that doesn't appear in the tail of any other inheritance list. Guarantees classes appear before parents, left-to-right order preserved, no duplicates. Check with `ClassName.__mro__`."*

**Q: `__new__` vs `__init__`?**
> *"`__new__` creates the object — called first, receives the class, must return an instance. `__init__` initializes it — called second, receives the created instance. Override `__new__` for Singleton, subclassing immutables like `int` or `tuple`. For 99% of cases, only `__init__` is needed."*

**Q: Composition vs Inheritance — when to use which?**
> *"Inheritance is 'is-a'. Composition is 'has-a'. Prefer composition — inheritance creates tight coupling, changes to parent ripple through all children. Composition is flexible — swap components without touching the container. Use inheritance for genuine type hierarchies with shared behavior. Use composition for flexibility or when you'd inherit just to reuse a method."*

**Q: What is the Descriptor Protocol?**
> *"Descriptors define `__get__`, `__set__`, or `__delete__`. When you access an attribute, Python checks if the class has a descriptor for that name and delegates to it. This is how `@property`, `@classmethod`, `@staticmethod`, and Django model fields work internally. I use custom descriptors for reusable validation logic across multiple class attributes."*

---

# 🗺️ STUDY ROADMAP

| Week | Focus | Action |
|------|-------|--------|
| Week 1 | Part 1 Basics (Modules 1-5) | Explain each concept aloud without notes |
| Week 2 | Part 2 Intermediate (Modules 6-7) | Build a project using closures + generators |
| Week 3 | Part 3 Advanced (Modules 8-14) | Build a mini-framework using metaclasses + decorators |
| Week 4 | All Q&A | Answer every question timed — 45–90 seconds each |

---

# 💡 THE GOLDEN INTERVIEW FORMULA

Every technical answer = **What + Why + Example + Production**

- **What** — one sentence definition
- **Why** — what problem it solves
- **Example** — concrete code or scenario
- **Production** — how you've used or would use it at work

> Never just define. Always connect to real impact.
> Interviewers at Manager level want to see you've *applied* these concepts, not just memorized them.
