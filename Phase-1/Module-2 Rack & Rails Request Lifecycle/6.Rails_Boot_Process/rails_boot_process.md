# Rails Boot Process — Senior Backend Interview Study Guide

> Scope: modern Rails (Zeitwerk, Rails 7/8 conventions) on MRI Ruby. Exact initializer and server-command details vary by release; the ordering rules and architecture are stable. For a production incident or deploy-sensitive change, check the exact Rails and server versions.

## 1. Overview

### What it is

Think of Rails booting like turning on a complicated machine before you can use it.

1. What is "booting"?

When you run:

rails server

Ruby starts a process, but your Rails application isn't ready yet.

Rails then does a bunch of preparation:

```
Start Ruby
   ↓
Load required gems
   ↓
Load Rails
   ↓
Load your application
   ↓
Configure database, routes, logging, etc.
   ↓
Prepare autoloading
   ↓
Build the web application
   ↓
🚀 Ready to receive requests
```

That whole preparation is Rails booting.

2. Booting ≠ handling a request

This is important.

Suppose you start your server:

rails server

Rails boots once:

```
BOOT
 ↓
Rails is ready
 ↓
Request 1
Request 2
Request 3
Request 4
...
```

It doesn't completely boot Rails for every request.

That's why if booting takes 5 seconds, you don't normally wait 5 seconds for every HTTP request.

3. What are all those complicated things?

Let's translate them into simple language.

Bundler

Your Rails app uses gems like:

```
rails
pg
redis
sidekiq
devise
```
Bundler makes sure Rails uses the correct versions of those gems.

Think:

"Before starting, make sure we have exactly the tools this project expects."

Engines

An engine is basically a mini Rails application/plugin that can add its own code and configuration.

For example, an authentication engine might add:

```
routes
controllers
models
configuration
```

Rails gives these engines a chance to set themselves up during boot.

Zeitwerk

This is Rails' file/code loader.

Imagine you have:

```
app/models/user.rb
```

and inside:

```
class User
end
```

You don't normally need to write:

require "user"

Rails/Zeitwerk figures out:

"They are using User, and User should come from user.rb."

It also handles eager loading, where Rails loads lots of application code upfront, commonly in production.

4. What does Rails configure?

During boot, Rails prepares things like:

```
Database → "How do I connect to PostgreSQL?"
Routes → "Which URL goes to which controller?"
Logging → "Where/how do I write logs?"
Credentials → "How do I access secrets?"
Redis/cache → "How should caching work?"
Active Record → "How do my Ruby models talk to the DB?"
Active Job → "How do background jobs work?"
Action Mailer → "How do I send emails?"
I18n → "How do translations work?"
```

5. The simplest mental model

Think of Rails boot like opening a restaurant:

```
🏢 Open restaurant
      ↓
👨‍🍳 Get staff ready          ← Gems
      ↓
🔧 Turn on equipment          ← Rails components
      ↓
📦 Prepare ingredients        ← Load application code
      ↓
📋 Prepare order system       ← Routes
      ↓
🛢️ Connect to suppliers      ← Database
      ↓
🚪 Open the doors
      ↓
🍔 Serve customers            ← Handle requests
```

Rails boot = preparing everything.

Request handling = actually doing the work.

And the tradeoff is: more preparation makes Rails powerful and flexible, but that preparation takes time and memory.

### Why it exists

Ruby normally loads files directly when your code asks for them. Rails must do a larger set of setup steps in the correct order so the app behaves consistently. That includes:

- Making sure Bundler exposes a fixed set of gems.
- Letting framework parts and engines add configuration safely.
- Assembling Rack middleware into one callable app.
- Letting Zeitwerk know which paths and rules to use for autoloading and eager loading.
- Setting up logging, credentials, cache, DB config, routes, Active Job, I18n, Active Record, and Action Mailer at the right times.

The benefit is convention and extensibility. The cost is slower startup, more memory, some implicit ordering rules, and a phase where not every API is ready.

### When this matters

Understanding boot helps you:

- Diagnose crashes that happen before the first request.
- Decide whether code belongs in config, an initializer, to_prepare, or runtime.
- Optimize cold start time and memory usage.
- Build engines or Railties safely.
- Design preload or warmup behavior for servers that fork.

### Common misconceptions

| Misconception | Reality |
|---|---|
| Rails loads every app file at startup. | Only when eager loading is enabled. In development, Rails usually autoloads files on demand. |
| Every initializer runs after Rails is "ready." | Initializers run during initialization. after_initialize runs later, but it still may run before sockets are bound or workers fork. |
| An initializer is the place for all setup. | Initializers are for process-global boot work. Don't use them for data jobs, long-running remote sync, or per-request tasks. |
| boot.rb boots Rails. | config/boot.rb usually sets BUNDLE_GEMFILE and Bootsnap. Rails framework/app initialization happens later. |
| preload_app! equals eager load. | Preload happens in a parent before fork; eager load loads constants. They are different. |
| Production boot proves all code works. | Eager-loading validates many names and files, but it doesn't test every route, secret, query, or downstream runtime path. |

---

### 1. Rails has 3 different lifecycles

Think of these as three different levels:

**Process lifecycle**

```text
Ruby starts → Rails loads → Ruby process stops
```

**Application lifecycle**

```text
Rails starts → configuration happens → Rails becomes ready
```

**Work lifecycle**

```text
Request comes in → Rails handles it → Request finishes
Request comes in → Rails handles it → Request finishes
...
```

So:

> **Booting Rails = preparing the application.**
> **Handling requests/jobs = using the prepared application.** 

---

### 2. `require` vs Zeitwerk

Ruby has `require`:

```ruby
require "something"
```

It basically means:

> "Load this file/library."

Rails has **Zeitwerk**, which makes loading your application code easier.

For example:

```text
app/models/user.rb
```

contains:

```ruby
class User
end
```

Zeitwerk understands:

> `user.rb` → `User`

So when Rails needs `User`, it can automatically load the file.

**You generally don't need to manually `require` your Rails models/controllers/services.** 

---

### 3. Bundler

Your `Gemfile` says:

```text
Rails
Redis
Sidekiq
Postgres
Devise
...
```

`Gemfile.lock` specifies the exact versions.

**Bundler makes sure your application uses those gems and versions.**

That's basically its job. 

---

### 4. Railties and Engines

These are basically **ways to plug things into Rails**.

A gem can tell Rails:

> "During boot, please add my configuration/middleware/routes/etc."

That's what a **Railtie** helps with.

An **Engine** is basically a more powerful plugin that can behave somewhat like a mini Rails application.

You don't need to memorize this yet. Just remember:

> **Railtie/Engine = something that can add functionality to Rails during boot.** 

---

### 5. Initializers

An initializer is simply:

> **Code that Rails runs while starting up.**

For example:

```ruby
# config/initializers/my_service.rb

MyService.configure(...)
```

Rails runs these during boot.

You can also tell Rails:

> "Run this after X."

or

> "Run this before Y."

Rails figures out the correct order. 

---

### 6. Eager loading vs Autoloading

This is important.

**Autoloading:**

> "I'll load this code when I actually need it."

**Eager loading:**

> "Load everything now during boot."

Example:

```text
Autoloading:
Boot → 2 seconds
Request → needs User → load User

Eager loading:
Boot → load User, Order, Payment, etc.
Request → everything is already loaded
```

Production commonly uses eager loading.

It makes the first request faster and can catch naming problems during startup, but **boot takes longer and uses more memory**. 

---

### 7. Middleware

Think of middleware as **security/checkpoints before your request reaches Rails**.

```text
Browser
   ↓
Middleware
   ↓
Middleware
   ↓
Rails Controller
   ↓
Response
```

For example, middleware can deal with things like sessions, cookies, logging, etc.

Rails builds this middleware chain during boot. 

---

### 8. The whole Rails boot process

This is probably the **most important thing to understand** from the document:

```text
rails server
     ↓
Start Ruby
     ↓
Load Bundler
     ↓
Load Rails
     ↓
Load application configuration
     ↓
Run initializers
     ↓
Set up Zeitwerk
     ↓
Set up database configuration
     ↓
Set up middleware
     ↓
Set up routes
     ↓
Eager load code (if enabled)
     ↓
🚀 Server starts
     ↓
Requests come in
```

That's the big picture. 

---

### 9. One important warning

Try **not to do expensive things during boot**.

For example, this is usually a bad idea:

```ruby
# initializer
User.count
```

Why?

Imagine you deploy **10 Rails processes**.

All 10 might execute that query while starting.

Similarly, don't casually call external APIs during boot:

```ruby
SomeAPI.fetch_data!
```

Now your entire application startup depends on that API being available.

The document calls these **boot dependencies**. 

---

### If you remember only 5 things

```text
1. Rails boot = preparing the application.

2. Bundler = makes the correct gems available.

3. Zeitwerk = automatically loads your Rails code.

4. Initializers = code Rails runs during startup.

5. Eager loading = load lots of application code during startup.
```

And the overall mental model:

**`Ruby starts → Rails prepares everything → Rails becomes ready → requests/jobs happen.`**

That's enough foundation before going deeper into the Rails boot process.


# Rails Boot Process — Advanced Concepts Simplified

## 1. Best Practices

The main goal of Rails boot should be:

> **Fast + predictable + safe + easy to debug**

### Do:

```
- Keep initializers small.
- Keep configuration in `application.rb` and environment files.
- Follow Zeitwerk naming conventions.
- Test eager loading in CI.
- Make reload-related setup idempotent.
- Keep track of boot time, memory, and DB connections.
```

### Avoid during boot:

```
❌ Database queries
❌ Database writes
❌ API/network calls
❌ Long-running work
❌ Starting background threads
❌ Unnecessary side effects
```

Why?

Because **every Rails process has to do this work before it becomes ready.**

---

# 2. Bad Initializer

Avoid putting everything into an initializer:

```ruby
# config/initializers/setup_everything.rb

User.find_each { |u| SearchIndex.sync(u) }

Faraday.get(ENV.fetch("CONFIG_URL"))

Thread.new { loop { Metrics.flush } }
```

This means:

```text
Rails starts
   ↓
Query database
   ↓
Call external API
   ↓
Start thread
   ↓
Finally finish booting
```

That's bad.

Instead:

```text
Database work → migration/job
API work      → lazy/background work
Long work     → background job
Threads       → explicit lifecycle management
```

---

# 3. Don't Put Work at the Top of a File

Avoid:

```ruby
# app/services/rates.rb

Rates.refresh!

class Rates
end
```

Why?

That code runs whenever Rails loads the file.

It could happen during:

```text
Boot
Eager loading
Reloading
Normal autoloading
```

Instead, keep files mostly for **definitions**:

```ruby
class Rates
  def refresh
    # actual work
  end
end
```

> Defining code is fine. Automatically doing expensive work while loading the file is dangerous.

---

# 4. Don't Depend on Filename Order

Don't do this:

```text
01_config.rb
02_database.rb
99_patch.rb
```

and assume Rails will always execute things in that order.

Instead, explicitly tell Rails:

```text
Run this AFTER X
Run this BEFORE Y
```

Rails can then understand the dependency.

---

# 5. Global Objects Can Cause Problems

For example:

```ruby
CLIENT = Vendor::Client.new(...)
```

This creates one global client during boot.

That can become problematic with:

* Reloading
* Testing
* Forking
* Rotating credentials
* Resetting connections

Prefer creating clients in a controlled way, for example lazily:

```ruby
def self.client
  @client ||= Vendor::Client.new(...)
end
```

---

# 6. Don't Hide Boot Errors

Avoid:

```ruby
begin
  CriticalService.configure!
rescue
  Rails.logger.error(...)
end
```

If the service is **required for the application to work**, let Rails fail.

Better:

```text
Critical service unavailable
        ↓
Rails doesn't start
        ↓
Clear error
        ↓
Fix the problem
```

A half-working application can be worse than an application that clearly refuses to start.

---

# 7. Important Interview Questions

## What happens after `bin/rails server`?

Simplified:

```text
bin/rails
   ↓
config/boot.rb
   ↓
Bundler / Bootsnap
   ↓
Rails command
   ↓
Application + environment
   ↓
Rails initializes
   ↓
Middleware + routes + autoloaders
   ↓
Eager loading
   ↓
Server starts
```

---

## What is `boot.rb`?

Usually responsible for things like:

```text
Bundler
Bootsnap
```

It happens very early.

---

## What is `environment.rb`?

It loads the Rails application and initializes it.

Think:

```text
boot.rb
→ prepare Ruby/gems

environment.rb
→ prepare Rails application
```

---

## What is an initializer?

A piece of code that Rails runs during startup.

```ruby
initializer "my_service.setup" do
  # setup
end
```

---

## Why eager-load in production?

Main reasons:

```text
✓ Find code/naming errors early
✓ Avoid loading code on the first request
✓ Can improve memory sharing with forked workers
```

---

# 8. Does Rails Always Connect to PostgreSQL During Boot?

**No.**

During boot Rails mainly prepares the database configuration and connection pool.

Actual DB connections are often created **when they are needed**.

For example:

```ruby
User.count
```

may cause a real DB connection to be checked out.

So:

```text
Boot
 ↓
Prepare DB configuration
 ↓
Server ready

Later...
 ↓
Request needs DB
 ↓
Get DB connection
```

---

# 9. What is Bootsnap?

Bootsnap is basically a **startup speed improvement**.

It caches expensive things like:

```text
require lookup
Ruby compilation
```

So Rails can start faster.

Important:

> Bootsnap changes **speed**, not the Rails boot order.

---

# 10. `preload_app!` vs Eager Loading

These are different concepts.

### Eager loading

```text
Load application classes
```

### `preload_app!`

```text
Load application
     ↓
Parent process
     ↓
Fork workers
```

Example:

```text
Parent
  ↓
Loads Rails
  ↓
Fork
 ↙   ↘
Worker Worker
```

Workers can share some memory with the parent.

But things like:

```text
DB connections
Sockets
Threads
External clients
```

may need to be reset after the fork.

---

# 11. Development Reloading

In development, Rails can reload your application code.

Example:

```text
You change user.rb
       ↓
Rails unloads old User
       ↓
Rails loads new User
```

This creates an important problem.

If some global object saved the **old User class**:

```ruby
Registry.user_class = User
```

Rails may later replace `User` with a new class.

Now:

```text
Registry → old User
Rails    → new User
```

That's why reload-aware setup should often use:

```ruby
Rails.application.reloader.to_prepare do
  # setup
end
```

And it should be **idempotent**.

Meaning:

> Running it twice should not break anything or create duplicates.

---

# 12. The Most Important Architecture

Think about Rails boot like this:

```text
Ruby Process
     ↓
Bundler
     ↓
Rails
     ↓
Application Configuration
     ↓
Initializers
     ↓
Zeitwerk
     ↓
Middleware
     ↓
Routes
     ↓
Eager Loading
     ↓
Server / Worker / Console
     ↓
Requests / Jobs
```

---

# 13. Where Should Code Go?

A simple rule:

| What you're doing           | Where it belongs                  |
| --------------------------- | --------------------------------- |
| Global configuration        | `application.rb`                  |
| Environment-specific config | `config/environments/*.rb`        |
| Rails integration setup     | Initializer / Railtie             |
| Reload-aware setup          | `to_prepare`                      |
| HTTP request logic          | Middleware / Controller / Service |
| Database schema changes     | Migration                         |
| Heavy/background work       | Background Job                    |
| One-time deployment work    | Deploy task                       |

---

# 14. Things You Generally DON'T Want During Boot

```text
❌ DB queries
❌ DB writes
❌ API calls
❌ Long-running calculations
❌ Starting threads
❌ Unbounded retries
❌ Manually requiring Zeitwerk-managed app files
❌ Depending on eager-load order
```

---

# 15. Useful Commands

### Check whether Rails can boot

```bash
bin/rails runner 'puts :ok'
```

### Check Zeitwerk

```bash
bin/rails zeitwerk:check
```

### See Rails information

```bash
bin/rails about
```

### See middleware

```bash
bin/rails middleware
```

### Check production boot

```bash
RAILS_ENV=production bin/rails runner 'puts :ok'
```

---

# 16. Important Edge Cases

### `runner` works but server fails

Possible reason:

```text
Runner
 ↓
Rails boots
 ↓
Code runs
```

But server additionally has:

```text
Socket
Preload
Fork
Server configuration
```

So always test the **real server setup** when debugging server-specific issues.

---

### Application works in development but fails in production

Often related to:

```text
Eager loading
Zeitwerk naming
Environment configuration
```

Run:

```bash
bin/rails zeitwerk:check
```

---

### Worker starts before migrations finish

You can have:

```text
New application code
       ↓
Old database schema
       ↓
💥 Error
```

That's why deployments and migrations need to be coordinated.

---

# 17. The Cheat Sheet

```text
ENTRYPOINT

bin/rails
   ↓
config/boot.rb
   ↓
Rails command
   ↓
environment.rb
   ↓
application.rb + environment config
   ↓
Rails.application.initialize!
   ↓
middleware + routes + autoloaders
   ↓
eager loading?
   ↓
runtime
```

## Put code here

```text
Global config
→ config/application.rb

Environment config
→ config/environments/production.rb

Rails integration
→ initializer / Railtie

Reload-aware setup
→ to_prepare

Request logic
→ middleware/controller/service

Database schema/data
→ migration/job/deploy task
```

## Don't put this in boot by default

```text
Remote API calls
DB queries/writes
Long work
Threads
Unbounded retries
Manual require of app files
```

---

# 18. Interview Sound Bite

If someone asks:

> **"Explain the Rails boot process."**

A good short answer is:

> Rails boot is the process of preparing a Ruby process to run a Rails application. Bundler makes the required gems available, Railties and engines contribute configuration and initializers, Rails sets up autoloading, middleware and routes, and production usually eager-loads application code before serving traffic. Boot should be deterministic and lightweight, so expensive DB/network work should generally be avoided.

That's the level of understanding you should aim for first.

```

The biggest takeaway from this section is really just:

**Rails boot should prepare things, not perform the application's actual work.** :contentReference[oaicite:0]{index=0}
```
