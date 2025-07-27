+++
title = "Functional Core Imperative Shell - Filesystem operations"
slug = "fics-filesystem-operations"
date = 2025-07-26
tags = ["python", "functional"]
categories = ["guide"]
+++

## Context

I was writing a tool (basically a setup script) at `$work`, for people to setup
or update their profiles on our servers: configuration files for bash, podman,
git, etc. As well as put all their files in the correct folders and symlink some
files to the correct place. I also needed to install and update some executables
which everyone is expected to have available.

Another important factor was that the script needed to provide dry run
capabilities. Indeed, as people might sometimes not update for a long time or
have a wildly different setup than expected, they should be able to know the
impact of running the script on their system beforehand.

Yet another caveat, the tool should run (dry or regular) and be flexible: you
can't just drop everything because a file is missing or a folder doesn't exist
yet; the setup should be as close to completed as possible, barring catastrophic
mistakes.

I had heard about ["Functional Core Imperative
Shell"](https://www.destroyallsoftware.com/screencasts/catalog/functional-core-imperative-shell)
(FICS) in some podcasts and articles beforehand and I thought it did fit what I
was trying to.

From my understanding, FICS defines a functional core which relies on immutable
datastructures and idempotent functions; the core creates and pushes immutable
data to the imperative shell, which handles mutations. In my case, this means
that the core is in charge of bundling data about which paths get symlinked,
which files are created, which executables need to be installed… But, it never
directly changes the filesystem, this is the prerogative of the imperative
shell! This also has interesting properties when testing, unless there's
significant business logic going on, I don't need to test the functional core
too much and I can focus on the imperative shell. Since Python is my language of
choice, I'll show how to use [`pytest` temporary
files](https://docs.pytest.org/en/stable/how-to/tmp_path.html) to do that.

## Into the weeds

Here I want to document what my thought process is, because the first tool I
want to reach out for, out of habit, is Object Oriented Programming (OOP).

First rough draft  would look like:

```python
from pathlib import Path

class FileManager:
    def __init__(self):
        pass

    def make_symlink(self, src: Path, dst: Path) -> None:
        ...

    def mv(self, src: Path, dst: Path) -> None:
        ...
```

Not great for many reasons: I'm not hiding any state in the object that other
objects shouldn't know about. I'm also not about to reproduce a whole parallel
(buggy) implementation of a filesystem in that object. Moreover, the methods
could be class methods, because I'm not going to need any attributes of the
class: the state is the filesystem itself!

Thinking back on it, and because Python allows for objects to be callable,

```python
from dataclasses import dataclass
from pathlib import Path

@dataclass
class MakeSymlinkOp:
    src: Path
    dst: Path

    def __call__(self):
        ...

@dataclass
class MoveFileOp:
    src: Path
    dst: Path

    def __call__(self):
        ...
```

Wait! If I'm building an object that only has a couple of methods (especially
`__call__`), then I most likely just want a function that takes data as input,
not an object with custom behavior.

I like the idea of having a different type per operation, and I will be able to
use Python's [`match`
statements](https://docs.python.org/3/tutorial/controlflow.html#match-statements)
and implement different behaviors based on the type. We're straying further away
from OOP, as the equivalent implementation would rely on
[polymorphism](https://wiki.c2.com/?PolymorphismAndInheritance), i.e each
operation object would implement the same method(s) differently, under a common
interface. In Python, that kind of contract is usually enforced by using
[abstract base classes](https://docs.python.org/3/library/abc.html). We won't be
doing any of that here though.

## Defining operations

Now comes a difficult question: how much meaning do I want to encode into the
types? One option would be to just define:

```python
from dataclasses import dataclass
from pathlib import Path

@dataclass(frozen=True, kw_only=True)
class PathData:
    src: Path
    dst: Path
```

where the data is "dumb" and the type is minimally informative about the
destination of the data.

Another option is:

```python
from dataclasses import dataclass
from pathlib import Path

type PathData = SymlinkData | MoveData

@dataclass(frozen=True, kw_only=True)
class SymlinkData:
    src: Path
    dst: Path

@dataclass(frozen=True, kw_only=True)
class MoveData:
    src: Path
    dst: Path
```

With two different types which both carry the same attributes and types but the
semantics are clearly different.

Over time, I'm leaning more towards the second option, because it captures my
intent better and it's more informative to others as well.

Something that is also important to my implementation, since this will be used
for user-facing operations, is that every operation needs a justification as to
why it will be run or not.

### On pure functions

## Applying operations
