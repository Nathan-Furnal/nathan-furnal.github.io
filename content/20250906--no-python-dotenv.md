+++
title = "You might not need python-dotenv"
slug = "no-python-dotenv"
date = 2025-09-06
tags = ["python", "practices"]
categories = ["guide"]
+++

I often see [`python-dotenv`](https://github.com/theskumar/python-dotenv) as a
dependency in applications where one needs to load the `.env` and think to
myself: 'Why?'. It's a convenient package but the usage most applications have
of it can easily be implemented which is what I'll show here. Note that I'm
talking about applications, not libraries you'd want to package for others.

Another thing that bugs me, is that it keeps easing a pattern I don't think
people should use: loading configurations from the environment. Your settings
and/or configurations should be immutable Python objects, not read from a global
environment that can change under your feet, but [I've ranted about that
before](https://blog.natfu.be/managing-settings-in-python-projects/). Also,
`python-dotenv` provides ways to load the values and not mutate the environment;
the reproach is about the most common usage.

## Building our own `dotenv`

A simple application structure can look like this:

```text
.
├── app
│   ├── conf.py
│   └── main.py
├── .env
├── pyproject.toml
├── tests
└── uv.lock
```

Where `conf.py` is the file of interest. Within, we can start with a simple
implementation for reading the `.env` file(s).

Since we know and control the structure of the project, it's possible to find
the root of the project which is where you'll find `.env` files. We'll get
started and use
[`__file__`](https://docs.python.org/3/reference/datamodel.html#module.__file__)
which, following the docs:

> indicates the pathname of the file from which the module was loaded (if loaded from a file)

We'll also read from the environment file and split the values on the equal
character.

```python
from pathlib import Path

ROOT_DIR = Path(__file__).parents[1]

def get_dotenv(fp: Path) -> dict[str, str]:
    content = fp.read_text()
    splitvals = [x.split("=") for x in content.splitlines()]
    return {k.strip(): v.strip() for k, v in splitvals}
```

Now, the `.env` file has the following content,

```text
APP_ENV=dev
DB_NAME=my_db
DB_USER=root
DB_PASSWORD=passw
DB_PORT=1111
OTHER_VAL=hello
IRRELEVANT='abcdefghijklm'
```

and the `main.py` has

```python
import conf

print(conf.get_dotenv(conf.ROOT_DIR / ".env"))
```

Which, when run with `python main.py` in an activated virtual environment or via
`uv run main.py` will print the content of the `.env` as a dictionary.

For some, this might just be enough! Others though, might benefit from expanding
variables as shown in the `python-dotenv` documentation, where

```text
DOMAIN=example.org
ADMIN_EMAIL=admin@${DOMAIN}
ROOT_URL=${DOMAIN}/app
```

is expanded to the following object:

```python
{'DOMAIN': 'example.org', 'ADMIN_EMAIL': 'admin@example.org', 'ROOT_URL': 'example.org/app'}
```

The library takes care to properly parse variables, such that this file content

```text
DOMAIN=example.org
ADMIN_EMAIL=admin@${DOMAIN}
COMPOUND=${ADMIN_EMAIL}/var/stuff/${DOMAIN}
```

correctly expands `COMPOUND` which requires `ADMIN_EMAIL` to expand `DOMAIN`.

The above implementation is clearly correct and the kind of behavior I would
expect from such a library but what if I could get away with only expanding
variables without allowing nesting? After all, this kind of nesting rarely comes
up.

### Expanding variables

In order to expand variables properly, I would need to write a parser. But to
hack my way around one level of variable definition, I can use [regular
expressions](https://docs.python.org/3/howto/regex.html).

At a high level, the solution is to first get the `.env` content into a
dictionary as a first pass, then find all expressions like `${<something>}` and
substitute that back into the raw content.

Python even lets us pass a function when substituting strings with regular
expressions, which leads to a short implementation:

```python
from pathlib import Path
import re

def get_dotenv(fp: Path) -> dict[str, str]:
    content = fp.read_text()
    pat = re.compile(r"\$\{\b.+\b\}")
    splitvals = [x.split("=") for x in content.splitlines()]
    raw = {k.strip(): v.strip() for k, v in splitvals}
    def varexpand(matchobj: re.Match[str]) -> str:
        strmatch = matchobj.group(0).lstrip("${").rstrip("}")
        return raw.get(strmatch, '')
    expanded = re.sub(pat, varexpand, content)
    splitvals = [x.split("=") for x in expanded.splitlines()]
    return {k: v for k, v in splitvals}
```

We can see this in action with the `.env` file with one level of variables mentioned
above and running `main.py` outputs

```python
{'DOMAIN': 'example.org', 'ADMIN_EMAIL': 'admin@example.org', 'ROOT_URL': 'example.org/app'}
```

Voilà, a nice 80/20 solution to read `.env` files for your applications. You can
stop here if that's all you needed but now I'll show you how to put your
configuration into an immutable Python object.

## Storing the `.env` content in an immutable object

Instead of returning a dictionary, I'd like to build a `dataclass` that can be
build from a `.env` file. By marking it as `frozen`, it will be "immutable".
Which means that it will raise an error when trying to set attributes though
there are some ways around that. Still in `conf.py`, it is defined like so:

```python
from dataclasses import dataclass

@dataclass(frozen=True, kw_only=True)
class EnvConfig:
    app_env: str
    db_name: str
    db_user: str
    db_password: str
    db_port: int
```

Then, I would like to build it from an environment file, while ignoring extra
keys as well as ensuring the types of the attributes are correct. When you
define a `dataclass`, this attributes and types are neatly accessible in the
`__annotations__` attribute.

```python
>>> EnvConfig.__annotations__
{'app_env': <class 'str'>, 'db_name': <class 'str'>, 'db_user': <class 'str'>, 'db_password': <class 'str'>, 'db_port': <class 'int'>}
```

Note that the values of the dictionaries are **callable**, we'll be able to call
those classes on the values of the dictionary obtained from the environment.
Let's also create a `classmethod` that will take a path to the relevant file.
Now, `conf.py` looks like

```python
import os
import typing as t
from dataclasses import dataclass
from pathlib import Path
import re

ROOT_DIR = Path(__file__).parents[1]


def get_dotenv(fp: Path) -> dict[str, str]:
    content = fp.read_text()
    pat = re.compile(r"\$\{\b.+\b\}")
    splitvals = [x.split("=") for x in content.splitlines()]
    raw = {k.strip(): v.strip() for k, v in splitvals}
    def varexpand(matchobj: re.Match[str]) -> str:
        strmatch = matchobj.group(0).lstrip("${").rstrip("}")
        return raw.get(strmatch, '')
    expanded = re.sub(pat, varexpand, content)
    splitvals = [x.split("=") for x in expanded.splitlines()]
    return {k: v for k, v in splitvals}


@dataclass(frozen=True, kw_only=True)
class EnvConfig:
    app_env: str
    db_name: str
    db_user: str
    db_password: str
    db_port: int

    @classmethod
    def from_dotenv(cls: type[t.Self], fp: Path = ROOT_DIR / ".env") -> t.Self:
        data = {}
        annotations = cls.__annotations__
        dotenv = get_dotenv(fp)
        for k in annotations.keys() & map(str.lower, dotenv.keys()):
            data[k] = annotations[k](dotenv[k.upper()])
        return cls(**data)
```

I'm making some assumptions of course, such that the name of the attributes
appear in uppercase in the `.env` file. Since dictionary keys behave like `set`
objects, I can use the `&` operator to only retrieve keys that appear in both
the annotations and the `.env` file content. Meaning that creating the object
will fail on missing keys, which is what I usually want. I can also use the
callable values of the annotations to convert the environment file content to
the right type and then return the newly created object.

In `main.py`, with the first `.env` I presented, let's do

```python
import conf

print(conf.EnvConfig.from_dotenv(conf.ROOT_DIR / ".env"))
```

which outputs,

```python
EnvConfig(app_env='dev', db_name='my_db', db_user='root', db_password='passw', db_port=1111)
```

Again, there are some trade-offs here but I think it provides a lot of value for
a small amount of code.

## Using `load_dotenv` anyways

I can appreciate that not everyone thinks like me and that you'd want to have
something similar to the `load_dotenv` function of the `python-dotenv` library,
to load the content of environment files into the global environment.

Here it goes,

```python
import os
import typing as t
from dataclasses import asdict, dataclass
from pathlib import Path
import re

ROOT_DIR = Path(__file__).parents[1]


def get_dotenv(fp: Path) -> dict[str, str]:
    content = fp.read_text()
    pat = re.compile(r"\$\{\b.+\b\}")
    splitvals = [x.split("=") for x in content.splitlines()]
    raw = {k.strip(): v.strip() for k, v in splitvals}
    def varexpand(matchobj: re.Match[str]) -> str:
        strmatch = matchobj.group(0).lstrip("${").rstrip("}")
        return raw.get(strmatch, '')
    expanded = re.sub(pat, varexpand, content)
    splitvals = [x.split("=") for x in expanded.splitlines()]
    return {k: v for k, v in splitvals}


@dataclass(frozen=True, kw_only=True)
class EnvConfig:
    app_env: str
    db_name: str
    db_user: str
    db_password: str
    db_port: int

    @classmethod
    def from_dotenv(cls: type[t.Self], fp: Path = ROOT_DIR / ".env") -> t.Self:
        data = {}
        annotations = cls.__annotations__
        dotenv = get_dotenv(fp)
        for k in annotations.keys() & map(str.lower, dotenv.keys()):
            data[k] = annotations[k](dotenv[k.upper()])
        return cls(**data)

    def as_str_dict(self) -> dict[str, str]:
        return {k: str(v) for k, v in asdict(self).items()}


def load_dotenv(fp: Path) -> None:
    dotenv = get_dotenv(fp)
    os.environ.update(dotenv)


def load_dotenv_from_conf(conf: EnvConfig) -> None:
    os.environ.update(conf.as_str_dict())
```

I've added a `load_dotenv` function which is self-explanatory. I've also added a
way to load the `EnvConfig` into the environment. We can use another nice bit of
standard library with `dataclasses.asdict` and then ensure that all values are
strings, a constraint of `os.environ`.

## Conclusion

I hope I've shown you a useful implementation! There are some nice quality of life
features I could imagine like adding support for comments or finding `.env` files
automatically but then I would most likely end up with something similar to
`python-dotenv` =)
