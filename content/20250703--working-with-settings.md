+++
title = "Managing settings in Python projects"
slug = "managing-settings-in-python-projects"
date = 2025-07-05
tags = ["python", "softwareDesign"]
categories = ["guide"]
+++

## Motivation

In general, I've not seen any strong prescription in how to deal with settings when
working on Python applications. As the project grows and when you're exposing multiple
kinds of configurations, it makes sense to have a common interface to handle settings.

Otherwise, it can lead to some frustration, as any time you're working on an application
with multiple people, you'll find a mix of `.env` or JSON files, hard-coded global
variables and even some amount of setting and getting global environment variables.

What I've found useful instead of using any mix of the above solutions, is to define
**local**, **immutable** and **dumb** python objects defined in one place, which I can
then pass around the application. Ideally, I do not want to add a new dependency solely
for configuration purposes so the solution I'm looking for should rely on the standard
library. Note that if you already depend on
[pydantic](https://docs.pydantic.dev/latest/), they have a library called
[`pydantic-settings`](https://docs.pydantic.dev/latest/concepts/pydantic_settings/)
which fits the bill but I wouldn't introduce it otherwise.

Depending on your background and experience, you might never have encountered
those issues but if you work in data science or on machine learning adjacent
projects, they'll probably ring a bell, in part because of choices made in the
most common frameworks and libraries in the field.

N.B: You can [jump to the code](#code) if you prefer.

## What makes useful settings

Developing the previous section a bit more in depth, I believe great settings are:

- **local** in the sense that they shouldn't modify or be affected by some global state
  and should be safe to pass around in different parts of the code.
- **immutable** such that there is no question about the configuration, it is setup
  once and there is no potential change of state as you would have with a `dict`.
- **dumb**, meaning that there is no specific behavior of business logic implemented
  by the settings object, it should be plain data.

You might have an inkling to where I'm going with this: using
[`dataclasses`](https://docs.python.org/3/library/dataclasses.html), which were added
back in Python 3.7 and truly blossomed in Python 3.10 when many new features appeared.

I'll also add that you should put your configuration related `dataclasses` in one module,
which I usually call `settings.py` or `config.py`. Moreover, you should have one object
per kind of setting (web server, DB, resource access,…) though having a larger object
that bundles all configurations is fine.

Let's go with an example now, where I'll setup the [connection to a DB with
SQLAlchemy](https://docs.sqlalchemy.org/en/20/core/engines.html#engine-configuration),
with a bit more legwork in order to build a `Secret` type, which isn't safe by
any means but won't default to printing the value of the secret.

In a `config.py`, there's now:

```python
from dataclasses import dataclass
from typing import final, override

@final
@dataclass(frozen=True, repr=False)
class Secret:
    _value: str

    @property
    def value(self) -> str:
        raise NotImplementedError

    def get_secret_value(self) -> str:
        return self._value

    @override
    def __str__(self) -> str:
        return "Secret(*****)"

    @override
    def __repr__(self) -> str:
        return "Secret(*****)"


@dataclass(frozen=True, kw_only=True)
class DBSettings:
    drivername: str
    username: str
    password: Secret
    host: str
    port: int
    database: str

    def to_dict(self) -> dict[str, str | int]:
        # dataclasses.asdict is also valid
        # as long as the password is handled properly
        return {
            "drivername": self.drivername,
            "username": self.username,
            "password": self.password.get_secret_value(),
            "host": self.host,
            "port": self.port,
            "database": self.database,
        }
```

Which makes it possible to pass the dictionary generated from `DBSettings`, to
the `URL.create` function of SQLAlchemy.

By using `frozen=True`, any attempt at modifying the members of `DBSettings`
will raise a `FrozenInstanceError`.

You can improve the snippet by setting default values with the `field` function
of `dataclasses`.

### Settings from files

Of course, you might want or need to read the configuration from a file, let's
say a `.env` one. Such a file will look like this:

```text
DB_DRIVERNAME=postgresql+psycopg
DB_USERNAME=user
...
```

I don't want to install `python-dotenv`, it's yet another dependency that I
think isn't needed. A simple alternative is to create a class method that
receives a file path, and build the settings from there. I'll also create a
couple helper functions to find the root of the project since environment and
configuration files are often at the root.

Still in the `config.py`, I'll add:

```python
from pathlib import Path

PROJ_ROOT = Path(__file__).parents[1]


def get_project_root() -> Path:
    return PROJ_ROOT
```

Now, to get a configuration file from the root, you can do:

```python
from app import config

env_file = config.get_project_root() / ".env"
# ...
```

We'll want to be able to read files and generate a settings object straight from an
`.env` file, I've made some choices about how to read the file: only accepting prefixed
variables and ignoring extraneous variables.

Here's the abridged version of the `DBSettings` object with a new prefix as well
as a couple tools annotations since they are static analysis tools and I am
relying on dynamic typing by fetching the types of the fields straight from the
class definition.

```python
from dataclasses import dataclass, fields
from pathlib import Path
from typing import ClassVar, Self, final, override


@dataclass(frozen=True, kw_only=True)
class DBSettings:
    env_prefix: ClassVar[str] = "DB_"
    drivername: str
    username: str
    password: Secret
    host: str
    port: int
    database: str

    @classmethod
    def from_dotenv(cls: type[Self], fp: Path) -> Self:
        cls_fields = fields(cls)
        key_type_map: dict[str, type] = {  # pyright: ignore[reportAssignmentType]
            f.name: f.type for f in cls_fields if not f.name.startswith("_")
        }
        data: dict[str, str | Secret | int] = {}
        lines = [
            line
            for line in fp.read_text().lstrip().splitlines()
            if line.startswith(cls.env_prefix)
        ]
        for line in lines:
            key, val = (s.strip() for s in line.split("="))
            key = key.lower().removeprefix(cls.env_prefix.lower())
            if key in key_type_map:
                dtype = key_type_map[key]
                data[key] = dtype(val)
        return cls(**data)  # pyright: ignore[reportArgumentType]
```

I think this is a decently useful implementation which can be extended or copied
for anyone's needs.

## On testing and debugging

Thanks to the approach outlined above, creating test specific settings becomes
only a matter of creating a settings object with a testing configuration. This
lowers the bar for testing and shortens the feedback loop when changing the
configuration and re-running the tests.

When you're debugging, it only becomes a matter of setting a breakpoint or
reading the logs, to access the state of your application defined by the
settings. You don't need to chase down some global variables, print the whole
environment or trace back what file accessed some _config_ in what order.

This is true in many more cases than simply settings and configuration objects.
Any time you can reduce the surface of the code provoking state changes and
localize the effects of those changes, you should do so, it will pay back
dividends in clearer and tighter feedback loops.

## Tools nudging you in the right (and wrong) direction

I'll stand on the shoulders of giants and show some libraries I think do a good
job showing users how to use settings properly and which are not.

In terms of settings, the web framework FastAPI has a [guide on how to define
settings](https://fastapi.tiangolo.com/advanced/settings/) which uses
`pydantic-settings`, mentioned earlier.

SQLAlchemy [also explains how to
configure](https://docs.sqlalchemy.org/en/20/core/engines.html) its engine and
connections properly through settings.

In both cases, they nudge you in the right direction since they receive some
form of settings objects which is passed around the application and you're free
to define them as you want, most of the time.

When you go into the data science realm though, you often encounter issues both
for historic and tooling reasons. [Jupyter notebooks](https://jupyter.org/)
which are really popular for experimenting encourage having everything in one
notebook and run some cells or all cells when there are some changes to the
code. As a result, it's frequent for developers to put important variables in
the environment or in global variables, at the top of the notebook. When the
code is moved to production, those patterns stay as well and "infect" everything
with mutable global state.

In my experience, a common frustrating case is when you're dealing with
[anaconda](https://docs.conda.io/en/latest/) and tools that require to be in the front
row seat like the job scheduler, [Airflow](https://airflow.apache.org/). If you, like me,
have an instance of Airflow running on a server with a lot of projects, it can be tough.
Looking at Airflow's documentation
[on its configuration](https://airflow.apache.org/docs/apache-airflow/stable/howto/set-config.html)
as well as
[its module documentation](https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/modules_management.html),
there is a ton of explanation in how to export environment variables, modify the
`PYTHONPATH` environment variables, and so on. Being able to use configuration files like
`airflow.cfg` is great but the fact the documentation is falling back on global
environment variables isn't. Same for
[Airflow variables](https://airflow.apache.org/docs/apache-airflow/stable/howto/variable.html)
where the first example includes using environment variables. If you mix bash and python,
typically when retrieving data and then processing it, you're in for a world of hurt. I
also mention `conda` because it's typical to activate an environment globally for a
project to have access to all its executables, in tandem with Airflow, the state of your
application quickly becomes tough to manage.

Now, you can take some steps to alleviate the issues I just mentioned:

- Use files over hard-coded environment variables when possible.
- Create settings objects and use them in your DAG files (or similar for other
  tools) as much as possible.
- Avoid setting variables in the global environment (`os.environ`) and pass down
  the required data to each class/function/DAG, when needed.
- Use absolute paths to refer to executables and register them once in a
  settings object.
- Conversely, avoid global mutation like `source <file>`, `conda activate`, etc.

## Code

You can read the [full code and
tests](https://github.com/Nathan-Furnal/settings-management) to get a better
picture of the final result.

## Conclusion

There's little consensus on how to manage settings properly in Python projects,
but in general you'll want your settings to be local, immutable and act as plain
data. Python's standard library offers a way to do it with frozen `dataclasses`
and we can easily add some functionalities to read from configuration files like
a `.env` file.

Some tools will nudge you in the right direction because they use and document
settings objects while some other tools make it harder and you need to be more
vigilant in how you define the configuration and make it accessible to different
parts of the code.
