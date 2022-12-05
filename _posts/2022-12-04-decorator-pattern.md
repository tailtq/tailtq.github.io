---
title: Decorator - A pattern that is ubiquitously used in Python
author: Tai Le
date: 2022-12-05
tags: [Python, Back-end]
---

Besides using the Bridge pattern, developers should also use Decorator to reduce the complexity of our code. **Adapter**, **Bridge**, and **Decorator** are from the same family but are used for different purposes. Adapter helps us convert an object's interface to an appropriate form before using it in the intended methods. Bridge helps separate a big class into smaller ones. Decorator helps add new logic without modifying the current class.

In today's post, we will dive into Decorator to see how it really works.

![Decorator Pattern](/assets/img/2022-12-05/decorator.png)
_Retrieved from RefactoringGuru_


## 1. Definition

According to RefactoringGuru, "Decorator is a structural design pattern that lets you attach new behaviors to objects by placing these objects inside special wrapper objects that contain the behaviors".

This pattern qualifies the Single Responsibility and Open-Closed principles in SOLID, the object is only responsible for what it currently does and doesn't allow modifying the code. Instead, Decorator pattern provides a proxy to execute additional new logic before running the main one.


## 2. Types of decorators & examples

There are two types of decorators in Python, one is the built-in decorator, and another is writing code like other languages. The native built-in of Python has been widely used in many projects to build packages, libraries, and frameworks such as FastAPI and Django. The second one is heavily OOP-based which is often used in Java, C#, and PHP. We will go through each of them in the following section.


### a. Python native built-in decorators

Built-in decorators in Python are actually high-order functions that wrap around one function to execute a behavior before or after the execution of that function. You might already hear the term **high-order functions** before, it's a very popular concept in JavaScript and [functional programming](https://en.wikipedia.org/wiki/Functional_programming). Basically, the decorator is defined by a function that is wrapped by another one, the outer return the inner as the result. The `@` must go along as a prefix when using it.

Please take a look at the code below for further understanding:

```python
import random

def save_log(func):
    def decorator(*args, **kwargs):
        result = func(*args, **kwargs)
        # execute saving log code
        print("Saving result to another service...")

        return result

    return decorator

@save_log
def execute_sql_query():
    print("SQL query is executing...")
    result = random.randint(1, 100)

    return result

execute_sql_query()
# Result:
# SQL query is executing...
# Saving result to another service...
```

In the example, I added a new behavior which is `save_log` without adding additional logic to the `execute_sql_query` function, that's the purpose of decorators. In my perspective, the native built-in is an unextendable approach because it violates some rules in SOLID. However, because of its simplicity and efficiency, many people (including me), frameworks, and libraries are still using it. As people always say, choosing one over another is just a matter of trade-off.


### b. Class-based decorators

Class-based decorators are the more popular type among languages that support OOP. You might have already encountered them in Java, C#, or PHP. Even though their code structure is hard to understand in the beginning, we can combine other OOP practices to make the code clearer and more extendable.

![Decorator Pattern](/assets/img/2022-12-05/decorator-class-based.png)
_Retrieved from SourceMaking_

Above is the diagram that can help us best understand how we should use the class-based decorator pattern based on the example above. This can be split into 3 parts:
- `Interface`: define the behavior that every decorator and their wrappee must follow.
- `SQLQuery` (wrappee): main executor.
- `Decorators` (decorators): define the new behavior that execute before or after the wrappee.

As we can see above, `SQLQuery` and `Decorators` implement the `Interface` and each decorator's `execute` method will call `wrappee.execute`; therefore, we can easily add new logic before or after calling it. In our previous example, we save the query's result to a log file. Let's see how we can implement the SQL querying example above using classes.

```python
import random

# Python doesn't support interface but multiple inheritances, so I'll use class here
# everything should share the same method so objects can be called in a nested way
class Interface:
    def execute(self):
        raise NotImplementedError()

class SQLQuery(Interface):
    def execute(self):
        print("SQL query is executing...")
        result = random.randint(1, 100)

        return result

class BaseSavingLogDecorator(Interface):
    def __init__(self, wrappee: Interface) -> None:
        super().__init__()
        self.wrappee = wrappee

class SaveLogDecorator(BaseSavingLogDecorator):
    def execute(self):
        result = self.wrappee.execute()
        # execute saving log code
        print("Saving result to another service...")

        return result

SaveLogDecorator(SQLQuery()).execute()

# we can also stack each decorator on top of the other
class SaveLogToDatabaseDecorator(BaseSavingLogDecorator):
    def execute(self):
        result = self.wrappee.execute()
        # execute saving log code
        print("Saving result to database...")

        return result

class SaveLogToTextFileDecorator(BaseSavingLogDecorator):
    def execute(self):
        result = self.wrappee.execute()
        # execute saving log code
        print("Saving result to a text file...")

        return result

# nested call
SaveLogToTextFileDecorator(SaveLogToDatabaseDecorator(SQLQuery())).execute()
# Result:
# SQL query is executing...
# Saving result to database...
# Saving result to a text file...
```

As you can see above, the order of decorators' operation is depended on how you stack the objects and whether you decide to call `wrappee.execute` before or after executing decorators' logic, this applies for both class-based decorators and built-in decorators.


## 3. Dive deeper into built-in decorators

Before jumping to real-world examples, we should understand a bit more about built-in decorators to make our work more smoothly when we need to use them.


### a. Add arguments to decorators

Going back to the log-saving example, if we have too many places to store our log, defining one decorator for each place is tedious and hard to manage. It would be great if we can pass a `place_type` argument to the general `save_log` decorator, and we can use the factory pattern in that decorator. Let's see how it works.

```python
import random
from typing import Any

def save_log_to_database(result: Any):
    print("Saving result to database...")

def save_log_to_text_file(result: Any):
    print("Saving result to a text file...")

def save_log(place_type: str):
    save_log_func = None

    if place_type == "database":
        save_log_func = save_log_to_database
    elif place_type == "text_file":
        save_log_func = save_log_to_text_file
    else:
        raise NameError("Invalid place_type")

    def save_log_inner(func):
        def decorator(*args, **kwargs):
            result = func(*args, **kwargs)
            # execute saving log code
            save_log_func(result)

            return result

        return decorator

    return save_log_inner

@save_log("text_file")
@save_log("database")
def execute_sql_query():
    print("SQL query is executing...")
    result = random.randint(1, 100)

    return result

execute_sql_query()
# Result:
# SQL query is executing...
# Saving result to database...
# Saving result to a text file...
```

Basically, the idea is to **add another layer of function**, which is quite complicated and hard for naming (that's why I name the inner one pretty badly), the outer one accepts parameters, processes them, and returns the true decorator.


### b. Introspection

Introspection in Python is the ability of an object to be aware of its own attributes and methods. It allows objects to query and modify their own structure and contents. It is an important feature of the language as it allows developers to access information about objects at runtime. This is done by using the built-in functions `dir()`, `getattr()`, `hasattr()`, `isinstance()`, `type()`, and attributes like `__name__`. *(generated by OpenAI GPT-3 :D)*

When using built-in decorators, we will lose the introspection ability and functions' information because Python would describe the decorator, not the main function itself.

```python
# @save_log("text_file")
# @save_log("database")
def execute_sql_query():
    print("SQL query is executing...")
    result = random.randint(1, 100)

    return result

print(execute_sql_query, execute_sql_query.__name__)
# Result:
# <function execute_sql_query at 0x10f0e9488> execute_sql_query

@save_log("text_file")
@save_log("database")
def execute_sql_query():
    print("SQL query is executing...")
    result = random.randint(1, 100)

    return result

print(execute_sql_query, execute_sql_query.__name__)
# Result:
# <function save_log.<locals>.save_log_inner.<locals>.decorator at 0x1088e3598> decorator
```

This one will be more dramatic if we try to use quick fixes in our code, aka shortcuts. For example, assuming that we have a cache function that saves every result of multiple functions into memory with the key based on parameters. When data is updated, we have to clear the cache to fetch the new result. So as a quick fix, I intentionally assign `clear_cache` to remove the cache of each function.

```python
import random

def clear_cache():
    print("Clearing cache...")

def cache(func):
    print("Setting clear_cache attribute...")
    setattr(func, "clear_cache", clear_cache)

    def decorator(*args, **kwargs):
        result = func(*args, **kwargs)
        # execute caching
        print("Caching result into memory...")

        return result

    return decorator

@cache
def execute_sql_query():
    print("SQL query is executing...")
    result = random.randint(1, 100)

    return result

execute_sql_query()
execute_sql_query.clear_cache()
# Result:
# Setting clear_cache attribute...
# SQL query is executing...
# Caching result into memory...
# Traceback (most recent call last):
#   File "test.py", line 27, in <module>
#     execute_sql_query.clear_cache()
# AttributeError: 'function' object has no attribute 'clear_cache'
```

So how can we solve this problem? Actually, Python provides the `functools.wraps` decorator to update these attributes. Below is the solution, please remember that we must call `@functools.wraps(func)` **after** setting the attributes, **not before**.

```python
import functools
import random

def clear_cache():
    print("Clearing cache...")

def cache(func):
    print("Setting clear_cache attribute...")
    func.clear_cache = clear_cache

    @functools.wraps(func)
    def decorator(*args, **kwargs):
        result = func(*args, **kwargs)
        # execute caching
        print("Caching result into memory...")

        return result
    # setattr(func, "clear_cache", clear_cache)

    return decorator

@cache
def execute_sql_query():
    print("SQL query is executing...")
    result = random.randint(1, 100)

    return result

execute_sql_query()
execute_sql_query.clear_cache()

# Result:
# Setting clear_cache attribute...
# SQL query is executing...
# Caching result into memory...
# Clearing cache...
```


## 4. Real-world examples

- **Web Application Frameworks**: Popular web application frameworks such as Flask and Django use decorators to simplify URL routing.
- **Logging**: Python's logging module provides an easy way to add logging to functions using decorators.
- **Memoization**: Memoization is a technique used to speed up code by caching the results of expensive function calls. This can be done with decorators.
- **Class Decorators**: In Python, classes can also be decorated with decorators. This is often used for adding custom functionality such as class methods and properties.


## 5. References

- RefactoringGuru. (n. d.). _Decorator_. Retrieved from [https://refactoring.guru/design-patterns/decorator](https://refactoring.guru/design-patterns/decorator).
- SourceMaking. (n. d.). _Decorator_. Retrieved from [https://sourcemaking.com/design_patterns/decorator](https://sourcemaking.com/design_patterns/decorator).
- TutorialsPoint. (n. d.). _Design Patterns - Decorator Pattern_. Retrieved from [https://www.tutorialspoint.com/design_pattern/decorator_pattern.htm](https://www.tutorialspoint.com/design_pattern/decorator_pattern.htm).
- Hjelle. G. A. (Jan 11, 2015). Primer on Python Decorators. RealPython. Retrieved from [https://realpython.com/primer-on-python-decorators/](https://realpython.com/primer-on-python-decorators/).
