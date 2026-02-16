PYTHON MASTERY: BASICS TO ADVANCED
Complete Interview Preparation Guide
TABLE OF CONTENTS
Python Fundamentals
Data Types & Structures Deep Dive
Tricky Concepts (Mutability, Copying, etc.)
Functions & Decorators
OOP (Object-Oriented Programming)
Advanced Topics
Tricky Interview Questions
Common Pitfalls & Gotchas
1. PYTHON FUNDAMENTALS
1.1 Variables & Memory Management
How Python Stores Variables
# Python uses reference semantics
a = 10
b = a  # b points to same object as a

print(id(a))  # Memory address
print(id(b))  # Same memory address!

# But for mutable objects...
list1 = [1, 2, 3]
list2 = list1  # Both point to SAME list
list2.append(4)
print(list1)  # [1, 2, 3, 4] - MODIFIED!

# Interview Question: Why?
# Answer: Lists are mutable. list2 is a reference, not a copy.
Small Integer Caching
# Python caches small integers (-5 to 256)
a = 10
b = 10
print(a is b)  # True (same object!)

# But not for large integers
x = 1000
y = 1000
print(x is y)  # False (different objects)
print(x == y)  # True (same value)

# TRICKY INTERVIEW QUESTION:
a = 256
b = 256
print(a is b)  # True

a = 257
b = 257
print(a is b)  # False (in most cases)

# Why? Python caches integers from -5 to 256 for optimization
1.2 Data Types Deep Dive
Immutable vs Mutable
# IMMUTABLE: int, float, str, tuple, frozenset, bool
# Once created, cannot be changed

# Example: Strings are immutable
s = "hello"
print(id(s))
s = s + " world"  # Creates NEW string
print(id(s))  # Different memory address!

# MUTABLE: list, dict, set
# Can be changed in place

lst = [1, 2, 3]
print(id(lst))
lst.append(4)  # Modifies SAME object
print(id(lst))  # Same memory address!

# CRITICAL INTERVIEW CONCEPT:
def add_item(lst=[]):  # DANGEROUS DEFAULT!
    lst.append(1)
    return lst

print(add_item())  # [1]
print(add_item())  # [1, 1] - WTF?!
print(add_item())  # [1, 1, 1] - Same list every time!

# Why? Default argument is created ONCE when function is defined
# Solution:
def add_item_correct(lst=None):
    if lst is None:
        lst = []
    lst.append(1)
    return lst
2. TRICKY CONCEPTS - DEEP DIVE
2.1 Shallow Copy vs Deep Copy
The Problem
import copy

# Original list with nested structure
original = [1, 2, [3, 4], 5]

# Assignment (NOT a copy!)
assigned = original
assigned.append(6)
print(original)  # [1, 2, [3, 4], 5, 6] - MODIFIED!

# Shallow Copy - copies outer container, references inner objects
shallow = original.copy()  # or list(original) or original[:]
shallow.append(7)
print(original)  # [1, 2, [3, 4], 5, 6] - Not affected

# But watch this:
shallow[2].append(99)  # Modify nested list
print(original)  # [1, 2, [3, 4, 99], 5, 6] - AFFECTED!
print(shallow)   # [1, 2, [3, 4, 99], 5, 6, 7] - Both changed!

# Why? Shallow copy only copies references to nested objects

# Deep Copy - recursively copies everything
deep = copy.deepcopy(original)
deep[2].append(100)
print(original)  # [1, 2, [3, 4, 99], 5, 6] - NOT affected
print(deep)      # [1, 2, [3, 4, 99, 100], 5, 6, 7] - Only this changes
Visual Explanation
# Shallow Copy:
original = [[1, 2], [3, 4]]
shallow = original.copy()

# Memory layout:
# original -> [ref1, ref2]
# shallow  -> [ref1, ref2]  (same references!)
#              â†“     â†“
#            [1,2] [3,4]

# Deep Copy:
deep = copy.deepcopy(original)

# Memory layout:
# original -> [ref1, ref2]
#              â†“     â†“
#            [1,2] [3,4]
#
# deep     -> [ref3, ref4]
#              â†“     â†“
#            [1,2] [3,4]  (completely separate!)
Interview Questions on Copying
# Q1: What will this print?
a = [1, 2, 3]
b = a
c = a.copy()
d = a[:]

a.append(4)
print(b)  # [1, 2, 3, 4] - reference to same list
print(c)  # [1, 2, 3] - shallow copy
print(d)  # [1, 2, 3] - shallow copy (slicing creates new list)

# Q2: What about nested structures?
original = [[1, 2], [3, 4]]
copy1 = original.copy()
copy2 = copy.deepcopy(original)

original[0].append(99)
print(copy1)  # [[1, 2, 99], [3, 4]] - affected!
print(copy2)  # [[1, 2], [3, 4]] - not affected

# Q3: Tricky tuple question
t1 = ([1, 2], 3)
t2 = t1  # reference
t3 = copy.copy(t1)  # shallow copy
t4 = copy.deepcopy(t1)  # deep copy

t1[0].append(99)
print(t1)  # ([1, 2, 99], 3)
print(t2)  # ([1, 2, 99], 3) - reference
print(t3)  # ([1, 2, 99], 3) - shallow copy, list is referenced!
print(t4)  # ([1, 2], 3) - deep copy, completely separate
2.2 Tuples - Immutable But Tricky!
Understanding Tuple Immutability
# Tuples are immutable - you cannot change the tuple itself
t = (1, 2, 3)
# t[0] = 10  # TypeError: 'tuple' object does not support item assignment

# But if tuple contains mutable objects...
t = ([1, 2], 3)
t[0].append(99)  # This WORKS!
print(t)  # ([1, 2, 99], 3)

# Why? The tuple is still immutable (references don't change)
# But the LIST inside is mutable!

# Visual:
# t -> (ref_to_list, 3)
#       â†“
#      [1, 2]  <- This can change
#
# The reference (ref_to_list) cannot change, but the list it points to can!

# Interview Question:
t = (1, 2, 3)
# Can you modify this tuple?
# Answer: No, but you can create a new tuple
t = t + (4,)  # Creates NEW tuple
print(t)  # (1, 2, 3, 4)
Tuple vs List - When to Use What?
# Use Tuple When:
# 1. Data should not change
coordinates = (10.5, 20.3)  # x, y coordinates

# 2. Dictionary keys (lists can't be keys!)
locations = {
    (0, 0): "Origin",
    (1, 1): "Diagonal"
}

# 3. Function returns multiple values
def get_user():
    return ("John", 25, "john@example.com")

name, age, email = get_user()  # Tuple unpacking

# 4. Performance (tuples are slightly faster)

# Use List When:
# 1. Data needs to change
shopping_list = ["milk", "eggs"]
shopping_list.append("bread")

# 2. Need list methods (append, remove, etc.)
Tricky Tuple Questions
# Q1: Single element tuple
t1 = (1)      # This is an int!
t2 = (1,)     # This is a tuple
print(type(t1))  # <class 'int'>
print(type(t2))  # <class 'tuple'>

# Q2: Tuple unpacking
a, b = 1, 2  # Creates tuple (1, 2) then unpacks
a, b = b, a  # Swap in one line! Creates (b, a) then unpacks

# Q3: * operator
a, *b, c = [1, 2, 3, 4, 5]
print(a)  # 1
print(b)  # [2, 3, 4] - gets the "rest"
print(c)  # 5

# Q4: Tuple with one mutable element
t = ([1, 2],)
# Can we hash this tuple? (for use as dict key)
# hash(t)  # TypeError: unhashable type: 'list'
# Tuples are only hashable if all elements are hashable
2.3 Decorators - Complete Guide
What is a Decorator?
# A decorator is a function that takes a function and returns a new function
# It "wraps" the original function to add functionality

# Simple example:
def my_decorator(func):
    def wrapper():
        print("Before function call")
        func()
        print("After function call")
    return wrapper

@my_decorator
def say_hello():
    print("Hello!")

say_hello()
# Output:
# Before function call
# Hello!
# After function call

# What @ does:
# say_hello = my_decorator(say_hello)
Decorators with Arguments
# Problem: Decorated function takes arguments
def my_decorator(func):
    def wrapper(*args, **kwargs):  # Accept any arguments
        print(f"Calling {func.__name__}")
        result = func(*args, **kwargs)
        print(f"Function returned: {result}")
        return result
    return wrapper

@my_decorator
def add(a, b):
    return a + b

result = add(3, 5)
# Output:
# Calling add
# Function returned: 8
Preserving Function Metadata
from functools import wraps

# Without @wraps:
def bad_decorator(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@bad_decorator
def my_function():
    """This is my function"""
    pass

print(my_function.__name__)  # wrapper (wrong!)
print(my_function.__doc__)   # None (lost!)

# With @wraps:
def good_decorator(func):
    @wraps(func)  # Preserves metadata
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@good_decorator
def my_function():
    """This is my function"""
    pass

print(my_function.__name__)  # my_function (correct!)
print(my_function.__doc__)   # This is my function (preserved!)
Common Decorator Patterns
from functools import wraps
import time

# 1. Timing Decorator
def timer(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(f"{func.__name__} took {end - start:.4f} seconds")
        return result
    return wrapper

@timer
def slow_function():
    time.sleep(1)
    return "Done"

# 2. Logging Decorator
def logger(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__} with args={args}, kwargs={kwargs}")
        result = func(*args, **kwargs)
        print(f"{func.__name__} returned {result}")
        return result
    return wrapper

@logger
def add(a, b):
    return a + b

# 3. Retry Decorator
def retry(times=3):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for i in range(times):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    print(f"Attempt {i+1} failed: {e}")
                    if i == times - 1:
                        raise
        return wrapper
    return decorator

@retry(times=3)
def unreliable_function():
    import random
    if random.random() < 0.7:
        raise Exception("Random failure")
    return "Success"

# 4. Authentication Decorator (FastAPI style)
def require_auth(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        # Check if user is authenticated
        if not is_authenticated():
            raise Exception("Unauthorized")
        return func(*args, **kwargs)
    return wrapper

# 5. Caching Decorator
def cache(func):
    cached_results = {}
    
    @wraps(func)
    def wrapper(*args):
        if args in cached_results:
            print(f"Returning cached result for {args}")
            return cached_results[args]
        
        result = func(*args)
        cached_results[args] = result
        return result
    return wrapper

@cache
def expensive_computation(n):
    print(f"Computing for {n}...")
    time.sleep(1)
    return n * n

print(expensive_computation(5))  # Computes
print(expensive_computation(5))  # Returns cached
Decorator with Parameters
# Decorator that takes arguments
def repeat(times):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(times):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(times=3)
def say_hello():
    print("Hello!")

say_hello()
# Output:
# Hello!
# Hello!
# Hello!

# How it works:
# 1. repeat(3) returns decorator function
# 2. decorator(say_hello) returns wrapper function
# 3. say_hello = wrapper
Class-Based Decorators
class CountCalls:
    def __init__(self, func):
        self.func = func
        self.count = 0
    
    def __call__(self, *args, **kwargs):
        self.count += 1
        print(f"Call {self.count} of {self.func.__name__}")
        return self.func(*args, **kwargs)

@CountCalls
def say_hello():
    print("Hello!")

say_hello()  # Call 1 of say_hello
say_hello()  # Call 2 of say_hello
Stacking Decorators
@decorator1
@decorator2
@decorator3
def my_function():
    pass

# Equivalent to:
# my_function = decorator1(decorator2(decorator3(my_function)))

# Example:
@timer
@logger
def add(a, b):
    return a + b

# First logger wraps add, then timer wraps the result
2.4 Mutability Deep Dive
Mutable Default Arguments - The Biggest Gotcha
# DANGEROUS!
def append_to_list(item, my_list=[]):
    my_list.append(item)
    return my_list

print(append_to_list(1))  # [1]
print(append_to_list(2))  # [1, 2] - WAT?!
print(append_to_list(3))  # [1, 2, 3] - Same list!

# Why? Default argument is created ONCE when function is defined
# It's shared across all calls!

# Solution:
def append_to_list_correct(item, my_list=None):
    if my_list is None:
        my_list = []
    my_list.append(item)
    return my_list

print(append_to_list_correct(1))  # [1]
print(append_to_list_correct(2))  # [2] - New list each time!
Mutability in Loops
# Problem: Creating list of functions
functions = []
for i in range(5):
    functions.append(lambda: i)  # Captures reference to i

# What do you think this prints?
for func in functions:
    print(func())  # 4, 4, 4, 4, 4 - All print 4!

# Why? Lambda captures reference to i, not its value
# When called, i = 4 (final value)

# Solution 1: Default argument (creates binding)
functions = []
for i in range(5):
    functions.append(lambda x=i: x)  # x=i captures VALUE

for func in functions:
    print(func())  # 0, 1, 2, 3, 4 - Correct!

# Solution 2: Use functools.partial
from functools import partial

def print_value(x):
    return x

functions = [partial(print_value, i) for i in range(5)]
Mutability in Class Attributes
# DANGEROUS!
class MyClass:
    shared_list = []  # Class attribute (shared by ALL instances)
    
    def add_item(self, item):
        self.shared_list.append(item)

obj1 = MyClass()
obj2 = MyClass()

obj1.add_item(1)
print(obj2.shared_list)  # [1] - Shared!

# Solution: Use instance attributes
class MyClass:
    def __init__(self):
        self.my_list = []  # Instance attribute (unique per instance)
    
    def add_item(self, item):
        self.my_list.append(item)

obj1 = MyClass()
obj2 = MyClass()

obj1.add_item(1)
print(obj2.my_list)  # [] - Separate!
2.5 Python Memory Management
Reference Counting
import sys

a = [1, 2, 3]
print(sys.getrefcount(a))  # 2 (a + temporary in getrefcount)

b = a  # Create another reference
print(sys.getrefcount(a))  # 3

del b  # Remove reference
print(sys.getrefcount(a))  # 2

# When refcount reaches 0, memory is freed
Garbage Collection
import gc

# Circular references
class Node:
    def __init__(self, value):
        self.value = value
        self.next = None

# Create circular reference
node1 = Node(1)
node2 = Node(2)
node1.next = node2
node2.next = node1  # Circular!

# Even if we delete both...
del node1
del node2
# ...they still reference each other!
# Garbage collector handles this

# Force garbage collection
gc.collect()
2.6 Generators & Iterators
Generators - Lazy Evaluation
# Normal function - returns all at once
def get_numbers(n):
    result = []
    for i in range(n):
        result.append(i * i)
    return result

# Uses O(n) memory
numbers = get_numbers(1000000)  # Creates huge list!

# Generator - yields one at a time
def get_numbers_gen(n):
    for i in range(n):
        yield i * i  # Yields, doesn't return

# Uses O(1) memory!
numbers_gen = get_numbers_gen(1000000)  # No list created
for num in numbers_gen:
    print(num)  # Computed on-demand

# Generator expressions
squares = (x * x for x in range(10))  # Generator
squares_list = [x * x for x in range(10)]  # List

# Key difference:
print(type(squares))  # <class 'generator'>
print(type(squares_list))  # <class 'list'>
Custom Iterators
class Countdown:
    def __init__(self, start):
        self.current = start
    
    def __iter__(self):
        return self
    
    def __next__(self):
        if self.current <= 0:
            raise StopIteration
        self.current -= 1
        return self.current + 1

# Usage
for i in Countdown(5):
    print(i)  # 5, 4, 3, 2, 1
2.7 Context Managers
The with Statement
# Without context manager
file = open('file.txt', 'r')
try:
    content = file.read()
finally:
    file.close()  # Must remember to close!

# With context manager
with open('file.txt', 'r') as file:
    content = file.read()
# File automatically closed!

# Custom context manager
class MyContext:
    def __enter__(self):
        print("Entering context")
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        print("Exiting context")
        # Return True to suppress exceptions
        return False

with MyContext() as ctx:
    print("Inside context")

# Using contextlib
from contextlib import contextmanager

@contextmanager
def my_context():
    print("Setting up")
    yield  # Everything before yield = __enter__
    print("Tearing down")  # Everything after = __exit__

with my_context():
    print("Inside context")
3. TRICKY INTERVIEW QUESTIONS
Question 1: What's the output?
def func(a, b=[]):
    b.append(a)
    return b

print(func(1))  # ?
print(func(2))  # ?
print(func(3, []))  # ?
print(func(4))  # ?

# Answer:
# [1]
# [1, 2] - Same list!
# [3]
# [1, 2, 4] - Same list again!
Question 2: What's the output?
x = [1, 2, 3]
y = x
z = x[:]

x.append(4)

print(y)  # ?
print(z)  # ?

# Answer:
# [1, 2, 3, 4] - y is reference
# [1, 2, 3] - z is copy
Question 3: What's the output?
def outer():
    x = 1
    def inner():
        x = 2
        print(f"inner: {x}")
    inner()
    print(f"outer: {x}")

outer()

# Answer:
# inner: 2
# outer: 1
# inner's x is local variable, doesn't affect outer's x
Question 4: What's the output?
def outer():
    x = 1
    def inner():
        x += 1  # Error!
        print(x)
    inner()

# Answer: UnboundLocalError
# Why? x += 1 is x = x + 1
# Python sees assignment, thinks x is local
# But x isn't defined yet!

# Solution:
def outer():
    x = 1
    def inner():
        nonlocal x  # Tell Python to use outer's x
        x += 1
        print(x)
    inner()
Question 5: What's the output?
a = [1, 2, 3]
b = [1, 2, 3]

print(a == b)  # ?
print(a is b)  # ?

# Answer:
# True - same values
# False - different objects

c = a
print(a is c)  # True - same object
Question 6: Closure
def multiplier(n):
    def multiply(x):
        return x * n
    return multiply

times_3 = multiplier(3)
print(times_3(10))  # 30

times_5 = multiplier(5)
print(times_5(10))  # 50

# Each closure maintains its own 'n'
Question 7: Late Binding in Loops
funcs = []
for i in range(5):
    funcs.append(lambda: i)

print([f() for f in funcs])  # ?

# Answer: [4, 4, 4, 4, 4]
# All lambdas reference same 'i', which ends at 4

# Fix:
funcs = []
for i in range(5):
    funcs.append(lambda x=i: x)  # Bind i's VALUE

print([f() for f in funcs])  # [0, 1, 2, 3, 4]
Question 8: Multiple Inheritance - MRO
class A:
    def method(self):
        print("A")

class B(A):
    def method(self):
        print("B")

class C(A):
    def method(self):
        print("C")

class D(B, C):
    pass

d = D()
d.method()  # ?

# Answer: "B"
# MRO (Method Resolution Order): D -> B -> C -> A
print(D.__mro__)
4. ADVANCED TOPICS
4.1 List Comprehensions vs Generator Expressions
# List comprehension - creates list immediately
squares_list = [x**2 for x in range(1000000)]  # Uses lots of memory

# Generator expression - lazy evaluation
squares_gen = (x**2 for x in range(1000000))  # Uses minimal memory

# Conditional
even_squares = [x**2 for x in range(10) if x % 2 == 0]

# Nested
matrix = [[1, 2], [3, 4], [5, 6]]
flat = [num for row in matrix for num in row]  # [1, 2, 3, 4, 5, 6]

# Dict comprehension
squares_dict = {x: x**2 for x in range(5)}

# Set comprehension
unique_lengths = {len(word) for word in ["hello", "world", "hi"]}
4.2 *args and **kwargs
# *args - variable positional arguments (tuple)
def func(*args):
    print(args)  # Tuple of arguments
    for arg in args:
        print(arg)

func(1, 2, 3)  # (1, 2, 3)

# **kwargs - variable keyword arguments (dict)
def func(**kwargs):
    print(kwargs)  # Dict of arguments
    for key, value in kwargs.items():
        print(f"{key} = {value}")

func(name="John", age=25)  # {'name': 'John', 'age': 25}

# Both together
def func(*args, **kwargs):
    print(f"args: {args}")
    print(f"kwargs: {kwargs}")

func(1, 2, name="John", age=25)
# args: (1, 2)
# kwargs: {'name': 'John', 'age': 25}

# Unpacking
def add(a, b, c):
    return a + b + c

numbers = [1, 2, 3]
print(add(*numbers))  # Unpacks list

data = {'a': 1, 'b': 2, 'c': 3}
print(add(**data))  # Unpacks dict
4.3 Lambda Functions
# Lambda - anonymous function
square = lambda x: x ** 2
print(square(5))  # 25

# With multiple arguments
add = lambda a, b: a + b

# Common uses
numbers = [1, 2, 3, 4, 5]

# map
squared = list(map(lambda x: x**2, numbers))

# filter
evens = list(filter(lambda x: x % 2 == 0, numbers))

# sorted with key
words = ["apple", "pie", "zoo", "a"]
sorted_by_length = sorted(words, key=lambda x: len(x))

# But prefer named functions for readability!
def square(x):
    return x ** 2

# Better than lambda for complex logic
4.4 Global, Nonlocal, Local
x = "global"

def outer():
    x = "outer"
    
    def inner():
        x = "inner"
        print(f"inner: {x}")
    
    inner()
    print(f"outer: {x}")

outer()
print(f"global: {x}")

# Output:
# inner: inner
# outer: outer
# global: global

# Using nonlocal
def outer():
    x = "outer"
    
    def inner():
        nonlocal x  # Modifies outer's x
        x = "inner"
        print(f"inner: {x}")
    
    inner()
    print(f"outer: {x}")  # Now prints "inner"!

# Using global
x = "global"

def modify_global():
    global x  # Modifies global x
    x = "modified"

modify_global()
print(x)  # "modified"
5. PYTHON BEST PRACTICES
5.1 Exception Handling
# Basic
try:
    result = 10 / 0
except ZeroDivisionError:
    print("Cannot divide by zero")

# Multiple exceptions
try:
    result = int("abc")
except (ValueError, TypeError) as e:
    print(f"Error: {e}")

# Finally (always runs)
try:
    file = open("file.txt")
    # process file
finally:
    file.close()  # Always closes

# Else (runs if no exception)
try:
    result = 10 / 2
except ZeroDivisionError:
    print("Error")
else:
    print(f"Result: {result}")  # Runs if successful
finally:
    print("Done")

# Custom exceptions
class ValidationError(Exception):
    pass

def validate_age(age):
    if age < 0:
        raise ValidationError("Age cannot be negative")
    return age
5.2 Type Hints
from typing import List, Dict, Optional, Union, Tuple

# Basic types
def greet(name: str) -> str:
    return f"Hello, {name}"

# Collections
def process_numbers(numbers: List[int]) -> int:
    return sum(numbers)

def get_user_data() -> Dict[str, str]:
    return {"name": "John", "email": "john@example.com"}

# Optional (can be None)
def find_user(user_id: int) -> Optional[str]:
    if user_id == 1:
        return "John"
    return None

# Union (multiple types)
def process_data(data: Union[str, int]) -> str:
    return str(data)

# Tuple
def get_coordinates() -> Tuple[float, float]:
    return (10.5, 20.3)

# Type checking with mypy (static analysis)
# Run: mypy your_file.py
6. COMMON PITFALLS & GOTCHAS
Pitfall 1: Modifying List While Iterating
# WRONG!
numbers = [1, 2, 3, 4, 5]
for num in numbers:
    if num % 2 == 0:
        numbers.remove(num)  # Modifies while iterating!

print(numbers)  # [1, 3, 5] - Unexpected! Some items skipped

# CORRECT: Iterate over copy
numbers = [1, 2, 3, 4, 5]
for num in numbers[:]:  # Create copy with [:]
    if num % 2 == 0:
        numbers.remove(num)

# BETTER: List comprehension
numbers = [1, 2, 3, 4, 5]
numbers = [num for num in numbers if num % 2 != 0]
Pitfall 2: Comparing with == vs is
# is checks identity (same object)
# == checks equality (same value)

a = [1, 2, 3]
b = [1, 2, 3]
c = a

print(a == b)  # True (same values)
print(a is b)  # False (different objects)
print(a is c)  # True (same object)

# Always use == for value comparison
# Use is only for None, True, False
if x is None:  # Correct
    pass

if x == None:  # Works but not preferred
    pass
Pitfall 3: String Concatenation in Loop
# SLOW! Creates new string each time
result = ""
for i in range(1000):
    result += str(i)  # O(nÂ²) complexity!

# FAST! Join at end
result = "".join(str(i) for i in range(1000))  # O(n)
This is Part 1 of Python Mastery. Shall I continue with more advanced topics and create the FastAPI guide next?
