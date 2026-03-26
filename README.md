# Docker Compose Fragments, Anchors, Aliases, Merge Keys, Extensions, and Profiles

A beginner-friendly guide to reusing configuration in `compose.yaml` without turning the file into a copy-paste mess.

---
## Table of contents

- [Why this matters](#why-this-matters)
- [The short definitions](#the-short-definitions)
- [The most important mental model](#the-most-important-mental-model)
- [Part 1: A simple, useful example](#part-1-a-simple-useful-example)
- [Part 2: Alias vs merge, side by side](#part-2-alias-vs-merge-side-by-side)
- [Part 3: A realistic example with 3 services](#part-3-a-realistic-example-with-3-services)
- [Part 4: Overriding a merged block](#part-4-overriding-a-merged-block)
- [Part 5: Profiles for optional services](#part-5-profiles-for-optional-services)
- [Part 6: Complete example](#part-6-complete-example)
- [Part 7: Common mistakes beginners make](#part-7-common-mistakes-beginners-make)
- [Part 8: Practical rules of thumb](#part-8-practical-rules-of-thumb)
- [Final takeaway](#final-takeaway)

---

## Why this matters

As soon as a Compose file has more than one or two services, repetition starts showing up everywhere:

- the same restart policy
- the same environment variables
- the same logging settings
- the same networks
- the same healthcheck options

That repetition is annoying, but the real problem is maintenance. If the same block is copied into three services and later only two copies get updated, the file becomes inconsistent.

This is where **YAML reuse features** help.

In Docker Compose documentation, this kind of reuse is often described under **fragments**. Under the hood, these are standard YAML features that Compose understands:

- `&name` → **anchor**
- `*name` → **alias**
- `<<:` → **merge key**

And when some services should be optional, Compose adds:

- `profiles:` → start only certain services when needed

---

## The short definitions

### 1. Anchor: `&name`
An **anchor** gives a YAML value a reusable name.

```yaml
x-app-defaults: &app_defaults
  restart: unless-stopped
  networks:
    - appnet
```

Here, `&app_defaults` is the anchor.

---

### 2. Alias: `*name`
An **alias** reuses the anchored value **as-is**.

```yaml
environment: &common_env
  APP_ENV: production
  LOG_LEVEL: info

# later
environment: *common_env
```

Here, `*common_env` means: “use that earlier value here unchanged.”

---

### 3. Merge key: `<<:`
The **merge key** is used when the anchored value is a **mapping** (a key/value object) and that mapping should be merged into another mapping.

```yaml
service:
  <<: *app_defaults
  image: nginx:alpine
```

That means: “take the key/value pairs from `app_defaults` and merge them into this mapping.”

This is the pattern most often used for service defaults.

---

## The most important mental model

There are two different kinds of reuse:

- `*alias` = reuse a value unchanged
- `<<: *alias` = merge a mapping into another mapping so it can be extended or overridden

That distinction matters much more than “list vs mapping”. An alias can point to a list, mapping, or scalar. A merge key only works with mappings.

A full side-by-side example appears in **Part 2: Alias vs merge, side by side**.

---

# Part 1: A simple, useful example

A realistic starter setup:

- `web` = frontend container
- `api` = backend container
- `db` = PostgreSQL

`web` and `api` should share some defaults:

- same restart policy
- same network
- same logging options

## Without anchors and fragments

```yaml
services:
  web:
    image: nginx:alpine
    restart: unless-stopped
    networks:
      - appnet
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  api:
    image: mycompany/api:latest
    restart: unless-stopped
    networks:
      - appnet
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    environment:
      APP_ENV: production

  db:
    image: postgres:16
    restart: unless-stopped
    networks:
      - appnet
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret

networks:
  appnet:
```

This works, but it repeats a lot.

---

## Better: reuse common service defaults

```yaml
x-service-defaults: &service_defaults
  restart: unless-stopped
  networks:
    - appnet
  logging:
    driver: json-file
    options:
      max-size: "10m"
      max-file: "3"

services:
  web:
    <<: *service_defaults
    image: nginx:alpine
    ports:
      - "8080:80"

  api:
    <<: *service_defaults
    image: mycompany/api:latest
    environment:
      APP_ENV: production

  db:
    image: postgres:16
    restart: unless-stopped
    networks:
      - appnet
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret

networks:
  appnet:
```

## What improved?

`web` and `api` now inherit:

- `restart: unless-stopped`
- the `appnet` network
- the logging block

That block is written once and reused in both services.

---

## Why `x-service-defaults` starts with `x-`

This is a very common Compose convention.

Keys starting with `x-` are **extensions**: helper blocks meant for reuse. Compose ignores them as top-level runtime configuration, which makes them a good place to store anchors.

That means this:

```yaml
x-service-defaults: &service_defaults
  restart: unless-stopped
  networks:
    - appnet
```

is basically saying:

> “This is not a service. This is a reusable template block.”

Using `x-` is not mandatory, but in Compose files it is usually the clearest pattern.

---

# Part 2: Alias vs merge, side by side

This is the section most beginners need.

```yaml
x-env: &common_env
  APP_ENV: production
  LOG_LEVEL: info

services:
  api:
    image: mycompany/api:latest
    environment: *common_env

  worker:
    image: mycompany/worker:latest
    environment:
      <<: *common_env
      QUEUE: emails
```

## What is happening here?

### `&common_env`
Creates an anchor named `common_env`.

### `environment: *common_env`
Reuses the whole environment mapping **unchanged**.

So `api` gets:

```yaml
environment:
  APP_ENV: production
  LOG_LEVEL: info
```

### `environment: { <<: *common_env, ... }`
Reuses the common environment as a starting point, then adds more keys.

So `worker` gets:

```yaml
environment:
  APP_ENV: production
  LOG_LEVEL: info
  QUEUE: emails
```

## The practical rule

- use `*common_env` when the exact same block is needed
- use `<<: *common_env` when shared values are needed **plus** service-specific additions or overrides

---

## Why not always use `<<:`?

Because merge and alias solve different problems.

This is perfectly valid and often the cleanest option:

```yaml
environment: *common_env
```

There is no need to wrap that in a merge when nothing should be changed.

A merge becomes useful only when a new mapping must be built from the old one:

```yaml
environment:
  <<: *common_env
  API_PORT: "8080"
```

---

## Why `<<:` does not work for lists

The merge key works only on **mappings**, not on sequences/lists.

This is valid:

```yaml
x-env-list: &env_list
  - APP_ENV=prod
  - LOG_LEVEL=info

services:
  api:
    environment: *env_list
```

Here, `*env_list` just reuses the list unchanged.

This is **not** valid YAML merge usage:

```yaml
environment:
  <<: *env_list
```

Why? Because `*env_list` points to a **list**, but `<<:` only merges **mappings**.

So:

- alias can reuse lists, mappings, or scalars
- merge only works with mappings

---

# Part 3: A realistic example with 3 services

A more realistic application setup:

- `api`
- `worker`
- `scheduler`

They all use the same image.
They all need:

- a shared env block
- the same restart policy
- the same network
- the same database connection settings

But each service has a different command.

## Example

```yaml
x-app-base: &app_base
  image: mycompany/laravel-app:latest
  restart: unless-stopped
  networks:
    - backend
  environment: &app_env
    APP_ENV: production
    APP_DEBUG: "false"
    DB_HOST: db
    DB_PORT: "5432"
    DB_DATABASE: app
    DB_USERNAME: app
    DB_PASSWORD: secret

services:
  api:
    <<: *app_base
    command: php artisan serve --host=0.0.0.0 --port=8000
    ports:
      - "8000:8000"

  worker:
    <<: *app_base
    command: php artisan queue:work --verbose --tries=3

  scheduler:
    <<: *app_base
    command: sh -c "while true; do php artisan schedule:run; sleep 60; done"

  db:
    image: postgres:16
    restart: unless-stopped
    networks:
      - backend
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data

networks:
  backend:

volumes:
  pgdata:
```

## Why this is good

The repeated application setup lives in one place:

- image
- restart
- network
- common environment

Each service only declares what is unique:

- `command`
- `ports` for `api`

That is the sweet spot: **shared defaults above, service-specific details below**.

---

# Part 4: Overriding a merged block

A common question is:

> “What if one service should inherit the defaults, but change one value?”

That is exactly where merge keys shine.

## Example

```yaml
x-app-base: &app_base
  image: mycompany/app:latest
  restart: unless-stopped
  environment: &common_env
    APP_ENV: production
    LOG_LEVEL: info
    CACHE_DRIVER: redis

services:
  api:
    <<: *app_base

  worker:
    <<: *app_base
    environment:
      <<: *common_env
      LOG_LEVEL: debug
      WORKER_CONCURRENCY: "5"
```

## Result

`api` gets:

```yaml
environment:
  APP_ENV: production
  LOG_LEVEL: info
  CACHE_DRIVER: redis
```

`worker` gets:

```yaml
environment:
  APP_ENV: production
  LOG_LEVEL: debug
  CACHE_DRIVER: redis
  WORKER_CONCURRENCY: "5"
```

So the worker:

- inherits the common env
- overrides `LOG_LEVEL`
- adds `WORKER_CONCURRENCY`

That is a very common pattern in real Compose files.

---

# Part 5: Profiles for optional services

Profiles let some services stay optional.

Typical use cases:

- a database admin UI that only starts sometimes
- a mail testing tool for development
- a debugging container
- a one-off helper service

## The key idea

- services **without** `profiles:` start normally
- services **with** `profiles:` start only when that profile is enabled, or when that service is explicitly targeted on the command line

---

## Simple profile example

```yaml
services:
  api:
    image: mycompany/api:latest
    ports:
      - "8000:8000"

  db:
    image: postgres:16

  adminer:
    image: adminer
    profiles:
      - debug
    ports:
      - "8081:8080"
```

### What happens?

If this command is run:

```bash
docker compose up -d
```

Compose starts:

- `api`
- `db`

But not `adminer`, because `adminer` belongs to the `debug` profile.

If this command is run:

```bash
docker compose --profile debug up -d
```

Compose starts:

- `api`
- `db`
- `adminer`

---

## Starting only one profiled service

There are two useful patterns.

### Enable the whole profile

```bash
docker compose --profile debug up -d
```

This starts:

- all normal services
- all services assigned to `debug`

### Target one profiled service directly

```bash
docker compose up -d adminer
```

When a profiled service is explicitly targeted on the command line, it can run without manually enabling its profile.

In that case, Compose starts:

- `adminer`
- any declared dependencies of `adminer`

It does **not** automatically start other services that merely share the same profile.

That makes this very useful for one-off helper services.

---

# Part 6: Complete example

This `compose.yaml` combines:

- extension blocks with `x-`
- anchors
- aliases
- merge keys
- shared env
- shared service defaults
- 3 application services
- 2 optional profile-based services

```yaml
x-app-service: &app_service
  image: mycompany/shop-api:latest
  restart: unless-stopped
  networks:
    - backend
  depends_on:
    - db
    - redis
  environment: &app_env
    APP_ENV: production
    APP_DEBUG: "false"
    DB_HOST: db
    DB_PORT: "5432"
    DB_DATABASE: shop
    DB_USERNAME: shop
    DB_PASSWORD: secret
    REDIS_HOST: redis
    REDIS_PORT: "6379"

x-debug-service: &debug_service
  restart: unless-stopped
  networks:
    - backend
  profiles:
    - debug

services:
  api:
    <<: *app_service
    command: php artisan octane:start --server=roadrunner --host=0.0.0.0 --port=8000
    ports:
      - "8000:8000"

  worker:
    <<: *app_service
    command: php artisan queue:work --tries=3
    environment:
      <<: *app_env
      QUEUE_CONNECTION: redis
      WORKER_CONCURRENCY: "4"

  scheduler:
    <<: *app_service
    command: sh -c "while true; do php artisan schedule:run; sleep 60; done"

  db:
    image: postgres:16
    restart: unless-stopped
    networks:
      - backend
    environment:
      POSTGRES_DB: shop
      POSTGRES_USER: shop
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    networks:
      - backend

  adminer:
    <<: *debug_service
    image: adminer
    depends_on:
      - db
    ports:
      - "8081:8080"

  mailpit:
    <<: *debug_service
    image: axllent/mailpit
    ports:
      - "8025:8025"
      - "1025:1025"

networks:
  backend:

volumes:
  pgdata:
```

---

## How to use this file

### Start the normal application

```bash
docker compose up -d
```

This starts:

- `api`
- `worker`
- `scheduler`
- `db`
- `redis`

It does **not** start:

- `adminer`
- `mailpit`

because those are in the `debug` profile.

---

### Start the application with debug tools

```bash
docker compose --profile debug up -d
```

Now Compose starts everything above plus:

- `adminer`
- `mailpit`

---

### Start only one helper service

```bash
docker compose up -d adminer
```

That starts `adminer` and its declared dependencies. It does not automatically start every other service in the `debug` profile.

---

### Preview the fully expanded config

A very useful learning and debugging command is:

```bash
docker compose config
```

This shows the resolved Compose model after aliases and merges have been applied.

When fragments feel confusing, this command is often the fastest way to see what Compose actually received.

---

# Part 7: Common mistakes beginners make

## Mistake 1: Thinking alias and merge are interchangeable

They are not.

Good when the whole value should be reused unchanged:

```yaml
environment: *common_env
```

Good when a mapping should be reused and then extended:

```yaml
environment:
  <<: *common_env
  EXTRA_FLAG: "1"
```

The first says “use exactly this value.” The second says “start with this mapping, then modify it.”

---

## Mistake 2: Using `<<:` on a list

The merge key works on **mappings**, not lists.

Good:

```yaml
x-env: &common_env
  APP_ENV: production
  LOG_LEVEL: info

services:
  api:
    environment:
      <<: *common_env
      EXTRA_FLAG: "1"
```

Not good:

```yaml
x-ports: &common_ports
  - "8000:8000"

services:
  api:
    ports:
      <<: *common_ports
```

That does not work the way most people expect.

For lists, either reuse the whole value with `*alias` or repeat the list if it is short.

---

## Mistake 3: Anchoring the wrong level

This:

```yaml
x-env: &common_env
  APP_ENV: production
```

is very different from this:

```yaml
environment: &common_env
  APP_ENV: production
```

The anchor applies to the exact YAML node where it is placed.

So it matters whether the anchor represents:

- an `environment` mapping
- a whole service block
- a `logging` block
- a `healthcheck` block

---

## Mistake 4: Forgetting that later values override earlier merged values

In a merge like this:

```yaml
environment:
  <<: *common_env
  LOG_LEVEL: debug
```

`LOG_LEVEL: debug` wins over the merged value.

That override behavior is often exactly what is wanted.

---

# Part 8: Practical rules of thumb

When deciding what to do, these rules usually work well:

- Use `x-...` top-level extension blocks to store reusable Compose snippets.
- Anchor reusable mappings such as service defaults, environment blocks, logging blocks, or healthchecks.
- Use `*alias` when the exact same value should be reused unchanged.
- Use `<<: *alias` when a mapping should be reused and then extended or overridden.
- Prefer mapping syntax for `environment` when merges are needed.
- Use `profiles` for optional tools such as Adminer, Mailpit, or debugging helpers.
- Use `docker compose config` to inspect the final expanded result.

---

# Final takeaway

The most useful beginner summary is this:

- **Anchor** = give a YAML value a name
- **Alias** = reuse that value unchanged
- **Merge key** = reuse a mapping and extend or override it
- **Extension (`x-...`)** = a clean place to store reusable Compose blocks
- **Profile** = make a service optional

Once that mental model clicks, Compose files become much easier to read, scale, and maintain.
