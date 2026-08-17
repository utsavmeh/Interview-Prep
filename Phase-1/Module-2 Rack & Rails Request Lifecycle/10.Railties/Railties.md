# Railties — Beginner-Friendly Senior Rails Study Guide

> **Scope:** Modern Rails 7.1–8.1. Some initializer names/APIs can change between versions, so always check the exact Rails version in your `Gemfile.lock`. The overall Railties architecture has been stable since Rails 3.

---

# 1. What is Railties?

**Railties is the glue that connects Rails components and Rails gems to your application.**

Rails is made of many separate parts:

```text
Active Record
Action Controller
Action Mailer
Active Job
Action View
...
```

Each part can use a **Railtie** to tell Rails:

> "When Rails starts, I need you to configure or initialize me."

The main class is:

```ruby
Rails::Railtie
```

A Railtie can:

* add configuration
* register initializers
* add middleware
* add rake tasks
* add generators
* hook into Rails reloads
* integrate with Rails frameworks

---

## Railtie vs Engine vs Application

Think of them as levels:

```text
Rails::Railtie
      ↓
Basic Rails integration

Rails::Engine
      ↓
Reusable mini Rails application
(routes, models, controllers, views, etc.)

Rails::Application
      ↓
Your actual Rails application
```

So:

* **Railtie** → integration
* **Engine** → mini application
* **Application** → main application

---

## Why do we need Railties?

Without Railties, every Rails application would need to manually decide:

* what loads first;
* when Active Record initializes;
* when middleware is added;
* how gems add configuration;
* when reload callbacks run;
* how generators/tasks are registered.

Railties gives Rails a standard way to coordinate all of this.

---

## When should a gem use a Railtie?

Use a Railtie when a gem needs to integrate with Rails.

For example:

```text
MyGem
 ├── configuration
 ├── middleware
 ├── Rails callbacks
 ├── rake tasks
 └── generators
```

Don't use a Railtie just because your code is inside `lib/`.

Also, don't put normal business logic inside a Railtie.

---

# Common misconceptions

| Misconception                                     | Reality                                                                  |
| ------------------------------------------------- | ------------------------------------------------------------------------ |
| Railtie = Engine                                  | Engine is a more powerful type of Railtie                                |
| Every gem needs a Railtie                         | No. Only gems needing Rails integration                                  |
| Initializers run on every request                 | No. They run during boot                                                 |
| Initializer file order determines execution order | No. Rails uses dependencies such as `before`/`after`                     |
| `after_initialize` runs after every reload        | No. Use `to_prepare` for reload-aware setup                              |
| Railties execute SQL                              | No. Railties configure Active Record; Active Record/database execute SQL |

---

# 2. The `Rails::Railtie` Contract

A basic Railtie looks like this:

```ruby
module AcmeAudit
  class Railtie < Rails::Railtie
    config.acme_audit = ActiveSupport::OrderedOptions.new
    config.acme_audit.enabled = true

    initializer "acme_audit.configure" do |app|
      AcmeAudit.configure(
        enabled: app.config.acme_audit.enabled
      )
    end
  end
end
```

There are two important things here:

### Configuration

```ruby
config.acme_audit.enabled = true
```

This gives the application a setting.

### Initializer

```ruby
initializer "acme_audit.configure"
```

This tells Rails:

> "Run this setup during application boot."

---

# 3. How Rails Finds a Railtie

Rails doesn't search every installed gem looking for Railties.

The gem normally loads its Railtie itself:

```ruby
# lib/acme_audit.rb

require "acme_audit/client"
require "acme_audit/railtie" if defined?(Rails::Railtie)
```

The conditional is important.

It means:

```text
Rails available?
    │
    ├── Yes → load Railtie
    │
    └── No  → don't load Rails integration
```

So the gem can still work as a normal Ruby library.

Typically:

```ruby
Bundler.require(*Rails.groups)
```

loads the gem, which loads its entrypoint, which then loads the Railtie.

So:

```text
Bundler
  ↓
Gem
  ↓
Gem entrypoint
  ↓
Railtie
  ↓
Rails registers it
```

This is why **require order matters**.

---

# 4. Initializers

An initializer is code that Rails runs during application boot.

Example:

```ruby
initializer "acme_audit.configure" do |app|
  AcmeAudit.configure(
    logger: app.config.logger
  )
end
```

You can also control its position:

```ruby
initializer "acme_audit.middleware",
  after: "active_record.initialize_database" do |app|

  # setup
end
```

This means:

```text
Active Record initializer
          ↓
Acme Audit initializer
```

---

## Initializers are a graph

Rails doesn't simply run initializer files from top to bottom.

Instead:

```text
A
↓
B
↓
C
```

If B says:

```ruby
after: "A"
```

Rails knows:

```text
A → B
```

Rails uses **TSort** to calculate a valid order.

### Interview point

> **Initializer execution order is based on dependency relationships, not simply source/file order.**

---

## Important rules

### Give initializers unique names

Good:

```ruby
initializer "acme_audit.configure"
```

Bad:

```ruby
initializer "configure"
```

Use the gem name as a namespace.

---

### Don't add unnecessary dependencies

Only write:

```ruby
after: "something"
```

when your code actually depends on that ordering.

---

### Cycles cause boot failure

For example:

```text
A after B
B after A
```

Rails can't determine the correct order, so boot fails.

---

### Check the real initializer order

```bash
bin/rails initializers
```

This is especially useful when debugging Rails upgrades.

---

# 5. Configuration

A Railtie can provide default configuration:

```ruby
module FeatureFlags
  class Railtie < Rails::Railtie
    config.feature_flags = ActiveSupport::OrderedOptions.new
    config.feature_flags.adapter = :redis
    config.feature_flags.namespace = "flags"
  end
end
```

The application can override it:

```ruby
config.feature_flags.adapter = :database
```

Think:

```text
Gem defaults
    ↓
Application overrides
    ↓
Final configuration
```

---

## `ActiveSupport::OrderedOptions`

It allows convenient access:

```ruby
config.feature_flags.adapter
```

instead of:

```ruby
config.feature_flags[:adapter]
```

But remember:

* it's mutable;
* typos can return `nil`.

For complicated gems, it's better to validate configuration and copy it into your own final configuration object.

---

# 6. Rails Lifecycle Hooks

There are several important hooks.

## `before_configuration`

Runs very early:

```text
Application starts
      ↓
before_configuration
      ↓
configuration continues
```

---

## `before_initialize`

Runs after configuration has been assembled but before the main initializer graph:

```text
Configuration
      ↓
before_initialize
      ↓
Initializers
```

---

## `after_initialize`

Runs after the initializer graph:

```text
Initializers
      ↓
after_initialize
```

These are **broad lifecycle hooks**.

If you need precise ordering, prefer a named initializer:

```ruby
initializer "my_gem.setup",
  after: "some_initializer"
```

---

# 7. `config.to_prepare`

This is important when working with Rails reloads.

```ruby
config.to_prepare do
  # setup
end
```

In development:

```text
Rails boots
   ↓
to_prepare runs
   ↓
Code changes
   ↓
Rails reloads
   ↓
to_prepare runs again
```

In production, where reloading is normally disabled:

```text
Rails boots
   ↓
to_prepare runs once
```

---

## The important rule: idempotence

Your `to_prepare` code must be safe to run multiple times.

For example:

```ruby
config.to_prepare do
  Order.prepend(OrderAuditExtension) unless
    Order < OrderAuditExtension
end
```

The check prevents the extension from being repeatedly added.

Be especially careful with:

```ruby
ActiveSupport::Notifications.subscribe(...)
```

If you subscribe every time `to_prepare` runs, you can accidentally create:

```text
1 subscriber
2 subscribers
3 subscribers
4 subscribers
...
```

after repeated reloads.

---

# 8. `ActiveSupport.on_load`

`on_load` is useful when you want to extend a Rails framework **when it becomes available**.

Example:

```ruby
ActiveSupport.on_load(:active_record) do
  include MyGem::ModelMethods
end
```

Instead of assuming Active Record is already loaded:

```ruby
ActiveRecord::Base.include MyGem::ModelMethods
```

you say:

```text
"When Active Record loads, run this."
```

This makes your gem more flexible.

### Simple difference

```text
on_load
    ↓
Run when a framework loads

to_prepare
    ↓
Run when Rails prepares/reloads code
```

---

# 9. Middleware

Railties can modify the Rails middleware stack.

Example:

```ruby
initializer "acme_audit.middleware" do |app|
  app.middleware.insert_after(
    Rack::Head,
    AcmeAudit::RequestMiddleware
  )
end
```

Middleware looks like:

```text
Request
   ↓
Middleware A
   ↓
Middleware B
   ↓
Controller
   ↓
Middleware B
   ↓
Middleware A
   ↓
Response
```

Request goes:

```text
top → bottom
```

Response comes back:

```text
bottom → top
```

Therefore **middleware order matters**.

It can affect:

* authentication
* logging
* CORS
* exception handling
* request IDs
* security

Check the actual stack with:

```bash
bin/rails middleware
```

---

# 10. Engines

An Engine is essentially a reusable mini Rails application.

```ruby
module Billing
  class Engine < ::Rails::Engine
    isolate_namespace Billing
  end
end
```

It can contain:

```text
Billing Engine
├── models
├── controllers
├── views
├── routes
├── migrations
├── assets
└── middleware
```

The host application can mount it:

```ruby
mount Billing::Engine, at: "/billing"
```

---

## `isolate_namespace`

```ruby
isolate_namespace Billing
```

helps keep the engine's:

* routes
* helpers
* controllers
* models
* naming

separate from the host application.

But:

> **Namespace isolation is NOT a security boundary.**

It doesn't automatically provide:

* separate processes
* separate databases
* authorization
* tenant isolation

---

# 11. Rails Tasks and Generators

Railties can register:

### Rake tasks

```ruby
rake_tasks do
  load File.expand_path("../tasks/my_gem.rake", __dir__)
end
```

### Generators

```ruby
generators do
  require "generators/my_gem/install/install_generator"
end
```

### Console hook

```ruby
console do
  puts "MyGem enabled"
end
```

### Runner hook

```ruby
runner do |app|
  MyGem.configure(logger: app.config.logger)
end
```

These blocks **register behavior**.

They don't automatically mean the code runs immediately.

The command you're executing decides which registered blocks are used.

---

# 12. What Happens During Rails Boot?

When you run:

```bash
bin/rails server
```

the simplified flow is:

```text
bin/rails
    ↓
config/boot.rb
    ↓
Bundler
    ↓
Rails commands
    ↓
config/application.rb
    ↓
Rails frameworks loaded
    ↓
Gems loaded
    ↓
Railties collected
    ↓
Initializers collected
    ↓
Initializer graph sorted
    ↓
Initializers executed
    ↓
Routes / middleware / autoloading prepared
    ↓
Rack application ready
```

Let's understand the important steps.

---

## Step 1 — `bin/rails`

You run:

```bash
bin/rails server
```

Rails loads:

```text
config/boot.rb
```

which sets up Bundler and the environment needed to load Rails.

---

## Step 2 — Rails command system

Rails sees:

```text
server
```

and knows:

> "The user wants to start the server."

Eventually the Rails application is handed to the Rack server.

---

## Step 3 — `config/application.rb`

Rails loads your application:

```ruby
class MyApp::Application < Rails::Application
end
```

If you use:

```ruby
require "rails/all"
```

the Rails framework components are loaded.

Their Railties become available.

For example:

```text
Active Record
    ↓
ActiveRecord::Railtie

Action Controller
    ↓
ActionController::Railtie
```

---

## Step 4 — Gems load

Rails runs something like:

```ruby
Bundler.require(*Rails.groups)
```

Your gems load.

A gem may then load:

```ruby
MyGem::Railtie
```

So Rails now knows:

> "This gem wants to participate in my boot process."

---

## Step 5 — Rails collects Railties

Rails collects:

* framework Railties
* gem Railties
* Engines
* the application itself

Conceptually:

```text
Rails Application
      │
      ├── ActiveRecord Railtie
      ├── ActionController Railtie
      ├── MyGem Railtie
      ├── AnotherGem Railtie
      └── Billing Engine
```

---

## Step 6 — Initializers are collected

Rails takes all the initializers from these components.

For example:

```text
Active Record
 ├── initialize_database
 └── setup_connection

My Gem
 ├── configure
 └── middleware

Engine
 └── routes
```

Rails combines them into one initializer graph.

---

## Step 7 — Rails sorts and executes them

Rails uses:

```text
TSort
```

to determine the correct order based on:

```ruby
before:
after:
```

Then the initializers execute.

---

# 13. What Railties Actually Does During Boot

During boot, Rails and its Railties set up things such as:

* logger
* cache
* notifications
* load paths
* autoloaders
* reloader
* framework integrations
* database configuration
* middleware
* routes

Eventually:

```text
Rails.application
```

becomes a working Rack application.

At that point the server can call:

```ruby
Rails.application.call(env)
```

for HTTP requests.

---

# 14. The Important Internal Structures

| Structure                   | Simple meaning                           |
| --------------------------- | ---------------------------------------- |
| `Rails::Railtie.subclasses` | Railties Rails knows about               |
| `load_index`                | Helps keep Railtie loading deterministic |
| Initializer collection      | Stores all initializers                  |
| `TSort`                     | Calculates initializer order             |
| Application configuration   | Stores application settings              |
| Middleware proxy            | Collects middleware changes              |
| `Rails::Engine::Railties`   | Combines app, engines, and Railties      |
| `ActionDispatch::Reloader`  | Helps coordinate reload callbacks        |

### Useful interview point

If a Railtie doesn't appear among the registered Railties, a likely problem is:

> **Its file was never required.**

---

# 15. Railties and Zeitwerk

This distinction is very important:

> **Railties manages Rails lifecycle/integration. Zeitwerk manages autoloading and eager loading.**

Think:

```text
Railties
   ↓
"When should things be initialized?"

Zeitwerk
   ↓
"How should Rails load these constants?"
```

Modern Rails usually has two important autoloading areas:

```text
main
 ↓
reloadable code

once
 ↓
code that isn't reloaded
```

---

## Development

In development, reloadable constants can be removed and loaded again.

```text
User
 ↓
old User class removed
 ↓
new User class loaded
```

Therefore, don't keep old reloadable classes in long-lived global variables.

---

## Production

Production commonly uses:

```ruby
config.eager_load = true
```

Rails loads the configured code during boot.

Benefits:

* problems are found earlier;
* runtime is more predictable;
* pre-fork servers can benefit from copy-on-write memory sharing.

---

## Important Railtie rule

Don't capture a reloadable application class during boot and keep it forever.

Bad idea:

```ruby
MY_USER_CLASS = User
```

because after reload:

```text
Old User class
     ↓
stored somewhere globally

Rails reloads
     ↓
New User class
```

Now you may have two different `User` class objects.

For reloadable application constants, resolve them at the appropriate lifecycle point instead.

---

# 16. Where PostgreSQL Fits

Railties **does not talk directly to PostgreSQL**.

The simplified runtime flow is:

```text
HTTP request
    ↓
Rails executor
    ↓
Active Record
    ↓
Connection Pool
    ↓
pg gem
    ↓
PostgreSQL
    ↓
Result
    ↓
Active Record
    ↓
Application
```

Railties helps configure Active Record, but it doesn't execute the SQL itself.

This matters because bad Railtie code can still cause database problems.

For example:

```ruby
initializer "my_gem.connect" do
  PG.connect(...)
end
```

is dangerous because it can create connections during boot.

---

# 17. Request and Reload Lifecycle

In development, a simplified flow is:

```text
Code changes
    ↓
Rails detects change
    ↓
to_prepare callbacks
    ↓
Zeitwerk reloads code
    ↓
Rails executor
    ↓
Request
    ↓
Cleanup
```

The exact order of some steps can change between Rails versions.

The important rules are:

* don't cache reloadable classes globally;
* make `to_prepare` code idempotent;
* let Rails manage connection/request lifecycle.

---

# 18. Architecture

Railties sits between Rails frameworks and the application.

```text
CLI
 │
 │ bin/rails
 ↓
Railties
 │
 ├── boot
 ├── configuration
 ├── initializers
 ├── engines
 ├── middleware
 └── reload hooks
 │
 ├──────────────┬──────────────┐
 ↓              ↓              ↓
Active Record  Action Pack   Active Job
 ↓              ↓              ↓
Database        Rack          Queue
 │
 ↓
PostgreSQL
```

Railties is therefore an **orchestration layer**.

It isn't:

```text
MVC
```

and it isn't:

```text
business logic
```

---

# 19. Who Is Responsible for What?

### Rails/framework authors

Provide:

* Railties
* lifecycle hooks
* configuration
* framework integration

### Engine authors

Provide:

* reusable application functionality
* routes
* models/controllers/views
* namespace isolation
* host integration

### Gem authors

Should:

* keep the core library Rails-independent;
* add optional Railtie integration;
* expose clean configuration;
* avoid unnecessary boot work.

### Application developers

Should:

* configure components;
* compose engines/gems;
* use Railties only when lifecycle integration is actually needed.

---

# 20. Real Production Example — Request ID Gem

Imagine a gem that adds request IDs.

It could provide:

```ruby
module RequestContext
  class Railtie < Rails::Railtie
    config.request_context = ActiveSupport::OrderedOptions.new
    config.request_context.header = "HTTP_X_REQUEST_ID"

    initializer "request_context.middleware" do |app|
      app.middleware.insert_before(
        Rails::Rack::Logger,
        RequestContext::Middleware,
        header: app.config.request_context.header
      )
    end
  end
end
```

The Railtie's job is simply:

```text
Read configuration
      ↓
Add middleware
```

It shouldn't create request-specific state during boot.

---

# 21. Real Production Example — Active Record Extension

Suppose a gem wants to add functionality to Active Record models.

Use:

```ruby
ActiveSupport.on_load(:active_record) do
  include TenantScoping::Model
end
```

instead of assuming Active Record is already loaded.

A good gem should also avoid silently changing every model's behavior unless that's explicitly intended.

For example, automatically adding:

```ruby
default_scope
```

to every model can cause serious production problems.

---

# 22. Real Production Example — Engine

A billing engine could look like:

```ruby
module Payments
  class Engine < ::Rails::Engine
    isolate_namespace Payments

    config.generators do |g|
      g.test_framework :rspec
    end
  end
end
```

The host application mounts it:

```ruby
mount Payments::Engine, at: "/payments"
```

The engine should provide a clear configuration interface rather than reaching directly into host application internals.

---

# 23. Migration and Rake Tasks

If a gem needs database migrations, it can provide:

* install generator
* migration files
* rake tasks

But don't automatically modify the production database during application boot.

Bad:

```ruby
initializer "create_tables" do
  # run DDL
end
```

Better:

```text
Migration
   ↓
Deploy
   ↓
Application starts
```

Database changes should be an explicit deployment operation.

---

# 24. Common Mistakes

| Mistake                                     | Why it's bad                                    | Better approach                             |
| ------------------------------------------- | ----------------------------------------------- | ------------------------------------------- |
| Put business logic in a Railtie             | Runs during boot and hides application behavior | Use services/jobs/application code          |
| Use `to_prepare` for migrations/seeds       | It may run repeatedly                           | Use migrations/tasks/deployment             |
| Add middleware without checking position    | Changes request behavior                        | Test middleware order                       |
| Depend on accidental gem loading order      | Can break after dependency changes              | Use explicit `before`/`after`               |
| Cache `User` in a global/class variable     | Reload creates a new `User` class               | Resolve reloadable constants later          |
| Subscribe on every `to_prepare`             | Subscribers can duplicate                       | Register once or carefully manage lifecycle |
| Connect to DB during boot                   | Slows/fails every process start                 | Let Active Record manage connections        |
| Depend heavily on private initializer names | Rails upgrades can break you                    | Prefer public hooks                         |
| Make a generic gem require Rails            | Breaks plain Ruby usage                         | Keep Rails integration optional             |
| Turn Railtie into global application wiring | Creates hidden dependencies                     | Keep Railtie thin                           |

---

# 25. Performance

Railties mainly affects **boot performance**, not normal request CPU.

But boot time matters in:

* containers
* autoscaling
* serverless
* deployments
* pre-fork servers

Avoid doing this during boot:

```text
Network requests
Database queries
Large file loading
Credential API calls
Huge eager requires
```

Keep initialization:

```text
Fast
Deterministic
Minimal
```

---

## Middleware performance

Middleware runs on every request.

So even a small amount of unnecessary work becomes expensive at scale.

For example:

```ruby
request.headers.to_h
```

or expensive logging/tracing in every request can create significant allocations.

Measure middleware using production-like traffic.

---

## Database connections

Don't create PostgreSQL connections inside a Railtie during require/boot.

Especially with pre-fork servers:

```text
Master process
    ↓
DB connection created
    ↓
fork
 ┌──┴──┐
Worker Worker
```

Workers may inherit unsafe connections.

Let Active Record manage connections at the correct process/thread lifecycle.

---

# 26. Security

Railties execute code during application boot, so they are part of your application's security boundary.

### Gems

A gem's Railtie code can run automatically when the gem is loaded.

Therefore:

* review dependencies;
* lock versions;
* scan dependencies.

### Secrets

Never log:

```text
API keys
passwords
private keys
credentials
```

### Middleware

Middleware ordering can affect:

* authentication
* CSRF
* host authorization
* logging
* exception handling

### Engines

Remember:

```ruby
isolate_namespace
```

doesn't provide security.

The engine still needs:

* authentication
* authorization
* CSRF protection
* rate limiting
* auditing

### Security configuration

Fail closed.

For example, if production requires a signing key:

```ruby
raise "signing key required"
```

is safer than silently disabling verification.

---

# 27. Debugging Railties

These commands are extremely useful.

### Initializers

```bash
bin/rails initializers
```

Shows the actual initializer order.

### Middleware

```bash
bin/rails middleware
```

Shows the effective middleware stack.

### Tasks

```bash
bin/rails --tasks
```

Check whether your gem's tasks are registered.

### Zeitwerk

```bash
bin/rails zeitwerk:check
```

Checks autoloading/naming problems.

### Production boot

```bash
RAILS_ENV=production bin/rails runner 'puts Rails.application.class'
```

Useful for reproducing production-only boot problems.

---

# 28. Useful Console Checks

You can inspect Railties with:

```ruby
Rails::Railtie.subclasses.map { |k| [k.name, k.load_index] }
```

Check the application's Railties:

```ruby
Rails.application.railties.all.map(&:class).map(&:name)
```

Check middleware:

```ruby
Rails.application.config.middleware
```

Check Zeitwerk:

```ruby
Rails.autoloaders.main.dirs
```

These are useful for debugging, but some are internal APIs and shouldn't become permanent application dependencies.

---

# 29. Common Problems

| Problem                                | Likely reason                                          |
| -------------------------------------- | ------------------------------------------------------ |
| Railtie doesn't run                    | Its file wasn't required                               |
| `uninitialized constant` during boot   | Constant referenced too early or wrong Zeitwerk naming |
| Behavior duplicates after reload       | `to_prepare` isn't idempotent                          |
| Middleware order is wrong              | Wrong insertion point                                  |
| Production fails but development works | Eager loading/configuration differs                    |
| Console works but server fails         | Different boot/require path or stale process           |
| Circular dependency                    | Initializer `before`/`after` cycle                     |

---

# 30. Best Practices

Remember these rules:

1. **Keep Railties focused on Rails integration.**
2. **Keep the core of a gem Rails-independent.**
3. **Use unique initializer names.**
4. **Prefer public lifecycle hooks.**
5. **Make `to_prepare` idempotent.**
6. **Don't perform unnecessary network/database work during boot.**
7. **Validate configuration early.**
8. **Document Rails version compatibility.**
9. **Test middleware order.**
10. **Test your Railtie inside a real/dummy Rails application.**
11. **Test production eager loading.**
12. **Use `bin/rails initializers` when debugging boot order.**

---

# 31. Anti-Patterns

## 1. Boot-time service locator

Bad:

```ruby
initializer "analytics.everything" do
  Analytics.client =
    Analytics::Client.new(ENV.fetch("ANALYTICS_KEY"))
end
```

Problems:

* global mutable state;
* difficult tests;
* unclear reload behavior;
* unclear fork behavior.

Better:

```text
Rails configuration
       ↓
Immutable config
       ↓
Client created where needed
```

---

## 2. Database mutation during boot

Bad:

```ruby
initializer "create_default_roles" do
  Role.find_or_create_by!(name: "admin")
end
```

Why?

Every web process can try to do it.

That means:

```text
Deploy
  ↓
Worker 1 → database
Worker 2 → database
Worker 3 → database
Worker 4 → database
```

Instead, use:

* migrations
* seeds
* explicit release tasks

---

## 3. Repeated notification subscription

Bad:

```ruby
config.to_prepare do
  ActiveSupport::Notifications.subscribe(
    "process_action.action_controller"
  ) do |*args|
    # ...
  end
end
```

The callback may execute repeatedly.

Result:

```text
reload 1 → 1 subscriber
reload 2 → 2 subscribers
reload 3 → 3 subscribers
```

Subscribe once or explicitly manage the subscription lifecycle.

---

## 4. Using `after_initialize` for everything

Don't treat:

```ruby
after_initialize
```

as a universal solution.

Use:

```text
Named initializer
    ↓
precise boot ordering

on_load
    ↓
framework extension

to_prepare
    ↓
reload-aware setup

Migration/task/job
    ↓
operational work
```

---

# 32. Interview Questions

## Basic

### What is a Railtie?

A Railtie is Rails' integration mechanism.

It lets a gem or framework register:

* configuration
* initializers
* middleware
* tasks
* generators
* lifecycle hooks

---

### Why do we need Railties?

So independently developed Rails components and gems can participate in the same Rails boot process without putting everything inside `config/application.rb`.

---

### Railtie vs Engine vs Application?

```text
Railtie
  → basic integration

Engine
  → reusable mini application

Application
  → main Rails application
```

---

### When does an initializer run?

During application boot, after Rails has built and ordered the initializer graph.

Not on every request.

---

### What does `config.to_prepare` solve?

It allows setup to run again when Rails prepares/reloads code in development.

The setup must be idempotent.

---

# Intermediate

### How is initializer order decided?

Rails uses:

```text
before:
after:
```

to create dependencies and then topologically sorts the graph using `TSort`.

---

### Why use `ActiveSupport.on_load`?

Because the framework you're extending may not have loaded yet.

It allows your gem to integrate when the framework becomes available.

---

### How does a gem add middleware?

Create a Railtie and modify:

```ruby
app.middleware
```

inside an initializer.

---

### Why can `to_prepare` cause duplicate behavior?

Because it can run multiple times during development.

If you repeatedly:

```text
subscribe
register
prepend
include
add callback
```

without protection, behavior can accumulate.

---

### How do you investigate boot order?

Start with:

```bash
bin/rails initializers
```

Then inspect the relevant Rails source for the exact Rails version.

---

# Senior

### How would you design Rails integration for a plain Ruby gem?

Keep the core gem independent:

```text
Plain Ruby library
       ↓
Optional Railtie
```

The Railtie should provide:

* Rails configuration
* framework hooks
* middleware if needed
* reload-safe behavior
* tasks/generators
* integration tests

---

### A deploy intermittently fails after adding an initializer. What would you investigate?

Look for:

* network calls during boot;
* database calls;
* eager-loading errors;
* missing production credentials;
* incorrect initializer ordering;
* pre-fork connection creation;
* configuration rollout problems;
* races between processes.

---

### Why is this risky?

```ruby
after: "active_record.initialize_database"
```

Because this is an internal initializer name.

Rails can change internal initializer names/order between versions.

If you must depend on it:

```text
pin supported Rails versions
+
integration test
```

---

### How do Railties and Zeitwerk work together?

Railties controls:

```text
Rails lifecycle + integration
```

Zeitwerk controls:

```text
autoloading + eager loading
```

A Railtie must be careful not to permanently capture reloadable classes.

---

# Staff-Level Questions

### 30 engines and 200 initializers — how would you make boot reliable?

Use:

* naming conventions;
* clear ownership;
* minimal boot I/O;
* explicit contracts;
* initializer-order tests;
* middleware-order tests;
* production eager-boot tests;
* boot-time monitoring;
* Rails version compatibility tests.

---

### How would you break apart a huge `config/initializers` folder?

First classify each initializer:

```text
Configuration
Framework integration
Reload setup
Operational work
Business logic
```

Then:

```text
Reusable integration
        ↓
Gem/Engine Railtie

Business logic
        ↓
Service/application code

Database changes
        ↓
Migration

Operational work
        ↓
Task/job/deployment
```

Then replace hidden ordering with explicit dependencies.

---

# 33. Practical Examples

## Example A — Optional Rails integration

Core library:

```ruby
module PriceRules
  class Config
    attr_reader :rounding

    def initialize(rounding: :half_up)
      @rounding = rounding
    end
  end
end

require "price_rules/railtie" if defined?(Rails::Railtie)
```

Railtie:

```ruby
module PriceRules
  class Railtie < Rails::Railtie
    config.price_rules = ActiveSupport::OrderedOptions.new
    config.price_rules.rounding = :half_up

    initializer "price_rules.configure" do |app|
      PriceRules.config =
        PriceRules::Config.new(
          rounding: app.config.price_rules.rounding
        )
    end
  end
end
```

The important idea:

```text
Core gem
    ↓
works without Rails

Railtie
    ↓
translates Rails config into the gem's config
```

---

## Example B — Configuration validation

```ruby
initializer "webhook_verifier.validate_configuration" do |app|
  settings = app.config.webhook_verifier

  next unless Rails.env.production?

  raise "issuer required" if settings.issuer.blank?
  raise "public key required" if settings.public_key_pem.blank?
end
```

This is good because it:

* fails early;
* validates configuration;
* doesn't make a network call.

---

## Example C — Reload-safe decorator

```ruby
Rails.application.config.to_prepare do
  Order.prepend(OrderAuditExtension) unless
    Order < OrderAuditExtension
end
```

Why the check?

Because `to_prepare` can run repeatedly.

---

## Example D — Middleware test

```ruby
stack = Rails.application.middleware.middlewares

assert_operator(
  stack.index(RateLimit::Middleware),
  :<,
  stack.index(Rack::Runtime)
)
```

You're testing the **actual middleware order**, not just that your initializer exists.

---

# 34. Important Edge Cases

### Railtie loaded before Rails

If:

```ruby
defined?(Rails::Railtie)
```

is false, your conditional Railtie require won't run.

Make sure Rails is loaded before requiring the Rails integration.

---

### Duplicate initializer names

Avoid:

```text
configure
configure
configure
```

Use:

```text
my_gem.configure
another_gem.configure
```

---

### Initializer cycle

```text
A after B
B after A
```

Rails cannot sort it.

Remove the unnecessary dependency or redesign the shared setup.

---

### Forking servers

Don't create network clients/database connections too early.

Resources created before `fork` may be unsafe to share between workers.

---

### Multiple Rails applications in one process

Avoid assuming:

```ruby
MyGem.global_config
```

represents every application correctly.

Global module state can become problematic when multiple Rails apps exist in the same process.

---

### API-only Rails applications

Some middleware available in a normal Rails application may not exist in an API-only application.

Don't blindly write:

```ruby
insert_after SomeMiddleware
```

without checking that it exists.

---

### Preloaders

Tools such as Spring can keep processes alive.

So changing Railtie code may require restarting the preloader, not just restarting the Rails server.

---

# 35. Comparison Table

| Concept                    | Main purpose                            |
| -------------------------- | --------------------------------------- |
| `Rails::Railtie`           | Integrate a gem/framework with Rails    |
| `Rails::Engine`            | Reusable mini Rails application         |
| `Rails::Application`       | Main application                        |
| `config/initializers/*.rb` | Application-specific boot configuration |
| `ActiveSupport.on_load`    | Extend a framework when it loads        |
| `config.to_prepare`        | Reload-aware setup                      |
| Middleware                 | Wrap HTTP requests/responses            |
| Concern                    | Share Ruby behavior                     |
| Generator                  | Create/update project files             |
| Rake task                  | Explicit CLI/operational work           |

---

# 36. What to Study Next

After Railties, study these in this order:

```text
1. Rails Initialization
       ↓
2. Zeitwerk
       ↓
3. Rails Engines
       ↓
4. Rack & Middleware
       ↓
5. Rails Executor / Reloader
       ↓
6. Bundler & Ruby require
       ↓
7. Active Record Connection Pool
       ↓
8. PostgreSQL
       ↓
9. Rails Upgrade Strategy
```

This order makes sense because each topic builds on the previous one.

---

# 37. Revision Sheet

Remember these points:

* **Railties = Rails integration + boot orchestration.**
* `Rails::Railtie` is the base integration class.
* `Engine < Railtie`.
* `Application < Engine`.
* A gem must **require its Railtie** for Rails to know about it.
* Initializers run during **boot**, not every request.
* Initializers form a **dependency graph**.
* `before:` and `after:` control ordering.
* Rails uses **TSort** to sort them.
* Use unique initializer names.
* `ActiveSupport.on_load` is useful for extending frameworks safely.
* `config.to_prepare` is for **reload-aware setup**.
* `to_prepare` code must be **idempotent**.
* Railties don't execute SQL.
* Railties configure Active Record; Active Record communicates with PostgreSQL.
* Don't perform unnecessary DB/network work during boot.
* Don't cache reloadable classes globally.
* Middleware order matters.
* `isolate_namespace` is not a security boundary.
* Prefer public hooks over private Rails initializer names.
* Use:

```bash
bin/rails initializers
bin/rails middleware
bin/rails zeitwerk:check
```

when debugging.

---

# 38. One-Page Cheat Sheet

```ruby
# Gem entrypoint
require "my_gem/railtie" if defined?(Rails::Railtie)

module MyGem
  class Railtie < Rails::Railtie

    # Configuration
    config.my_gem = ActiveSupport::OrderedOptions.new
    config.my_gem.enabled = true

    # Initializer
    initializer "my_gem.configure" do |app|
      MyGem.configure(
        enabled: app.config.my_gem.enabled
      )
    end

    # Middleware
    initializer "my_gem.middleware" do |app|
      app.middleware.insert_after Rack::Head,
        MyGem::Middleware
    end

    # Framework extension
    initializer "my_gem.active_record" do
      ActiveSupport.on_load(:active_record) do
        include MyGem::Model
      end
    end

    # Reload-safe setup
    config.to_prepare do
      Widget.include(MyGem::WidgetExtension) unless
        Widget < MyGem::WidgetExtension
    end

    # Tasks
    rake_tasks do
      load File.expand_path("../tasks/my_gem.rake", __dir__)
    end

  end
end
```

Useful commands:

```bash
bin/rails initializers
bin/rails middleware
bin/rails zeitwerk:check

RAILS_ENV=production bin/rails runner 'puts :booted'
```

### Never do this casually

```text
❌ Network calls during boot
❌ Database mutations during boot
❌ Capture reloadable classes globally
❌ Assume middleware exists everywhere
❌ Depend on accidental gem load order
❌ Make every gem require Rails
❌ Put business logic inside Railties
```

### Remember this mental model

```text
                 Rails Application
                        │
                        ↓
                    Railties
          ┌─────────────┼─────────────┐
          ↓             ↓             ↓
     Configuration  Initializers   Middleware
          │             │             │
          └─────────────┼─────────────┘
                        ↓
                  Rails Frameworks
             ┌──────────┼──────────┐
             ↓          ↓          ↓
        ActiveRecord  Controller  ActiveJob
             │
             ↓
         PostgreSQL
```

**In one sentence:**

> **Railties is the system Rails uses to let frameworks, gems, engines, and the application register configuration and boot-time behavior and then combine all of it into one working Rails application.**
