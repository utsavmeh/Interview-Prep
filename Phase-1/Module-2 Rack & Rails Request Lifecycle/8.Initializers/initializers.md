# Mastering Rails Initializers — Beginner-Friendly Senior Engineer Study Guide

---

# 1. What is a Rails Initializer?

### What it is

A Rails initializer is simply a **Ruby file inside**:

```text
config/initializers/
```

For example:

```text
config/initializers/devise.rb
config/initializers/sidekiq.rb
config/initializers/redis.rb
```

Rails runs these files **while the application is starting up**.

The important thing to remember is:

> **An initializer is code that Rails runs during application startup to configure something.**

For example, suppose you use Stripe.

Stripe might need an API key:

```ruby
Stripe.api_key = ENV["STRIPE_API_KEY"]
```

You could put that configuration in:

```text
config/initializers/stripe.rb
```

Then, when Rails starts, it configures Stripe.

---

### Why do initializers exist?

Imagine you have many different libraries in your Rails application:

* Devise
* Sidekiq
* Redis
* Stripe
* Elasticsearch
* Sentry
* AWS
* Active Storage
* custom gems

Each of these may need some configuration.

Instead of putting all of that configuration inside `application.rb`, Rails gives you:

```text
config/initializers/
```

So you can organize things like:

```text
config/initializers/
├── devise.rb
├── redis.rb
├── sidekiq.rb
├── stripe.rb
└── cors.rb
```

This makes it much easier to find configuration.

---

### When should you use an initializer?

A good mental model is:

> **Use an initializer for something that needs to be configured once when Rails starts.**

For example:

```ruby
Stripe.api_key = ENV["STRIPE_API_KEY"]
```

or:

```ruby
Mime::Type.register "application/pdf", :pdf
```

or:

```ruby
Sidekiq.configure_server do |config|
  # configuration
end
```

These things don't need to happen every time a request comes in.

They need to happen when the application starts.

That's why an initializer is appropriate.

---

# 2. Important: Boot Process vs Initializers

This distinction is extremely important.

A lot of developers initially think:

> "Rails boot process = initializers."

That's not correct.

### Boot process

The **boot process is the entire process of starting Rails** and preparing it to do work.

It includes things like:

```text
Ruby starts
   ↓
Bundler loads gems
   ↓
Rails loads
   ↓
application.rb loads
   ↓
environment configuration loads
   ↓
Rails runs its initialization steps
   ↓
Rails loads your initializers
   ↓
Rails finishes initialization
   ↓
Application is ready
```

So:

> **Boot process = the entire startup journey.**

---

### Initializers

Initializers are **one part of that boot process**.

Think of it like starting your computer.

**Boot process:**

```text
Turn on computer
      ↓
BIOS/firmware starts
      ↓
Operating system loads
      ↓
Drivers load
      ↓
System services start
      ↓
Login screen appears
```

There are many things happening.

Now imagine:

```text
Initializers = one particular stage where certain configuration tasks happen
```

The same idea applies to Rails.

### Simple comparison

| Boot Process                                              | Initializers                                 |
| --------------------------------------------------------- | -------------------------------------------- |
| Entire Rails startup process                              | One stage inside startup                     |
| Includes Bundler, Rails, config, gems, initializers, etc. | Mainly your `config/initializers/*.rb` files |
| Starts before initializers                                | Happens during boot                          |
| Makes Rails ready to run                                  | Configures libraries/application behavior    |
| Much bigger concept                                       | Smaller concept                              |

So you can remember:

> **Boot process is the whole startup. Initializers are configuration scripts that Rails executes during that startup.**

---

# 3. Core Concepts

## The Boot Phase

Initializers are part of the Rails initialization process.

Before Rails can execute:

```ruby
config/initializers/devise.rb
```

Rails needs to have loaded the things that Devise depends on.

For example:

```ruby
Devise.setup do |config|
  ...
end
```

wouldn't work if Rails hadn't loaded Devise yet.

So Rails first loads the application configuration and dependencies, and then reaches the stage where it loads your initializers.

A simplified picture is:

```text
Rails starts
    ↓
Load Bundler / gems
    ↓
Load application configuration
    ↓
Load environment configuration
    ↓
Run Rails initialization steps
    ↓
Run config/initializers/*.rb
    ↓
Finish initialization
```

---

# 4. Initializer Load Order

Rails loads application initializer files in a predictable order.

Conceptually, Rails does something similar to:

```ruby
Dir["#{config.root}/config/initializers/**/*.rb"].sort.each do |file|
  load(file)
end
```

The important part here is:

```ruby
.sort
```

That means files are sorted before being loaded.

For example:

```text
config/initializers/
├── a.rb
├── b.rb
└── c.rb
```

will normally execute as:

```text
a.rb
b.rb
c.rb
```

And Rails also finds files inside subdirectories.

For example:

```text
config/initializers/
├── payments/
│   ├── stripe.rb
│   └── paypal.rb
└── redis.rb
```

---

## But should you depend on alphabetical order?

Technically, Rails sorts the initializer files.

But you **shouldn't design your application around filenames like this**:

```text
01_setup.rb
02_setup_something_else.rb
03_setup_another_thing.rb
```

and assume:

```text
01 → 02 → 03
```

is the correct dependency mechanism.

Why?

Because now your application's correctness depends on filenames.

Someone could rename:

```text
01_setup.rb
```

to:

```text
payment_setup.rb
```

and suddenly break the dependency.

Instead, if something genuinely needs to happen after another initialization step, use Rails' initialization mechanisms such as:

```ruby
after_initialize
```

or Rails' initializer dependency/order mechanisms.

The principle is:

> **Don't use filenames as your application's dependency-management system.**

---

# 5. Do Initializers Reload in Development?

No.

This is one of the most important things to remember.

Suppose you have:

```text
config/initializers/stripe.rb
```

and change:

```ruby
Stripe.api_key = ENV["STRIPE_API_KEY"]
```

to something else.

Rails doesn't normally reload that initializer just because you changed the file.

You need to restart the Rails process.

For example, if Puma is running:

```text
Ctrl+C
```

and then:

```bash
bin/rails server
```

again.

Why?

Because initializers are part of the **boot process**.

The initializer runs when the process boots.

It isn't request-time application code.

---

# 6. Should You Define Application Constants in Initializers?

Generally, no.

For example, don't do this:

```ruby
# config/initializers/settings.rb

APP_NAME = "My Application"
```

This creates a global constant.

Global constants can become difficult to manage as your application grows.

For custom application configuration, Rails gives you:

```ruby
Rails.application.config.x
```

For example:

```ruby
Rails.application.config.x.app_name = "My Application"
```

Then you can access it with:

```ruby
Rails.configuration.x.app_name
```

This keeps application configuration organized under Rails' configuration system.

---

# 7. Internal Working — What Actually Happens?

Now let's go deeper.

Understanding this is useful when you're working at senior level because it helps explain problems like:

* Why is Rails taking 30 seconds to start?
* Why does an initializer fail during `assets:precompile`?
* Why does a model constant cause a Zeitwerk error?
* Why does changing an initializer require a restart?
* Why does a gem initializer run before yours?

Let's walk through the process.

---

## Step 1: Entry Point

You might start Rails with:

```bash
bin/rails server
```

or potentially:

```bash
bundle exec puma
```

This starts the Ruby/Rails process.

---

# Step 2: `config/boot.rb`

Rails first uses:

```text
config/boot.rb
```

One of its important responsibilities is setting up Bundler and the application's dependency environment.

In simple terms:

> **`boot.rb` helps Rails get the gems/dependencies ready.**

For example:

```text
Rails
ActiveRecord
Devise
Redis
Sidekiq
etc.
```

---

# Step 3: `config/application.rb`

Next Rails loads:

```text
config/application.rb
```

This is where the main Rails application configuration lives.

You often see something like:

```ruby
require_relative "boot"

require "rails/all"

Bundler.require(*Rails.groups)

module MyApp
  class Application < Rails::Application
    # configuration
  end
end
```

At this stage Rails creates your application object.

Conceptually:

```text
Rails
  ↓
Your Application
```

---

# Step 4: Environment Configuration

Rails then loads the configuration for the current environment.

For example, if you're running development:

```text
config/environments/development.rb
```

If production:

```text
config/environments/production.rb
```

These files contain environment-specific settings.

For example:

```ruby
config.cache_classes = false
```

or other development/production behavior.

---

# Step 5: Rails Starts Its Initialization Process

Eventually Rails starts the main initialization process.

You can think of this as:

```ruby
Rails.application.initialize!
```

This is where Rails executes a sequence of initialization steps.

These initialization steps aren't all coming from your application.

Rails itself, Rails frameworks, and gems can register initialization steps.

For example:

```text
Rails
  ↓
ActiveRecord initialization
  ↓
ActionController initialization
  ↓
Gem initialization
  ↓
Your application initialization
  ↓
etc.
```

---

# Step 6: Rails Reaches `load_config_initializers`

During this process Rails reaches an internal initialization step responsible for loading:

```text
config/initializers/
```

Conceptually, it looks roughly like:

```ruby
initializer :load_config_initializers do
  config.paths["config/initializers"].existent.sort.each do |initializer|
    load_config_initializer(initializer)
  end
end
```

And that eventually loads your files.

So if you have:

```text
config/initializers/
├── devise.rb
├── redis.rb
└── stripe.rb
```

Rails executes them.

---

# Step 7: Your Initializers Execute

For example:

```ruby
# config/initializers/stripe.rb

Stripe.api_key = ENV["STRIPE_API_KEY"]
```

Rails executes that Ruby code.

Then:

```ruby
# config/initializers/devise.rb

Devise.setup do |config|
  config.mailer_sender = "please-reply@example.com"
end
```

Rails executes that too.

This is the actual moment when your initializer code runs.

---

# Step 8: `after_initialize`

After Rails has finished running the initialization process, it can execute:

```ruby
config.after_initialize do
  # code
end
```

This is useful when you need to wait until Rails has finished all its initializers.

For example:

```ruby
Rails.application.config.after_initialize do
  puts "Rails has finished initializing"
end
```

The important difference is:

```text
initializer
    ↓
runs during initialization

after_initialize
    ↓
runs after initialization has finished
```

---

# Step 9: Server Becomes Ready

Once the application has finished booting, the server can start accepting requests.

Conceptually:

```text
Rails boot
    ↓
Rails initialization
    ↓
Initializers
    ↓
after_initialize
    ↓
Application ready
    ↓
Puma/Rack handles requests
```

---

# 8. Architecture — Where Do Initializers Fit?

Think of your Rails application as several layers.

A simplified architecture might look like:

```text
             Your Application
                   │
          ┌────────┴────────┐
          │                 │
      Domain Logic      Controllers
          │                 │
          └────────┬────────┘
                   │
             Rails Framework
                   │
          ┌────────┴────────┐
          │                 │
         Gems          Infrastructure
```

Initializers mostly sit around the **configuration/infrastructure boundary**.

They connect things like:

```text
Rails
 ↓
Gem
 ↓
Your configuration
```

For example:

```ruby
# config/initializers/sidekiq.rb

Sidekiq.configure_server do |config|
  # configuration
end
```

The initializer is basically saying:

> "When my Rails application starts, configure Sidekiq like this."

---

# 9. Why Not Put Everything in `application.rb`?

You technically could put lots of configuration there.

But eventually you'd get something like:

```ruby
class Application < Rails::Application

  # Devise configuration

  # Redis configuration

  # Sidekiq configuration

  # Stripe configuration

  # CORS configuration

  # MIME configuration

  # Elasticsearch configuration

  # etc.
end
```

That becomes difficult to understand.

Instead:

```text
config/initializers/
├── devise.rb
├── redis.rb
├── sidekiq.rb
├── stripe.rb
└── cors.rb
```

Now each file has a clear responsibility.

This is an example of **separation of concerns**.

---

# 10. Real Production Example — Redis

Suppose your application uses Redis.

You could configure it in:

```text
config/initializers/redis.rb
```

```ruby
require "connection_pool"

pool_size = ENV.fetch("RAILS_MAX_THREADS", 5).to_i
timeout = ENV.fetch("REDIS_TIMEOUT", 5).to_i

Redis::Objects.redis = ConnectionPool.new(
  size: pool_size,
  timeout: timeout
) do
  Redis.new(
    url: ENV.fetch("REDIS_URL"),
    ssl_params: {
      verify_mode: OpenSSL::SSL::VERIFY_NONE
    }
  )
end
```

Let's understand what this is doing.

### First:

```ruby
require "connection_pool"
```

Load the library that provides connection pooling.

### Then:

```ruby
pool_size = ENV.fetch("RAILS_MAX_THREADS", 5).to_i
```

Get the number of Rails threads from the environment.

If it doesn't exist:

```text
5
```

is used.

### Then:

```ruby
ConnectionPool.new(...)
```

creates a pool of Redis connections.

Why?

Because your application can have multiple threads.

Instead of creating a completely new Redis connection every time, the application can reuse connections from the pool.

---

# 11. Devise Initializer

Devise is another common example.

You might have:

```text
config/initializers/devise.rb
```

with:

```ruby
Devise.setup do |config|
  config.mailer_sender = "please-reply@my-saas.com"
  config.password_length = 12..128

  config.warden do |manager|
    manager.default_strategies(
      scope: :user
    ).unshift :two_factor_authenticatable
  end
end
```

This means:

> "When Rails starts, configure Devise with these settings."

For example:

```ruby
config.password_length = 12..128
```

means passwords must be between 12 and 128 characters.

And:

```ruby
config.mailer_sender = ...
```

sets the sender address used by Devise emails.

---

# 12. MIME Types

You can also register custom MIME types.

For example:

```ruby
# config/initializers/mime_types.rb

Mime::Type.register "application/vnd.api+json", :json_api
Mime::Type.register "application/pdf", :pdf
```

This tells Rails:

> "These MIME types exist, and I want to refer to them using these symbols."

Again, this is configuration that needs to happen when the application starts.

If you change this initializer:

> **Restart Rails.**

---

# 13. Common Mistakes

Now let's look at mistakes developers commonly make.

---

## Junior Mistake #1: Putting Business Logic in Initializers

Don't do this:

```ruby
User.create(...)
```

or:

```ruby
Order.process_pending_orders
```

An initializer should generally configure things.

It shouldn't perform normal application business operations.

Think:

```text
Initializer → configuration
Application code → business logic
```

---

## Junior Mistake #2: Forgetting to Restart

You modify:

```text
config/initializers/foo.rb
```

and then wonder:

> "Why isn't my change working?"

Because the initializer ran when Rails booted.

Restart the process.

---

# 14. Mid-Level Mistake — The Zeitwerk Trap

This is an important Rails concept.

Suppose you write:

```ruby
# config/initializers/order.rb

Order.include(MyStateMachine)
```

It looks harmless.

But `Order` is an application class.

Rails uses **Zeitwerk** to automatically load application classes.

During boot, you have to be careful about when those application constants are referenced.

If you reference:

```ruby
Order
```

too early, you can interfere with the autoloading process.

This can result in problems such as:

```text
NameError
```

or other autoloading/reloading problems.

The safer approach is to use:

```ruby
Rails.application.config.to_prepare do
  Order.include(MyStateMachine)
end
```

We'll discuss `to_prepare` more later.

The important idea is:

> **Don't eagerly reference reloadable application classes from a normal initializer unless you know exactly why it's safe.**

---

# 15. Database Queries in Initializers

This is another big mistake.

Don't do:

```ruby
# config/initializers/setup.rb

User.admin.first
```

Why is this dangerous?

Because **many Rails commands boot the Rails application**.

Not just:

```bash
bin/rails server
```

For example:

```bash
bin/rails db:migrate
```

```bash
bin/rails assets:precompile
```

```bash
bin/rails console
```

and various other tasks may boot Rails.

Imagine your initializer does:

```ruby
User.admin.first
```

but the database hasn't been created yet.

Then:

```text
Rails boots
   ↓
Initializer runs
   ↓
User.admin.first
   ↓
Database connection/query
   ↓
Database doesn't exist
   ↓
Rails boot fails
```

Now even a command that doesn't actually need the database may fail.

That's why:

> **Avoid database queries in initializers.**

---

# 16. Senior Mistake — Network Calls During Boot

This is particularly important in production.

Imagine:

```ruby
# config/initializers/config.rb

response = Net::HTTP.get(...)
```

You're making an HTTP request while Rails is starting.

Now imagine that external service takes:

```text
10 seconds
```

Your Rails application takes at least 10 extra seconds to boot.

Now imagine the service is down.

Your application may never finish booting.

You get:

```text
Rails starts
   ↓
Initializer
   ↓
HTTP request
   ↓
External service unavailable
   ↓
Initializer fails
   ↓
Rails fails to boot
```

This can be particularly painful in Kubernetes.

You deploy a new container.

The container starts.

Rails waits for an external service.

The service is unavailable.

Rails never becomes ready.

Kubernetes restarts it.

And you can end up in a restart loop.

So:

> **Avoid blocking network calls in initializers unless the dependency is absolutely required and the failure behavior is deliberately designed.**

---

# 17. Memory Considerations

Initializers also affect memory.

Suppose you do:

```ruby
huge_data = File.read("50MB-file.json")
```

and parse it into a giant Ruby Hash.

Now your Rails process has a large object in memory.

If your server uses multiple processes/workers, this can affect memory usage.

With fork-based servers, some memory can initially be shared using **Copy-on-Write (CoW)**, which can make preloading useful.

But the important beginner-level idea is:

> **Anything you load globally during boot can increase your application's baseline memory usage.**

So don't put huge datasets into global memory just because you can.

---

# 18. Lazy Loading vs Loading During Boot

Suppose you have some expensive configuration that's only required by one background job.

You could load it during Rails boot:

```ruby
# initializer
ExpensiveThing.load!
```

But then every Rails process loads it.

Even if the application never uses it.

Instead, consider loading it only when it's actually needed.

That's called **lazy loading**.

Simple idea:

```text
Eager:
Rails starts → load everything

Lazy:
Rails starts → load only when needed
```

This can reduce boot time and memory usage.

---

# 19. Security Considerations

## Never Hardcode Secrets

Bad:

```ruby
Stripe.api_key = "sk_live_12345"
```

Why?

Because that secret can end up in:

* Git history
* GitHub
* backups
* logs
* other developers' machines

Instead, use environment variables or Rails credentials.

For example:

```ruby
Stripe.api_key =
  ENV["STRIPE_API_KEY"] ||
  Rails.application.credentials.stripe[:secret_key]
```

Now the actual secret isn't sitting inside your source code.

---

# 20. Global State Mutation

Be careful when modifying global Ruby state.

For example, monkey-patching core classes:

```ruby
String.class_eval do
  # modifications
end
```

or modifying globally shared framework behavior.

Because your application is multi-threaded and potentially multi-process, global modifications can create unexpected behavior.

You should have a very good reason before changing global behavior.

---

# 21. SSRF During Boot

Suppose your initializer does this:

```ruby
url = ENV["CONFIG_URL"]
Net::HTTP.get(URI(url))
```

That can be dangerous if an attacker can influence:

```text
CONFIG_URL
```

They may be able to make your server request an internal resource.

That's related to **SSRF — Server-Side Request Forgery**.

So if an initializer fetches a remote resource based on configuration:

> Validate the URL and don't blindly request arbitrary addresses.

---

# 22. Debugging Initializers

Suppose you run:

```bash
bin/rails server
```

and it seems to hang.

One possibility is an initializer.

For example:

```ruby
# config/initializers/foo.rb

sleep 30
```

Rails will appear to hang for 30 seconds.

Or:

```ruby
Net::HTTP.get(...)
```

might be waiting for a slow external service.

---

## Simple debugging

You can temporarily add:

```ruby
puts "Loading Redis initializer"
```

Then:

```ruby
puts "Redis initializer finished"
```

For example:

```ruby
puts "Loading Redis initializer"

# expensive operation

puts "Redis initializer finished"
```

Now you can see where startup is getting stuck.

---

# 23. Debugging Load Order

Suppose:

```text
initializer A
```

needs to happen before:

```text
initializer B
```

You can inspect the initializer paths.

Conceptually:

```ruby
Rails.application.config.paths["config/initializers"].existent.sort
```

This shows the initializer files Rails knows about.

You can also temporarily add:

```ruby
puts "Loading A"
```

and:

```ruby
puts "Loading B"
```

to confirm what's happening.

But again:

> Don't make alphabetical filenames your primary dependency mechanism.

---

# 24. Fixing `NameError` With `to_prepare`

Suppose this causes:

```text
uninitialized constant User
```

inside an initializer:

```ruby
User.include(MyModule)
```

Instead, you can use:

```ruby
Rails.application.config.to_prepare do
  User.include(MyModule)
end
```

Why is this better?

Because `to_prepare` is designed for code that needs to run when Rails' application code is ready.

Another important benefit:

> In development, `to_prepare` can run again when application code is reloaded.

So unlike a normal initializer:

```text
initializer
    ↓
normally runs once during boot
```

`to_prepare` is designed around Rails' reload lifecycle.

---

# 25. Best Practices

Let's go through the recommended practices one by one.

---

## 1. Make Initializers Idempotent

"Idempotent" sounds complicated, but the basic idea is:

> Running the setup more than once shouldn't cause a disaster.

For example, avoid code that keeps adding the same configuration repeatedly.

Even though normal initializers run once per boot, writing setup code in an idempotent way is a useful engineering habit.

---

# 26. One Responsibility Per Initializer

Prefer:

```text
config/initializers/
├── redis.rb
├── stripe.rb
├── sidekiq.rb
├── devise.rb
└── cors.rb
```

rather than:

```text
config/initializers/setup_everything.rb
```

The second file eventually becomes a huge dumping ground.

If you have:

```text
Stripe configuration
Redis configuration
Sidekiq configuration
Devise configuration
custom framework patches
```

all in one file, it becomes difficult to maintain.

---

# 27. `ActiveSupport.on_load`

This is useful when you want to customize Rails framework components.

Suppose you want to add your custom module to controllers.

A naive approach:

```ruby
ActionController::Base.send(
  :include,
  MyCustomModule
)
```

This immediately references:

```ruby
ActionController::Base
```

which means Rails has to load that framework component immediately.

Instead:

```ruby
ActiveSupport.on_load(:action_controller) do
  include MyCustomModule
end
```

Now you're basically saying:

> "When Action Controller is loaded, apply this customization."

This can avoid unnecessarily forcing framework components to load early.

---

# 28. `Rails.configuration.x`

For custom application settings, prefer:

```ruby
Rails.application.config.x
```

For example:

```ruby
Rails.application.config.x.payment_timeout = 5
```

Then:

```ruby
Rails.configuration.x.payment_timeout
```

This is cleaner than creating global constants like:

```ruby
PAYMENT_TIMEOUT = 5
```

---

# 29. Fail Fast

Suppose your application absolutely requires:

```text
API_KEY
```

Don't allow the application to start and then discover hours later that the key is missing.

Use:

```ruby
config.api_key = ENV.fetch("API_KEY")
```

`ENV.fetch` raises an error if the variable doesn't exist.

So:

```text
Rails starts
   ↓
Initializer
   ↓
API_KEY missing
   ↓
Boot fails immediately
```

That's actually useful.

You'd rather discover the problem immediately than have your production application running but unable to perform critical operations.

---

# 30. Anti-Patterns

## Environment-specific filenames

Don't do something like:

```text
config/initializers/production_setup.rb
```

and then:

```ruby
if Rails.env.production?
  # everything
end
```

If the configuration is specifically an environment configuration, put it in:

```text
config/environments/production.rb
```

Use initializers when you're configuring a gem or dependency across environments.

---

# 31. Database Seed Logic

Don't use initializers for:

```ruby
User.create(...)
```

or:

```ruby
Product.create(...)
```

That's not configuration.

That's data/business logic.

Use things such as:

```text
db/seeds.rb
```

or dedicated scripts/tasks.

---

# 32. Heavy Computation

Don't do expensive data processing during boot.

For example:

```ruby
huge_xml = File.read(...)
parse_huge_xml(huge_xml)
```

If it takes several seconds, every Rails process pays that cost during startup.

Move expensive work somewhere more appropriate.

---

# 33. Interview Questions

## Basic Question

**What is an initializer?**

Answer:

An initializer is a Ruby file inside:

```text
config/initializers/
```

that Rails executes during application boot.

It's mainly used to configure gems, libraries, and application-wide settings.

---

### Do I need to restart the server after changing one?

Yes.

Because initializers normally execute during boot.

So after changing:

```text
config/initializers/cors.rb
```

restart the Rails process.

---

# 34. Intermediate Question

**I put this in an initializer:**

```ruby
User.find_by(email: "admin@app.com")
```

and my deployment pipeline fails during:

```text
assets:precompile
```

Why?

Because `assets:precompile` can boot the Rails application.

That means your initializer runs.

Then:

```ruby
User.find_by(...)
```

tries to access the database.

But during asset compilation, the database might:

* not exist
* not be reachable
* not have migrations applied

So the entire Rails boot fails.

The important lesson:

> **An initializer runs when many Rails commands boot the application, not just when you start the web server.**

---

# 35. Senior Question

Suppose you need to modify an application model during initialization.

For example:

```ruby
Order.include(MyStateMachine)
```

How can you do it safely with Zeitwerk?

Use:

```ruby
Rails.application.config.to_prepare do
  Order.include(MyStateMachine)
end
```

Why?

Because `Order` is application code managed by Zeitwerk.

`to_prepare` gives Rails an appropriate place to execute code involving reloadable application classes.

It also works with development reloading.

---

# 36. Staff-Level Question

Your Rails application takes:

```text
45 seconds
```

to boot.

That's terrible for autoscaling.

How would you investigate it?

### Step 1 — Find slow initialization work

Measure how long individual initializers take.

You can write tooling around initializer loading and use something like:

```ruby
Benchmark.measure
```

to measure execution time.

---

### Step 2 — Find the expensive initializer

For example:

```text
redis.rb       → 20ms
devise.rb      → 50ms
stripe.rb      → 100ms
some_config.rb → 30 seconds
```

Now you've found the problem.

---

### Step 3 — Look for unnecessary eager loading

Maybe an initializer is doing:

```ruby
require "huge_library"
```

even though that library is only used by one part of the application.

Consider loading it later.

---

### Step 4 — Look for network calls

Search for things like:

```ruby
Net::HTTP
HTTParty
Faraday
RestClient
```

inside initializers.

A network request can dramatically increase boot time.

---

### Step 5 — Use lazy loading where possible

If something doesn't need to exist during boot, don't initialize it during boot.

Instead:

```text
load it when it's actually needed
```

---

### Step 6 — Use Rails hooks appropriately

For framework-specific customization, consider:

```ruby
ActiveSupport.on_load
```

instead of forcing framework components to load early.

---

# 37. Practical Example — Feature Flags

Suppose you have:

```text
config/features.yml
```

containing:

```yaml
new_checkout_flow:
  enabled: true
```

You want to load this configuration during boot.

You could use:

```ruby
# config/initializers/feature_flags.rb

Rails.application.config.x.features = begin
  config_file = Rails.root.join("config", "features.yml")

  if File.exist?(config_file)
    YAML.load_file(config_file).deep_symbolize_keys
  else
    {}
  end
end
```

Now you haven't created a global constant.

Instead, you've stored the configuration under:

```ruby
Rails.configuration.x.features
```

You can access it:

```ruby
Rails.configuration.x.features.dig(
  :new_checkout_flow,
  :enabled
)
```

So the result might be:

```ruby
true
```

---

# 38. Executing Code After All Initializers

Sometimes you don't just want to run during initialization.

You want to wait until **everything has finished initializing**.

For example:

```ruby
Rails.application.config.after_initialize do
  unless Rails.application.config.active_record.schema_format == :sql
    Rails.logger.warn(
      "We recommend using SQL schema format!"
    )
  end
end
```

The important thing here is:

```ruby
after_initialize
```

means:

> "Run this after Rails has finished initialization."

This is useful when your code depends on the final state of Rails configuration.

---

# 39. Edge Case — Rake Tasks vs Web Server

Sometimes you might think:

> "I only want this initializer to run when the web server starts."

For example:

```ruby
MyCache.warmup!
```

But remember:

```text
rails server
```

isn't the only thing that boots Rails.

Rake tasks can boot Rails too.

You might see code like:

```ruby
unless defined?(Rails::Console) ||
       File.basename($0) == "rake"

  MyCache.warmup!

end
```

This tries to avoid running the code in certain situations.

However, this approach can be brittle.

A better question is often:

> **Should this work really be happening inside an initializer at all?**

Maybe cache warming belongs in:

* a deployment step
* a separate worker
* a background job
* a dedicated command

rather than Rails boot.

---

# 40. Railtie / Engine Ordering

This gets more advanced.

Rails Engines and gems can define their own initializers.

For example, a gem might define:

```ruby
initializer "my_gem.setup" do
  # setup
end
```

Your application may need to run something:

```text
before the gem initializer
```

or:

```text
after the gem initializer
```

Rails provides an initializer DSL that can express ordering dependencies.

For example, conceptually:

```ruby
initializer "my_setup",
  before: "my_gem.setup" do

  # setup
end
```

This is much better than trying to manipulate filenames.

---

# 41. Comparison Table

| Concept                                          | Scope                | When it runs                 | Reloaded in Development?      | Best For                                                  |
| ------------------------------------------------ | -------------------- | ---------------------------- | ----------------------------- | --------------------------------------------------------- |
| **Initializers** (`config/initializers`)         | Application-wide     | During boot                  | No                            | Gems, Redis, Sidekiq, external service configuration      |
| **Environment Config** (`config/environments`)   | Environment-specific | Early during boot            | No                            | Development/production-specific Rails settings            |
| **Application Config** (`config/application.rb`) | Application-wide     | Very early during boot       | No                            | Base Rails configuration                                  |
| **Rack Middleware**                              | Request-level        | During each request          | Yes, depending on Rails setup | Request interception, headers, low-level request handling |
| **Controller Callbacks**                         | Controller-level     | During controller processing | Yes                           | Authorization, loading records, request-specific setup    |

---

# 42. Related Topics You Should Learn Next

Once you understand initializers, three topics become especially important.

## 1. Railties and Engines

Learn how Rails itself and gems can register initialization steps.

You'll understand things like:

```ruby
initializer "something"
```

and:

```ruby
before:
after:
```

---

## 2. Zeitwerk

Understand:

* autoloading
* constant lookup
* reloadable classes
* why referencing `User` too early can cause problems
* `to_prepare`

This is particularly important for senior Rails developers.

---

## 3. Rails Boot Process

Understand what happens before:

```text
config/initializers/
```

runs.

You should know the relationship between:

```text
config/boot.rb
       ↓
config/application.rb
       ↓
environment config
       ↓
Rails initialization
       ↓
initializers
       ↓
after_initialize
       ↓
Rails ready
```

---

# 43. Summary

The simplest way to understand Rails initializers is:

> **Initializers are Ruby files that Rails runs during startup to configure your application and its dependencies.**

They live here:

```text
config/initializers/
```

For example:

```text
config/initializers/
├── devise.rb
├── redis.rb
├── sidekiq.rb
└── stripe.rb
```

They are useful for things like:

```text
Configure Devise
Configure Redis
Configure Sidekiq
Configure Stripe
Register MIME types
Configure third-party gems
```

They are **not** a good place for:

```text
Business logic
Database queries
Data creation
Heavy computation
Blocking network calls
Random global constants
```

And remember the most important distinction:

```text
BOOT PROCESS
    │
    │  The entire process of starting Rails
    │
    ├── boot.rb
    ├── Bundler / gems
    ├── application.rb
    ├── environment config
    ├── Rails initialization
    │
    ├── INITIALIZERS
    │      ├── devise.rb
    │      ├── redis.rb
    │      ├── sidekiq.rb
    │      └── stripe.rb
    │
    ├── after_initialize
    │
    └── Rails application ready
```

So:

> **Boot process = the entire startup process.**

> **Initializer = one particular type of startup code that runs during that process.**

---

# 44. Cheat Sheet

### Where are initializers?

```text
config/initializers/*.rb
```

### When do they run?

During Rails boot.

### How many times?

Normally once per Rails process boot.

### Do they reload automatically in development?

No.

Restart the server after changing them.

### What are they good for?

```text
Third-party gem configuration
External service configuration
Application-wide configuration
Framework customization
```

### Custom application configuration

```ruby
Rails.application.config.x.custom_key = "value"
```

Access it with:

```ruby
Rails.configuration.x.custom_key
```

### Framework hook

```ruby
ActiveSupport.on_load(:active_record) do
  # customization
end
```

### Application-code hook

```ruby
Rails.application.config.to_prepare do
  # code involving reloadable application classes
end
```

### Run after initialization

```ruby
Rails.application.config.after_initialize do
  # code
end
```

### Require a critical ENV variable

```ruby
ENV.fetch("API_KEY")
```

### Avoid

```text
❌ Database queries
❌ Business logic
❌ Heavy computation
❌ Blocking network requests
❌ Hardcoded secrets
❌ Unsafe references to reloadable app classes
❌ Giant global objects
```

### Golden mental model

```text
Rails Boot
    ↓
"Start and prepare the Rails application"
    ↓
Initializers
    ↓
"Configure things needed by the application"
    ↓
Rails Ready
    ↓
Handle requests / jobs / commands
```

**If you remember only one sentence, remember this:**

> **The boot process is everything Rails does to become ready; initializers are configuration scripts that Rails executes as one part of that boot process.**
