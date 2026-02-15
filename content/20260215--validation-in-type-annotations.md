+++
title = "Runtime validation in type annotations"
slug = "validation-in-type-annotations"
date = 2026-02-15
tags = ["python", "typing"]
categories = ["experiment"]
+++

## Motivation

I was trying to understand a pattern I've seen in libraries like
[FastAPI](https://fastapi.tiangolo.com/tutorial/query-params-str-validations/#use-annotated-in-the-type-for-the-q-parameter),
[pydantic](https://docs.pydantic.dev/latest/concepts/fields/#the-annotated-pattern) or
[cyclopts](https://cyclopts.readthedocs.io/en/stable/groups.html#validators); where type
annotations are used for runtime validation via `typing.Annotated`.

You'll also see such examples straight from the
[Python documentation](https://docs.python.org/3/library/typing.html#typing.Annotated)
but without an explicit example in how to implement the validation.

Here's what I've come up with as a minimal example.

## Type annotation metadata

We can retrieve type annotations metadata with `typing.get_type_hints` when applied to a
type.

```python
import typing as t

class A:
    x: int
    y: t.Annotated[int, "metadata"]

    def __init__(self, x: int, y: int):
        self.x = x
        self.y = y

annotations = t.get_type_hints(A, include_extras=True)
```

We could also write the shorter,

```python
from dataclasses import dataclass
import typing as t

@dataclass
class A:
    x: int
    y: t.Annotated[int, "metadata"]

annotations = t.get_type_hints(A, include_extras=True)
```

Which gets you, in both cases, the annotations:

```python
{'x': <class 'int'>, 'y': typing.Annotated[int, 'metadata']}
```

The metadata is permissive and just like the example in the documentation (reproduced
here), you can pass objects as metadata.

```python
from dataclasses import dataclass
from typing import Annotated

@dataclass
class ValueRange:
    lo: int
    hi: int

T1 = Annotated[int, ValueRange(-10, 5)]
T2 = Annotated[T1, ValueRange(-20, 3)]
```

## Using the metadata

It's up to you, the developer, to handle the metadata. The way I see it, I'd like to go
over the **callable** metadata if it exists and call it on the attribute. I also want to
avoid errors on code that doesn't provide additional metadata in annotations.

I've made some executive choices here:

- Using a frozen `dataclass` because I think immutability as a default is better.
- Retrieve the `__metadata__` if it exists and only get the `callable`s amongst them.
- Apply the callable(s) in the metadata to the attribute.

Note the `object.__setattr__` which is
[the escape hatch](https://docs.python.org/3/library/dataclasses.html#frozen-instances)
to modify frozen instances of dataclasses during post-initialization.

```python
from dataclasses import dataclass
import typing as t

@dataclass(frozen=True)
class Base:
    def __post_init__(self):
        annotations = t.get_type_hints(type(self), include_extras=True)
        for k, annot in annotations.items():
            callables = [elem for elem in getattr(annot, "__metadata__", []) if callable(elem)]
            for f in callables:
                val = getattr(self, k)
                object.__setattr__(self, k, f(val))
```

Functions are callable objects and they are perfectly to use as metadata but we often
want to call them with additional parameters when the instance is initialized, not
directly in the annotation. The usual way to go around this issue is to create `callable`
classes. Let's say we want to bound numeric values to certain ranges, we can do this
similarly to what
[cyclopts does](https://github.com/BrianPugh/cyclopts/blob/2d2ffbe40c2b567d1fcbc3914b0c6940922a8c4a/cyclopts/validators/_number.py).

```python
from dataclasses import dataclass
import typing as t

def _lt(value: int | float, bound: int | float) -> int | float:
    if value >= bound:
        raise ValueError(f"{value} must be < to {bound}")
    return value

def _lte(value: int | float, bound: int | float) -> int | float:
    if value > bound:
        raise ValueError(f"{value} must be <= to {bound}")
    return value

def _gt(value: int | float, bound: int | float) -> int | float:
    if value <= bound:
        raise ValueError(f"{value} must be > to {bound}")
    return value

def _gte(value: int | float, bound: int | float) -> int | float:
    if value < bound:
        raise ValueError(f"{value} must be >= to {bound}")
    return value

@t.final  # Don't subclass this
@dataclass(frozen=True)
class Number:
    lt: int | float | None = None
    lte: int | float | None = None
    gt: int | float | None = None
    gte: int | float | None = None

    def __call__(self, value: int | float) -> int | float:
        if self.lt is not None:
            value = _lt(value, self.lt)
        if self.lte is not None:
            value = _lte(value, self.lte)
        if self.gt is not None:
            value = _gt(value, self.gt)
        if self.gte is not None:
            value = _gte(value, self.gte)
        return value

@dataclass(frozen=True)  # the children of a frozen parent must be frozen
class A(Base):
    x: t.Annotated[int, Number(gt=0, lt=100)]
    z: int

a1 = A(10, 100)  # works
a2 = A(-1, 100)  # fails validation! ValueError: -1 must be > to 0
```

## Better error handling

The current solution is fine but has an annoying caveat: validation stops at the first
exception. Most often than not, I want to see all the validation errors to fix them, not
re-run the code until I don't get exceptions anymore. Since version 3.11, Python has
[exception groups](https://docs.python.org/3/library/exceptions.html#exception-groups);
that part of the documentation is terse though and I found a better explanation in the
[exception tutorial](https://docs.python.org/3/tutorial/errors.html#raising-and-handling-multiple-unrelated-exceptions).

They are a great solution for grouping multiple unrelated (as in, not chained) exceptions
and give context as well. Let's see what the implementation looks like now:

```python
from dataclasses import dataclass
import typing as t

class ValidationErrorGroup(ExceptionGroup):
    def derive(self, excs):
        return ValidationErrorGroup(self.message, excs)


@dataclass(frozen=True)
class Base:
    def __post_init__(self):
        annotations = t.get_type_hints(type(self), include_extras=True)
        excs: list[Exception] = []
        for k, annot in annotations.items():
            callables = [
                elem for elem in getattr(annot, "__metadata__", []) if callable(elem)
            ]
            for f in callables:
                try:
                    val = getattr(self, k)
                    object.__setattr__(self, k, f(val))
                except ValueError as e:
                    e.add_note(f"Error for attribute '{k}': {val}")
                    excs.append(e)
        if excs:
            raise ValidationErrorGroup("Validation Errors", excs)
```

I've decided to only catch `ValueError` as I believe it's the one exception you should
use in validation functions/classes but you are free to add more.

You'll also see the `e.add_note` in there, to provide additional context when getting an
error.

Now, let's try the implementation with multiple validation errors:

```python
from dataclasses import dataclass
import typing as t

def nonempty_string(value: str) -> str:
    if len(value) == 0 or value.isspace():
        raise ValueError("String without any non-whitespace characters.")
    return value

@dataclass(frozen=True)
class A(Base):
    x: t.Annotated[int, Number(gt=0, lt=100)]
    y: t.Annotated[str, nonempty_string]
    z: int = 0

a = A(-1, "   ")
```

Will output:

```text
  + Exception Group Traceback (most recent call last):
  |   File "/home/nathan/.vscode/extensions/ms-python.python-2026.0.0-linux-x64/python_files/python_server.py", line 134, in exec_user_input
  |     retval = callable_(user_input, user_globals)
  |   File "<string>", line 1, in <module>
  |   File "<string>", line 6, in __init__
  |   File "<string>", line 26, in __post_init__
  | ValidationErrorGroup: Validation Errors (2 sub-exceptions)
  +-+---------------- 1 ----------------
    | Traceback (most recent call last):
    |   File "<string>", line 21, in __post_init__
    |   File "<string>", line 35, in __call__
    |   File "<string>", line 13, in _gt
    | ValueError: -1 must be > to 0
    | Error for attribute 'x': -1
    +---------------- 2 ----------------
    | Traceback (most recent call last):
    |   File "<string>", line 21, in __post_init__
    |   File "<string>", line 6, in nonempty_string
    | ValueError: String without any non-whitespace characters.
    | Error for attribute 'y':
    +------------------------------------
```

Thanks for reading! See below if you're interested in another way to approach the same
problem =)

## Bonus: using closures instead of classes

The more functionally inclined developers will know that late-binding of values can also
happen through closures rather objects (here, read classes).

Here's a solution using `functools.partial` for the bounding value and another closure
to capture the value being tested, like:

```txt
cmp_function(bound)(value)
```

```python
from collections.abc import Callable
from functools import partial, Placeholder  # placeholder requires 3.14+

def make_partial_cmp(
    func: Callable[[int | float, int | float], int | float],
) -> Callable[[int | float], partial[int | float]]:
    def f(value: int | float):
        return partial(func, Placeholder, value)

    return f

# Using the comparison functions from earlier
lt = make_partial_cmp(_lt)
lte = make_partial_cmp(_lte)
gt = make_partial_cmp(_gt)
gte = make_partial_cmp(_gte)

@dataclass(frozen=True)
class A(Base):
    x: t.Annotated[int, gte(0), lt(10)]  # multiple functions can be chained
```

For Python at least, callable classes are easier to read but I like this solution because
it avoid littering the code with `None` like in the `Number` example and it encourages
function re-use.
