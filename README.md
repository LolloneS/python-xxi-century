# Python for the XXI century
During the last few years, Python has seen his user base [grow significantly](https://www.zibtek.com/blog/the-incredible-growth-of-python/). [Developers like Python](https://insights.stackoverflow.com/survey/2020#technology-most-loved-dreaded-and-wanted-languages-loved) and even non-techies enjoy its ease of use and versatility. As a matter of fact, Python works great for a variety of tasks, including [automating stuff](https://automatetheboringstuff.com/), [deep learning](https://pytorch.org/), and generic "bash-with-decent-syntax" scripting. Unsurprisingly, though, Python has some flaws that people criticize daily. In particular, the [duck typing](https://en.wikipedia.org/wiki/Duck_typing) philosophy may come at a high cost in technical debt, especially in fast-growing codebases where code review is not performed systematically. Furthermore, being it an interpreted language, some errors may only arise at the worst possible moment at runtime rather than during a compilation step. In this blog post, we will learn how to use some tools and functionalities in order to enforce higher quality standards in Python codebases.

## Typing and MyPy
Introduced in Python 3.5, the [typing](https://docs.python.org/3/library/typing.html) library introduces support for type hints. Although these checks are not enforced in any way at runtime, consistency in type usage can be verified statically using [MyPy](http://mypy-lang.org/). For instance, the execution of MyPy can be set in a CI script so that a CI/CD pipeline will not pass unless types are correctly defined. 

For instance, let's consider the following code:
```python
import random
from typing import Dict, List

def print_animals(animals: Dict[str, List[str]]):
  for k, v in animals.items():
    print(f"Owner: {k}, Animals: {v}")

x = random.randint(0, 100)
if x != 42:
  print_animals({"Foo": ["tortoise", "sea lion"]})
else:
  print_animals(["Baz", "bar"])
```

A random execution of the script above would be fine 99% of the time, yet it would fail in case `x == 42`. Hard to spot for the oblivious developer, the bug will relentlessly be found by MyPy:
```bash
➜ mypy test.py
test.py:12: error: Argument 1 to "print_animals" has incompatible type "List[str]"; expected "Dict[str, List[str]]"
Found 1 error in 1 file (checked 1 source file)
```


## Dataclasses
Introduced in Python 3.7, [dataclasses](https://docs.python.org/3/library/dataclasses.html) provide the user with the capability to automatically generate special methods in classes. [PEP 557](https://www.python.org/dev/peps/pep-0557/) intuitively describes dataclasses as *mutable namedtuples with defaults*.

Think about the highly-renowned class `Point`, describing a point in the X-Y Cartesian plane. In "classic" Python, we would have to write *a lot* of boilerplate code: an `__init__` constructor, a `__repr__`, possibly an `__eq__`, etcetera. Dataclasses free the developer from the burden of defining such methods.

As a dataclass, `Point` would look as follows:
```python
from dataclasses import dataclass

@dataclass
class Point:
  x: float
  y: float

p = Point(1, 2)
p.x = 2
print(p) # Point(x=2, y=2)
```
In case `Point` objects were immutable, we could also define the dataclass as `frozen` to also get a free `__hash__`.

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Point:
  x: float
  y: float

p = Point(1, 2)
d = {}
d[p] = "Free hash!"
for k, v in d.items():
  print(f"{k}: {v}")  # Point(x=1, y=2): Free hash!
```

Dataclasses can, of course, be used in conjunction with `typing` and MyPy in order to enrich as much code as possible with meaningful data types:
```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Point:
  x: float
  y: float

def print_point(p: Point) -> None:
  print(f"x: {p.x},  y: {p.y}")

p = Point(1, 2)
print_point(p) # x: 1,  y: 2
```

```bash
➜ mypy asdasd.py
Success: no issues found in 1 source file
```

We strongly suggest using dataclasses as they are easy and quick to define and can lead to a much more comprehensible and maintainable codebase. Even for small scripts, representing objects or namedtuples using dataclasses allows for more standardization and better static checks before execution. Generally speaking, every non-trivial set of fields should be represented with a meaningful abstraction, and dataclasses are great for achieving the goal.


## Flake8
[Flake8](https://flake8.pycqa.org/en/latest/manpage.html) is a toolkit that wraps many other tools under the hood. It is great for checking whether the code is PEP8-compliant, and perform a basic static analysis of the codebase against common mistakes or bad practices. Again, a `flake8` step in a CI script is a great idea to enforce code style and a basic static analysis on the codebase.

```python
from math import *

print(sqrt(4))
```

```bash
➜ flake8 test.py
test.py:1:1: F403 'from math import *' used; unable to detect undefined names
test.py:3:7: F405 'sqrt' may be undefined, or defined from star imports: math
```


```python
def strange():
    x = 2
    for _ in range(4):
        x *= x
    return y

strange()
```

```bash
➜ flake8 test.py
test5.py:5:12: F821 undefined name 'y'
test5.py:7:1: E305 expected 2 blank lines after class or function definition, found 1
```

## Bandit
According to its creators, "[Bandit](https://bandit.readthedocs.io/en/latest/) is an extremely useful tool designed to find common security issues in Python code" via an analysis performed on an AST it creates starting from the codebase. Bandit looks for many different issues by means of [numerous plugins](https://bandit.readthedocs.io/en/latest/plugins/#complete-test-plugin-listing). It looks for the usage of "dangerous" functions such as `assert` or `eval`, hardcoded passwords, unsafe loads from files (e.g. YAML) and much much more.

```python
import yaml

y = yaml.load("evil.yaml")
x = input("Tell me which command to run")
exec(x)
```

```bash
➜ bandit test.py
Test results:
>> Issue: [B506:yaml_load] Use of unsafe yaml load. Allows instantiation of arbitrary objects. Consider yaml.safe_load().
   Severity: Medium   Confidence: High
   ...
>> Issue: [B102:exec_used] Use of exec detected.
   Severity: Medium   Confidence: High
   ...
```

Although Bandit's tests are far from perfect, they provide a great starting point for checking for common problems which are easy to fix but may still lead to serious consequences if ignored. For this reason, we strongly suggest implementing it as a step in your CI pipeline.
