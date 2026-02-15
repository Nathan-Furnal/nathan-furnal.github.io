+++
title = "Building your own Infrastructure as Code"
slug = "building-your-own-iac"
date = 2025-12-23
tags = ["python"]
categories = ["experimental"]
+++

## Motivation

If you've been working with Infrastructure as Code (IaC) in any capacity, you'll be used
to writing Terraform, Bicep or YAML to define the expected end state of your system, from
configuration files. Then, your cloud provider or self-hosted platform will
auto-magically turn those configuration files into internal API calls and you'll find out
at runtime if the end state is actually reachable.

In some occasions though, your IaC tool of predilection won't have a provider for the
platform you're working with or, the platform doesn't support an SDK for a language your
team is familiar with. In addition, if you want to keep the benefits of IaC: other
developers being able to quickly change configurations, tracking the config files in
version control, avoid making manual API calls… You might want to build your own, light
version, of IaC.

In that case, here are some experiments and considerations I've been running into. I'll
be giving examples working with the GitHub API but the logic applies to any platform
exposing an API.

## Defining a data model

What I mean by "data model" here, is what kind of values and overall content will be
considered valid in the files you expose to others. In my case, I chose to use YAML
because others in the team are already familiar with the format and the review process
with that kind of files. Here, the data model will be the fields that are considered
legal and the values that are valid or not.

This step requires the most thinking upfront. You'll need to define how your
users (other developers in this case) will use the IaC files, **regardless** of
how the final API is called. I think it's important, as described by Hynek
Schlawack [in "Design Pressure"](https://hynek.me/talks/design-pressure/), that
you do not try to match the shape of your data to the API you end up calling. It
will impose artificial constraints on your users and force your data into a
shape you might not want or need.

A good rule of thumb for how to structure that data is by looking at both
frequent and important changes. Frequent changes should be easy to make and
important changes should be possible. If you have a lot of repositories that are
added and removed because you work on Proof-of-Concept (POC) projects, then
optimize for that. If many different teams are working on a few resources, make
manipulating teams and adding users to teams easy. If provisioning a couple
resources happens monthly yet has a large impact on deployments, dedicate a
specific format to it and lock it down but make it easy to add new resources as
well.

Let's start with an example where for my usage, I want all repositories to
appear in a single file with whom can access it. Preferably, the access is bound
to teams, to avoid the manual work of adding and removing singular users
constantly. That being said, it is inevitable that people outside well-defined
teams might need access. I also want to see at a glance what kind of rights each
of those entities has on a repository. We have the constraints:

- Have a list of repositories.
- Define per-team access on a repository.
- Define per-user access on a repository.
- Define access level for each team or user.

Here's what it might look like, in a `repositories.yml` file.

```yaml
- name: example_repo

- name: another_repo
  users:
      - name: dev1
        access: read
      - name: dev2
        access: write
  teams:
      - name: platform
        access: maintain

- name: platform_repo
  teams:
      - name: platform
        access: admin
```

From that file, it should be possible to create repositories and define who has
access to it. Users and teams can be defined in another `users.yml` file (or
`users.yml` and `teams.yml`). One repository entry might turn out to be several
API calls to the GitHub API. It doesn't matter, I only care about ease of use
and change.

Here, I've made a commitment to a certain data shape. If you have the
opportunity to try it out with a couple users first, it's a good time to do so
now. For the longer you stick with a schema, the harder it is to change down the
line.

Right now, I have only defined the shape of the data, there's yet to be a
discussion on business rules I'll want to impose, like naming patterns or
forcing at least one maintainer on a repository. The next step though should be
to define a testing strategy, which will also guide our design.

## Design

I think that a great way to approach many software problems is to build what
Mark Seemann (and probably many others) calls a "vertical slice" in ["Code that
fits in your
head"](https://blog.ploeh.dk/2021/06/14/new-book-code-that-fits-in-your-head/).
In short, implementing a working feature as soon as possible, traversing the
whole implementation from top to bottom. Hence, a vertical slice. I also want to
shout out Alexis King's ["Parse, don't
validate"](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/)
and Gary Bernhardt's ["Functional Core, Imperative
Shell"](https://www.destroyallsoftware.com/screencasts/catalog/functional-core-imperative-shell)
which I will reference multiple times.

In order to test out a vertical slice, what I immediately want is testing that I
can round-trip data in either direction. Meaning that I want to serialize the
YAML to a valid series of requests to the API and that I want to receive data
from the API to compare to the YAML. The latter being used to compare the state
between the YAML file, which should be the ground truth and the API provider
(here, GitHub); this is useful for diagnostics without any state change as well.

There are many ways to slice this but the way I want to approach it, is by
having a common `Provider` interface where implementations will retrieve data
from different sources and then reconcile the differences when possible.

## Building in three steps

### Reading the configuration

### Creating valid objects

### Applying business rules

## Reconciling configuration and infrastructure
