# Rails Boot Process — Senior Backend Interview Study Guide

> Scope: modern Rails (Zeitwerk, Rails 7/8 conventions) on MRI Ruby. Exact initializer and server-command details vary by release; the ordering rules and architecture are stable. For a production incident or deploy-sensitive change, check the exact Rails and server versions.

## 1. Overview

### What it is

The Rails boot process is everything needed to start a Ruby process (for example: a server, console, job worker, test runner, migration, or Rack host) and make the Rails app ready to run work. Booting prepares the app; it is not the same as handling the first HTTP request.

A web process usually boots once and then serves many requests. A process can also boot just to run a single task (runner, migration, or job).

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

## 2. Core Concepts

### Separate three lifecycles

1. Process lifecycle: Ruby process starts, code loads, process exits. Some servers preload and then fork workers.
2. Application lifecycle: Rails.application is configured, initialized, optionally eager loaded, and— in development—may reload code.
3. Work lifecycle: Rack requests and background jobs run repeatedly inside executor/reloader boundaries.

A remote call during boot is a deployment dependency. A remote call during request handling is a traffic dependency. Treat them differently.

### Ruby loading

- require: looks in $LOAD_PATH and evaluates a feature once per process. Use it for gems, stdlib, and files that should not be reloaded.
- require_relative: resolves relative to the current file. Rails entrypoints commonly use this.
- load: re-evaluates a file every time; rarely needed.
- Constant lookup is a Ruby feature. Rails uses Zeitwerk to handle autoloading, reloading, and eager loading of app constants.

Zeitwerk maps file paths to constants. For example, in an autoload root:

~~~ruby
# app/services/payments/capture.rb
module Payments
  class Capture
  end
end
~~~

A file at app/models/user.rb should define User. Do not manually require files that Zeitwerk manages — that bypasses the loader and can make reload behavior wrong.

### Bundler and dependency resolution

config/boot.rb usually sets BUNDLE_GEMFILE and requires bundler/setup. Bundler reads Gemfile.lock and adjusts $LOAD_PATH so require "rails" works.

Bundler.setup doesn't always require every gem. Bundler.require(*Rails.groups) typically requires entrypoints for groups later. Gems may provide Railties that Rails discovers; other gems may require their own files on demand.

### Railties, engines, and Rails.application

A Railtie is a plugin point for Rails. Framework pieces and many gems provide a Railtie. It can add configuration, paths, generators, rake tasks, middleware, and initializers.

An Engine is a richer Railtie that can have app-like paths, routes, and a namespace. Rails::Application is the top-level Engine for your app. Rails collects initializers from the framework, engines, and the host application and runs them in an ordered graph.

### Configuration sources

| Source | Purpose | Timing |
|---|---|---|
| config/boot.rb | Bundler/Bootsnap | earliest |
| config/application.rb | global defaults, selected framework components | application class evaluation |
| config/environments/*.rb | environment overrides | before initializer execution |
| config/initializers/*.rb | integration setup | initializer graph phase |
| ENV / credentials / YAML | values consumed by config | process start or component-specific |
| config.ru / server config | Rack/server behavior | entrypoint/server phase |

Environment config usually wins because it's evaluated later. config.load_defaults sets framework defaults for a Rails version — it changes behavior and is not just cosmetic.

### Initializers and lifecycle hooks

An initializer is a named block of boot work. You can give it before: and after: dependencies. Rails orders initializers with topological sorting. Cycles are errors — don't rely on filenames for order.

~~~ruby
# config/application.rb
module Ledger
  class Application < Rails::Application
    config.before_configuration do
      # Application subclass setup.
    end

    config.before_initialize do
      # Directly before initializer execution.
    end

    config.after_initialize do
      # Once after Rails initialization.
    end

    config.to_prepare do
      # Boot and every reload preparation in reloadable environments.
    end
  end
end
~~~

after_initialize does not guarantee a listening socket, a worker fork, database reachability, or a first request. Don't assume external resources are ready just because after_initialize ran.

### Reloading and to_prepare

Zeitwerk is responsible for reloadable app code. In development, when a file changes, Rails unloads those constants before the next work boundary and sets up autoloads again. Long-lived global objects that hold reloadable classes can become stale.

Use Rails.application.reloader.to_prepare for wiring that must run both at boot and after code reloads. to_prepare must be idempotent (replacement or assignment is good; appending is dangerous).

### Autoloading, eager loading, and once paths

Rails has two main autoloaders:

- Rails.autoloaders.main: for reloadable application and engine code.
- Rails.autoloaders.once: for autoload_once_paths — these can autoload but are not unloaded on reload.

When config.eager_load = true, Rails loads eager-load paths at boot. Benefits: filename/constant validation at deploy time, no first-use load latency, and better copy-on-write memory sharing. Costs: longer boot and more memory; top-level side effects run at boot.

### Rack middleware and routes

Rack apps implement call(env) and return [status, headers, body]. Rails collects middleware declarations and later builds a nested callable stack around the endpoint. If you add middleware A then B, the request flows through A then B; the response flows back out the other way.

Routes load during application setup. The route set provides recognition, URL generation, helpers, mounted engines, and supports reloading in development.

### Executor and reloader

The executor wraps application work and manages execution context like the query cache and connection behavior. The reloader decides whether changed code should be unloaded before handling work.

### Spring, Bootsnap, preload, and forking

- Spring keeps a development process running to make commands faster. Restart Spring when diagnosing boot changes.
- Bootsnap caches expensive require resolution and compilation steps; it speeds boot, but does not change order.
- Servers like Puma (with preload_app!), Unicorn, and Passenger can load code in a parent and fork workers. Code loaded before fork may be shared across workers via copy-on-write. Be careful: open DB connections, sockets, threads, and mutable globals may be incorrectly shared across forks unless reset after fork.

---

## 3. Internal Working

### Canonical server timeline

~~~text
shell
  -> Ruby executes bin/rails
  -> config/boot.rb: Bundler, optional Bootsnap
  -> rails/commands dispatches server
  -> server/Rack host loads config.ru or config/environment.rb
  -> config/application.rb defines YourApp::Application
       -> rails/all selects framework Railties
       -> application + environment configuration evaluated
  -> Rails.application.initialize!
       -> ordered Railtie/engine/application initializers
       -> autoloaders, logger, DB config, middleware, routes
       -> eager loading if configured
  -> Rack server binds and accepts requests
  -> each request: executor/reloader -> middleware -> route -> endpoint
~~~

Generated bin/rails is intentionally small:

~~~ruby
#!/usr/bin/env ruby
APP_PATH = File.expand_path("../config/application", __dir__)
require_relative "../config/boot"
require "rails/commands"
~~~

A typical boot file looks like:

~~~ruby
ENV["BUNDLE_GEMFILE"] ||= File.expand_path("../Gemfile", __dir__)
require "bundler/setup"
require "bootsnap/setup"
~~~

Bundler makes gems loadable first. Then an app-aware command loads the Rails environment and initializes the application.

### Application creation and framework selection

~~~ruby
require_relative "boot"
require "rails/all"

Bundler.require(*Rails.groups)

module Ledger
  class Application < Rails::Application
    config.load_defaults 7.2
    config.autoload_lib(ignore: %w[assets tasks])
  end
end
~~~

rails/all loads the standard framework Railties. You can load a smaller set of components if you only need some parts:

~~~ruby
require "rails"
require "active_model/railtie"
require "active_job/railtie"
require "active_record/railtie"
require "action_controller/railtie"
require "action_mailer/railtie"
~~~

When the Application subclass runs, Rails records configuration and paths. Rails.application is a singleton. Rails.env normally comes from RAILS_ENV, then RACK_ENV, otherwise defaults to "development".

### Initializer graph mechanics

Railties expose initializer(name, options, &block). Rails gathers initializers from all contributors, builds a dependency graph with TSort, and runs them in a valid order.

~~~ruby
module Ledger
  class Railtie < Rails::Railtie
    initializer "ledger.configure", after: "active_record.initialize_database" do |app|
      Ledger.configure(app.config.x.ledger)
    end

    initializer "ledger.middleware", before: "build_middleware_stack" do |app|
      app.middleware.insert_before 0, Ledger::RequestId
    end
  end
end
~~~

Anchor names (like "build_middleware_stack") can change between Rails releases. Inspect the framework version before binding to internal names. Prefer public config hooks when possible.

Useful phases:

1. Bootstrap: set load paths and basic Active Support/logging/config.
2. Railtie and Engine configuration: components add paths and register behavior.
3. Application initialization: config/initializers and engine contributions run in the correct graph phase.
4. Finish: routes and middleware are finalized; after_initialize and eager loading run at their configured phases.

Don't try to memorize every initializer name. Explain how ordering and ownership work, and how you would inspect the specific Rails version.

### Zeitwerk internals

Zeitwerk scans loader roots, maps paths to constants, and registers autoloads. Accessing Payments::Capture causes Zeitwerk to require the file it expects. Rails uses a reloadable main loader and an optional once loader.

Eager loading asks every loader to require eligible files. If a file does not match its constant name (for example app/clients/stripe_client.rb defines StripeAPI), eager loading will raise a Zeitwerk::NameError. Use zeitwerk:check in CI to catch these problems.

### PostgreSQL and Active Record

Active Record's Railtie reads database config and sets up connection pool handling. Boot does not always open a PostgreSQL connection — checkout is usually lazy and happens when a thread needs a connection.

When a connection is created, the pg gem handles authentication/TLS; Active Record wraps connections and pools them. With multiple roles, replicas, or shards, this becomes more complex.

Danger: running a DB query (like User.count) in an initializer can cause many processes to open DB connections during a rolling deploy and exhaust max_connections. Avoid queries during boot when possible.

### Middleware/server handoff

Rails collects middleware declarations while configuring. Later it builds and instantiates the middleware stack around the endpoint. The Rack handler binds a socket and calls the assembled Rack app per request.

### Entrypoint variants

| Entrypoint | Difference |
|---|---|
| bin/rails server | Initializes then starts Rack server; server may preload/fork. |
| config.ru/rackup | Rack host requires environment and runs Rails.application; no bin/rails dispatch necessarily. |
| bin/rails console | Initializes then enters console; no active web request file watcher. |
| bin/rails runner | Initializes, runs code, exits; good as a boot smoke test. |
| Rails/Rake task | Loads tasks/app as needed; config.rake_eager_load matters. |
| Sidekiq/worker | Requires Rails env, then runs a worker loop and job lifecycle. |
| test | Boots test environment; may not eager load unless CI config enforces it. |

---

## 4. Architecture

~~~mermaid
flowchart TD
  A["OS process / entrypoint"] --> B["Ruby + Bundler + optional Bootsnap"]
  B --> C["Rails command or Rack host"]
  C --> D["Application class + framework Railties"]
  D --> E["Application, environment, engine configuration"]
  E --> F["Initializer dependency graph"]
  F --> G["Autoloaders / middleware / routes / integrations"]
  G --> H{"eager_load?"}
  H -- yes --> I["Zeitwerk eager load"]
  H -- no --> J["On-demand autoloading"]
  I --> K["Server / job / console runtime"]
  J --> K
  K --> L["Executor + reloader boundary"]
  L --> M["Rack middleware -> routes -> controller/job"]
~~~

Rails connects Ruby code loading and runtime to the Rack HTTP interface. Railties are how code plugs into Rails; middleware composes the HTTP path outwards. Engines add paths and behavior to the host app.

---

## 5. Real Production Examples

### Production-only eager-load failure

~~~ruby
# app/clients/stripe_client.rb
class StripeAPI
end
~~~

This may appear to work in development if callers reference StripeAPI lazily. In production, eager loading expects StripeClient and will raise an error early. Fix the file name or the constant name, or add an inflector rule.

### Multi-tenant SaaS connection storm

If an initializer reads all tenants and each new process opens DB connections, a rolling deploy can start many processes at once and exhaust DB connections.

Better: only boot static tenant config, and lazy-load tenant data after readiness with a bounded cache. Count pools per process and role when planning capacity.

### Reloadable policy registered once

~~~ruby
# Bad: config/initializers/auth.rb
AuthGateway.policy_class = AccountPolicy
~~~

AccountPolicy is reloadable. After a development reload, AuthGateway might keep the old class.

~~~ruby
# Good
Rails.application.reloader.to_prepare do
  AuthGateway.policy_class = AccountPolicy
end
~~~

Use to_prepare and make the registration replace state, not append duplicates.

### Preload/fork client hazard

~~~ruby
# Bad in a preloaded parent
PAYMENTS = Faraday.new(url: ENV.fetch("PAYMENTS_URL"))
~~~

A client opened before fork can keep sockets or threads across workers.

~~~ruby
module Payments
  def self.client
    @client ||= Faraday.new(url: ENV.fetch("PAYMENTS_URL"))
  end

  def self.reset!
    @client = nil
  end
end

# config/puma.rb; exact hook depends on server mode
on_worker_boot { Payments.reset! }
~~~

Also use standard hooks to reset Active Record connections after fork.

### Feature flags

~~~ruby
# Bad: hidden deploy-time dependency
FeatureFlags.sync_from_remote!
~~~

Configure the feature-flag client at boot but refresh flags lazily or via a controlled warmup. If your policy requires current flags before serving traffic, make that a clear readiness requirement with deadlines and fallbacks.

---

## 6. Common Mistakes

### Junior

- Running DB queries or remote I/O in an initializer.
- Requiring an autoloadable app file to fix a missing constant.
- Scattering Rails.env checks around initializers instead of using environment config.
- Depending on alphabetic ordering of initializer filenames.
- Storing secrets in code or logging environment values.

### Mid-level

- Keeping reloadable class objects in long-lived global registries.
- Adding all of lib to autoload/eager-load paths without ignoring task and generator files.
- Turning off production eager loading to make boot faster.
- Inserting middleware without checking the real stack order.
- Setting DB pool size based on machine-wide concurrency instead of per-process needs.
- Calling Rails.application.initialize! manually from scripts.

### Senior

- Treating private Rails or gem initializer names as stable public APIs.
- Preloading code that leaves sockets, DB connections, threads, or mutable singletons open across fork.
- Making boot depend on third-party services without timeouts, fallbacks, and alerts.
- Building global registries that require a full restart to update.
- Putting expensive allocations or side effects in eager-loaded files.

### Staff-level blind spot

Treating startup as only a developer concern. Startup time, memory, connection fan-out, crash loops, and external dependency policies affect deploy speed, autoscaling, and incident behavior.

---

## 7. Performance Considerations

### Measure

Track these metrics:

- Time from process start to ready-to-serve.
- Duration of slow initializers.
- Memory before and after eager load and after fork.
- Connection counts made during boot.
- Boot/restart failure rate.
- First-request latency vs. startup cost.

A quick baseline is: time bin/rails runner 'puts :ok'. But measure with the real production image, filesystem, environment, and fork mode.

### Levers

| Lever | Benefit | Trade-off |
|---|---|---|
| Bootsnap | Faster require lookups and compilation cache | Cache troubleshooting; extra deploy storage to measure. |
| Eager load | No first-use loading, deploy-time validation, CoW-friendly | Longer boot and more memory; top-level side effects run at boot. |
| preload + fork | Shares immutable pages across workers | You must reset connections/clients/threads after fork. |
| add_autoload_paths_to_load_path = false | Fewer Ruby lookups and Bootsnap indexes | Must follow Zeitwerk and avoid requiring app files. |
| Slim framework selection | Less code and middleware | You take on integration of omitted parts. |
| Lazy clients/caches | Faster, less fragile boot | First-use latency and still needs failure handling. |
| Background warmup | Move noncritical work off the critical path | Can stampede dependencies; readiness must be clear. |

### Copy-on-write

A child process after fork shares memory pages with the parent until one writes to a page. Eagerly load immutable code before fork to improve sharing. Avoid per-worker mutations of global hashes or memoized registries that reduce sharing.

### Avoid boot I/O

Network calls add DNS, TLS, rate limits, and credential dependency to the deploy path. If you can't avoid network calls at boot, use short timeouts, bounded retries with jitter, and clear fallbacks.

### Pool math

~~~text
maximum possible PostgreSQL connections
≈ web processes × pool per web process
 + job processes × pool per job process
 + migration/console/monitor/deploy headroom
~~~

Roles and shards multiply connection needs. Choose pool sizes based on concurrent DB users per process, not generic request concurrency. Rolling deploy overlap and initializer queries increase demand.

---

## 8. Security Considerations

- Boot runs privileged app code. Be careful with supply-chain and gem behavior — gems run with app credentials.
- Never log full ENV, credential objects, DB URLs, or tokens. Boot logs are widely retained.
- Required secrets should cause a clear boot failure with a non-secret error message. Optional services should have a degraded implementation.
- Treat environment values as untrusted input: parse and validate URLs, hosts, integers, booleans; never eval or shell-interpolate them.
- Protect health and readiness endpoints so you don't leak topology or error internals.
- Understand how sensitive descriptors, tokens, or clients behave across forks and during rotation.
- Separate build-time and runtime secret needs. Don't require production credentials during asset builds or CI unless absolutely necessary.

---

## 9. Debugging

### Classify before changing things

| Symptom | Likely phase | First move |
|---|---|---|
| cannot load such file before app boot | Bundler/load path/Bootsnap | Check Ruby/Bundler/Gemfile.lock and bundle exec; clear caches. |
| Error in config/application.rb | config evaluation | Read the first app frame in the stack and remove premature constant or I/O. |
| Error in config/initializers | initializer graph | Find the initializer, check ordering and I/O, and reproduce with bin/rails runner. |
| Zeitwerk::NameError in CI/prod | eager load | Run zeitwerk:check and fix name/path or inflector issues. |
| Works until source edit | reloader/stale class | Inspect to_prepare usage and cached class references. |
| Pods slow/not ready | I/O/eager work/pool | Time boot phases and count connections or dependencies. |
| rails s works, server host fails | entrypoint/preload/fork | Compare the commands, env, and server hooks. |

### High-signal commands

~~~bash
bin/rails runner 'puts "booted: #{Rails.version}"'
bin/rails zeitwerk:check
bin/rails about
bin/rails middleware
bin/rails routes
RAILS_ENV=production bin/rails runner 'puts :ok'
bin/rails runner 'pp Rails.application.initializers.map(&:name)'
~~~

When simulating production, use safe dummy config. Don't point local CI at real production services.

### Inspect from console

~~~ruby
Rails.application.config.eager_load
Rails.application.config.autoload_paths
Rails.application.config.eager_load_paths
Rails.autoloaders.main.dirs
Rails.autoloaders.once.dirs
Rails.application.middleware.map(&:klass)
~~~

Middleware shapes differ by Rails version and installed gems; don't assume the exact shape across apps.

### Time initialization

~~~ruby
# config/initializers/boot_timing.rb
started_at = Process.clock_gettime(Process::CLOCK_MONOTONIC)

Rails.application.config.after_initialize do
  elapsed = Process.clock_gettime(Process::CLOCK_MONOTONIC) - started_at
  Rails.logger.info(event: "rails.initialized", elapsed_seconds: elapsed.round(3))
end
~~~

Use monotonic timing or ActiveSupport::Notifications for accurate durations. Log outcomes but not secrets.

### Autoload diagnostics

~~~ruby
# temporary, noisy diagnostic
Rails.autoloaders.log!

# after Rails.logger exists
Rails.autoloaders.logger = Rails.logger
~~~

Run zeitwerk:check to catch naming issues in CI.

### Incident procedure

1. Capture the full exception, the first app frame, command, versions, RAILS_ENV, server mode, and revision.
2. Reproduce with the smallest entrypoint that matches the problem (runner for init issues, real server for preload/fork behavior).
3. Isolate application config, suspect initializers, eager load, and worker boot using timestamps.
4. For external failures, verify DNS, credentials, firewall, quotas, and concurrent startup from the deploy environment.
5. Fix lifecycle ownership, add regression tests, and remove temporary diagnostics.

After changing Gemfile, boot config, application config, or initializers, restart Spring/server. If you suspect a stale Bootsnap cache, clear the project-specific cache safely.

---

## 10. Best Practices

1. Make boot deterministic, fast, idempotent, and observable.
2. Put global defaults in application.rb and environment overrides in environment files.
3. Keep initializers small. Move business logic into testable objects.
4. Avoid DB writes/queries, remote I/O, long work, background threads, and irreversible side effects in boot.
5. Run production-like eager-load checks in CI.
6. Follow Zeitwerk naming rules; configure inflections when needed; never require reloadable app code manually.
7. Use to_prepare only for idempotent reload-aware wiring.
8. Define which code owns connections/clients/threads before and after fork.
9. Publish boot duration, memory, and connection budgets.
10. Test with the real production server/process model, not just rails server locally.

---

## 11. Anti-patterns

### Initializer junk drawer

~~~ruby
# config/initializers/setup_everything.rb
User.find_each { |u| SearchIndex.sync(u) }
Faraday.get(ENV.fetch("CONFIG_URL"))
Thread.new { loop { Metrics.flush } }
~~~

Putting this kind of work in an initializer forces every command to need DB and vendor availability, repeats work on every boot, and creates unclear thread/fork/shutdown behavior. Use migrations, background jobs, or deploy tasks instead.

### Top-level side effects in autoloaded code

~~~ruby
# app/services/rates.rb
Rates.refresh!
class Rates; end
~~~

Top-level code runs when a file autoloads, eager loads, or reloads. File evaluation is not a lifecycle hook and eager-load order is not guaranteed.

### Filename ordering

Files named 01_config.rb and 99_patch.rb hide ordering dependencies. Prefer a public config API or a named dependency graph.

### Global singleton with hidden lifecycle

~~~ruby
CLIENT = Vendor::Client.new(token: Rails.application.credentials.vendor_token)
~~~

Such singletons are hard to rotate, test, fork, or reload. Prefer factories or injected configs with explicit reset methods.

### Rescuing into a half-initialized app

~~~ruby
begin
  CriticalService.configure!
rescue StandardError => e
  Rails.logger.error(e.message)
end
~~~

For critical setup, fail fast. For optional setup, provide a clear degraded path and expose status. A half-initialized app serving traffic is worse than a clear boot failure.

---

## 12. Interview Questions

### Basic

1. **What runs after bin/rails server?** Ruby loads boot.rb, Bundler/Bootsnap, command dispatch, app/environment, Rails.application.initialize!, then the Rack server runtime.
2. **boot.rb vs environment.rb?** config/boot.rb sets up Bundler and caches (Bootsnap). config/environment.rb loads the application and initializes it.
3. **What is an initializer?** A named, orderable unit of boot setup from the framework, engines, or app.
4. **Why eager-load production?** It finds filename/constant errors early, removes first-use latency, and can improve copy-on-write memory sharing.

### Intermediate

5. **How do engines participate?** Engines are Railties with paths and routes; their initializers join the host app's initializer graph and their paths join the loader.
6. **Why can one-shot setup fail after development reload?** The app may have stored a class object that Rails later unloads and redefines. Use idempotent to_prepare or lazy resolution.
7. **Does boot always connect PostgreSQL?** No. Rails sets up pool config at boot, but connections are usually checked out lazily when needed.
8. **What does Bootsnap change?** Startup speed via cached require resolution and compilation, not lifecycle order.
9. **How does middleware order work?** Middleware wraps the app in nested Rack callables; the outer layer sees requests first and responses last.

### Senior

10. **How reduce a 45-second boot?** Measure, remove blocking I/O and data work, defer noncritical clients, remove eager-load side effects, test Bootsnap/preload, and validate memory impact.
11. **preload_app! vs eager_load?** preload_app! loads code in a parent before fork; eager_load loads constants. If you combine them, be sure to reconnect/reset resources after fork.
12. **Production eager-load error but dev works?** Reproduce with production-like config, run zeitwerk:check, fix naming/inflector issues, and add a CI guard.
13. **A gem needs an app class in config?** Use the gem's public API. If the app class reloads, assign idempotently in to_prepare or refer to the class by name lazily.

### Staff

14. **Startup strategy for 200 services?** Define a startup SLO, avoid hidden boot I/O, run eager-load checks in CI, add phased telemetry, document pre/post-fork hooks, and set clear readiness contracts and rollout plans.
15. **Feature flags must be current at startup?** Decide the safety level first. If required, make readiness depend on a bounded warmup with fallback and monitoring. If not, allow lazy refresh.
16. **How evolve initializer contracts across engines?** Provide public config hooks, avoid relying on private initializer names, make initialization idempotent, test host combinations, and document contracts.

---

## 13. Practical Coding Examples

### A. Declarative config plus injection

~~~ruby
# config/application.rb
module Ledger
  class Application < Rails::Application
    config.load_defaults 7.2
    config.x.reconciliation.batch_size = 500
  end
end

# config/environments/production.rb
Rails.application.configure do
  config.x.reconciliation.batch_size =
    Integer(ENV.fetch("RECONCILIATION_BATCH_SIZE", 1_000))
end

# app/services/reconciliation/run.rb
module Reconciliation
  class Run
    def initialize(batch_size: Rails.configuration.x.reconciliation.batch_size)
      @batch_size = batch_size
    end
  end
end
~~~

This keeps settings declarative and testable without doing work at boot.

### B. A thin internal Railtie

~~~ruby
# lib/acme/audit/railtie.rb
module Acme
  module Audit
    class Railtie < Rails::Railtie
      config.acme_audit = ActiveSupport::OrderedOptions.new

      initializer "acme_audit.configure", before: "load_config_initializers" do |app|
        Acme::Audit.configure(
          enabled: app.config.acme_audit.fetch(:enabled, true),
          sink: app.config.acme_audit[:sink]
        )
      end

      initializer "acme_audit.middleware", before: "build_middleware_stack" do |app|
        app.middleware.use Acme::Audit::RackMiddleware
      end
    end
  end
end
~~~

Let the integration own its setup and expose configuration. Verify any private anchor names for your Rails version.

### C. Reload-safe registration

~~~ruby
# config/initializers/serializers.rb
Rails.application.reloader.to_prepare do
  ApiRenderer.register("invoice", Invoices::Serializer)
end

# registry must replace, not append
def self.register(name, klass)
  @serializers ||= {}
  @serializers[name] = klass
end
~~~

### D. Avoid accidental DB boot work

~~~ruby
# Bad
DEFAULT_PLAN = Plan.find_by!(slug: "starter")

# Better
class Plans::Default
  def self.fetch
    Plan.find_by!(slug: "starter")
  end
end
~~~

Store identifiers in config and create required data with migrations, seeds, or deploy orchestration. Verify with health checks rather than running queries at every boot.

### E. Lazy external client

~~~ruby
module Risk
  class Client
    def self.instance
      @instance ||= new(url: ENV.fetch("RISK_URL"))
    end

    def initialize(url:)
      @connection = Faraday.new(url:) do |f|
        f.options.open_timeout = 0.2
        f.options.timeout = 0.5
      end
    end
  end
end
~~~

This delays socket creation until needed. Add error handling, retries, metrics, and post-fork reset logic.

### F. Eager-load CI smoke test

~~~ruby
# test/integration/eager_loading_test.rb
require "test_helper"

class EagerLoadingTest < ActiveSupport::TestCase
  test "application eager loads" do
    Rails.application.eager_load!
  end
end
~~~

Or in CI:

~~~bash
RAILS_ENV=production SECRET_KEY_BASE=dummy bin/rails zeitwerk:check
~~~

Use safe dummy config values; never point CI to production services.

### G. Explain middleware placement

~~~ruby
# config/application.rb
config.middleware.insert_before Rack::Head, RequestCorrelationId
~~~

A correlation ID should be available early so downstream logs can use it and the response can return it. Validate the stack with bin/rails middleware because it varies by version and gems.

---

## 14. Edge Cases

| Edge case | Why it surprises teams | Correct response |
|---|---|---|
| runner boots, server fails | socket binding or preload happens later | Test the real server command and hooks. |
| config.ru differs from bin/rails | different host/command/env/config | Compare command args, env, and server config. |
| console sees old code | console doesn't run a file-watcher loop | Reload or restart intentionally. |
| Worker starts before migrations | app boots against an old schema | Coordinate releases with expand-contract migrations. |
| STI subclasses missing in dev | lazy load cannot enumerate unknown subclasses | Eager load or preload the hierarchy with a documented pattern. |
| Acronym only fails CI | file name vs constant name mismatch (VATReport vs VatReport) | Configure an inflector or rename and run zeitwerk:check. |
| setup runs twice | to_prepare, reload, or tests may run init multiple times | Make setup idempotent. |
| lib/tasks defines constants | broad lib autoload makes task files part of the loader | autoload_lib with ignores or move runtime code elsewhere. |
| Rails.logger unavailable | logging initializes later | Move code later or use an early logger. |
| credentials absent at build | build vs run contracts differ | Separate what needs secrets at build and at runtime. |
| SIGTERM during slow boot | orchestrator kills before ready | Make boot fast and idempotent; avoid partial side effects. |
| connection inherited after fork | parent opened connections pre-fork | Reset or reconnect in server worker hooks. |
| circular initializer graph | no valid ordering | Remove coupling or create a clear phase. |

---

## 15. Comparison Table

| Concept | When | Repeats? | Good for | Bad for |
|---|---|---:|---|---|
| config/boot.rb | earliest startup | once/process | Bundler/Bootsnap | model access/app config |
| application.rb | app class evaluation | once/process | global declarative config | I/O/data work |
| environment config | before initialization | once/process | deployment overrides | ordering side effects |
| initializers | graph phase | normally once/process | thin global integration | jobs, remote sync, DB writes |
| before_initialize | immediately before graph | once/process | rare positioning | normal configuration |
| after_initialize | after graph | once/process | quick final wiring | assuming request/fork readiness |
| to_prepare | boot + reload preparation | boot + reload | reload-aware assignment | non-idempotent work |
| eager_load! | eager phase/manual | load cycle | validation/preload | load-order side effects |
| require | first feature request | once/process | gems/std lib | Zeitwerk-owned app files |
| middleware | request/response | every request | HTTP cross-cutting concerns | one-time setup |
| migration | explicit deployment | intentional | schema/data evolution | boot setup |

---

## 16. Related Topics

Study next:

1. Zeitwerk: roots, inflectors, ignored/collapsed paths, stale objects, eager-load tests.
2. Rack/Rails request lifecycle: middleware, env, routes, controller dispatch, streaming.
3. Rails configuration/credentials: defaults, custom config, 12-factor deployment.
4. Puma and Ruby concurrency: workers vs threads, CoW, signals, graceful shutdown.
5. Active Record connection handling: pools, roles/replicas/shards, query cache, PostgreSQL limits.
6. Engines/Railties: clean Rails extension points.
7. Active Support executor/reloader and thread context.
8. Deployment engineering: readiness/liveness, rolling rollout, migrations, incident response.

---

## 17. Summary — Revision Sheet

- Boot makes a Ruby process ready to run a Rails app; it is separate from request handling.
- Typical path: bin/rails -> config/boot.rb -> Rails command/Rack host -> application/environment -> initialize! -> middleware/routes/autoloaders -> server runtime.
- Railties and engines add configuration and initializers that Rails runs in dependency order.
- Put declarative settings in app/environment config; keep initializers small and side-effect-free.
- Zeitwerk owns app code. Match file paths to constants, avoid requiring reloadable app files, and test eager loading in CI.
- Eager load helps validation and CoW but costs time and memory. Its exact order is not guaranteed.
- to_prepare runs at boot and after reloads. Keep it idempotent.
- Active Record sets up pools at boot; code may still force real DB connections. Count connections during rollout planning.
- Preload/fork is different from eager load. Reset or reconnect clients, pools, and threads after fork.
- Startup duration, memory, connection fan-out, and dependency policies matter for reliability.

---

## 18. Cheat Sheet — One Page

~~~text
ENTRYPOINT
bin/rails -> config/boot.rb -> Rails command -> config/environment.rb
          -> application.rb + env config -> Rails.application.initialize!
          -> middleware/routes/autoloaders -> eager load? -> runtime

PLACE CODE
global config:       config/application.rb
environment override: config/environments/production.rb
global integration:  initializer / Railtie
reload-aware wiring: Rails.application.reloader.to_prepare
per request:         middleware/controller/service
schema/data:         migration/job/deploy task

DO NOT PUT IN BOOT BY DEFAULT
remote I/O | DB query/write | long work | threads | unbounded retries
manual require for Zeitwerk app file | reliance on eager-load order

COMMANDS
bin/rails runner 'puts :ok'
bin/rails zeitwerk:check
bin/rails about
bin/rails middleware
RAILS_ENV=production bin/rails runner 'puts :ok'

RELOAD
main-loader class objects can be unloaded/redefined in development.
Do not retain them in a nonreloadable global; use idempotent to_prepare.

FORK
preload_app! != eager_load.
Post-fork reset/reconnect DB, sockets, clients, threads, mutable state.

INTERVIEW SOUND BITE
Rails boot is a dependency-ordered composition phase: Bundler makes locked
gems loadable; Railties/engines contribute config and initializers; Rails
finalizes autoloaders, middleware and routes; production normally eager loads
before traffic. Keep it deterministic and I/O-light; treat boot time,
connection fan-out, and fork safety as reliability concerns.
~~~

---

## 19. Practice Exercises

### Easy

1. Add after_initialize logging to a fresh app. Compare runner, console, and server process output.
2. Run bin/rails middleware. Explain request and response direction through three layers.
3. Create app/services/billing/tax_calculator.rb, intentionally break its defined constant, run zeitwerk:check, and repair it.
4. Add config.x in application.rb, override it in development, and inject it into a service without doing boot work.

### Medium

5. Write an internal Railtie that adds middleware and one config option. Test it in a dummy host.
6. Demonstrate a stale reloadable class held in a global registry; repair it with idempotent to_prepare.
7. Instrument three expensive initializers. Produce startup contribution data and eliminate nonessential I/O.
8. Add a production-like eager-load CI job with safe dummy config.
9. Compare full-stack and intentionally slim API app boot/middleware. Separate framework effects from gem effects.

### Hard

10. Run Puma preloaded with workers. Deliberately create a pre-fork HTTP client and DB checkout; refactor to post-fork-safe hooks and measure memory/connections.
11. Design readiness for mandatory payment secret, optional Redis, and feature flags allowed to be stale ten minutes. Define timeouts, fallback, status, metrics, and rollout behavior.
12. Write an RFC for a 40-second boot: measurement, SLO, dependency classification, connection math for 25% rolling rollout, changes, risks, rollback.
13. Build a mountable engine usable by two hosts. Ensure public config, no leaking globals, reload safety, and no private Rails initializer dependency.

---

## 20. Additional Resources

### Official docs and source

- [The Rails Initialization Process](https://guides.rubyonrails.org/initialization.html) — canonical deep walkthrough.
- [Configuring Rails Applications](https://guides.rubyonrails.org/configuring.html) — hooks, config, and defaults.
- [Autoloading and Reloading Constants](https://guides.rubyonrails.org/autoloading_and_reloading_constants.html) — essential Zeitwerk guide.
- [Rails Railties source](https://github.com/rails/rails/tree/main/railties/lib/rails) — use the release tag matching Gemfile.lock.
- [Rails Initializable source](https://github.com/rails/rails/blob/main/railties/lib/rails/initializable.rb) — initializer ordering.
- [Rails Application Bootstrap](https://github.com/rails/rails/blob/main/railties/lib/rails/application/bootstrap.rb) and [Finisher](https://github.com/rails/rails/blob/main/railties/lib/rails/application/finisher.rb) — framework bootstrap details.
- [Zeitwerk](https://github.com/fxn/zeitwerk) — loader rules/APIs beneath Rails integration.
- [Rack SPEC](https://github.com/rack/rack/blob/main/SPEC.rdoc).
- [Puma documentation](https://puma.io/puma/).
- [Bundler Gemfile guide](https://bundler.io/guides/gemfile.html).

### Books and videos

- *Crafting Rails Applications* — José Valim.
- *Rails AntiPatterns* — Chad Pytel and Tammer Saleh. Its lifecycle lessons are useful; validate APIs against modern Rails.
- RailsConf/RubyKaigi talks by Rails core members. Xavier Noria’s Zeitwerk/autoloading talks pair especially well with the official guide.

### Source-reading drill

At the exact Rails tag your app uses, trace:

1. bin/rails
2. config/boot.rb
3. rails/commands command dispatch
4. config/environment.rb and config/application.rb
5. Rails::Application#initialize!
6. Rails::Initializable and Rails::Application::Finisher
7. your resulting middleware stack and autoloaders

This turns memorized terms into an architecture you can reason about under interview pressure.
