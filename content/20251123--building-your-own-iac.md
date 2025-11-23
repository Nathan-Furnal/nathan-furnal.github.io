+++
title = "Building your own Infrastructure as Code"
slug = "building-your-own-iac"
date = 2025-11-23
tags = ["python"]
categories = ["experimental"]
+++

## Motivation

If you've been working with Infrastructure as Code (IaC) in any capacity, you'll
be used to write Terraform, Bicep or YAML to define the expected end state of
your system, from configuration files. Then, your cloud provider or self-hosted
platform will auto-magically turn those configuration files into internal API
calls and you'll find out at runtime if the end state is actually reachable.

In some occasions though, your IaC tool of predilection won't have a provider
for the platform you're working with or, the platform doesn't support an SDK for
a language your team is familiar with. In addition, if you want to keep the
benefits of IaC: other developers being able to quickly change configurations,
tracking the config files in version control, avoid making manual API calls…
You might want to build your own, light version, of IaC.

In that case, here are some experiments and considerations I've been running
into. I'll be giving examples working with the GitHub API but the logic applies
to any platform exposing an API.

## Defining a data model

## Testing strategy

## Building in three steps

### Reading the configuration

### Creating valid objects

### Applying business rules

## Reconciling configuration and infrastructure
