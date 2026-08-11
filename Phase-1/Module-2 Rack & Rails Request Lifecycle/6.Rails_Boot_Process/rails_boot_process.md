# Rails Boot Process — Senior Backend Interview Study Guide

> Scope: modern Rails (Zeitwerk, Rails 7/8 conventions) on MRI Ruby. Exact initializer and server-command details vary by release; the ordering rules and architecture are stable. For a production incident, read the Rails source at the tag in Gemfile.lock.

## 1. Overview

### What it is

The Rails boot process is the work from starting a Ruby process—server, console, job worker, test, runner, migration, or Rack host—to an initialized Rails application that can execute work. Rails loads the resolved gems and framework components, creates Rails.application, evaluates app/environment/engine configuration, executes a dependency-ordered initializer graph, configures autoloading, builds routes and middleware, and usually eager loads code in production.

Boot is not the first HTTP request. A process can boot only to run a task. A web process generally boots once per process and then handles many requests.

### Why it exists

Ruby normally loads dependencies explicitly. Rails coordinates a much larger composition problem:

- Bundler must expose a reproducible set of gems.
- Framework components and engines must contribute setup in a safe order.
- Rack middleware must become one callable application.
- Zeitwerk needs paths, inflections, and reload/eager-load policy.
- Logging, credentials, cache, DB configuration, routes, Active Job, I18n, Active Record, and Action Mailer need lifecycle boundaries.

The payoff is convention and extensibility. The cost is startup time, memory, implicit order, and a stage where some APIs are not ready.

### When this matters

Use this knowledge to diagnose pre-request crashes; decide whether setup belongs in configuration, initializer, to_prepare, or runtime; optimize cold starts; build engines/Railties; design preload/fork safety; and explain production deployment behavior.

### Common misconceptions

| Misconception | Reality |
|---|---|
| Rails loads every app file at startup. | Only when eager loading is enabled. Development normally autoloads on demand. |
| Every initializer runs after Rails is ready. | Initializers run during initialization. after_initialize follows that sequence, not necessarily worker fork or first request. |
| An initializer is the place for all setup. | It is process-global boot code, not a home for data jobs, remote sync, or per-request work. |
| boot.rb boots Rails. | It normally establishes Bundler and Bootsnap; framework/app initialization comes later. |
| preload_app! equals eager load. | Preloading loads an app in a parent before fork; eager loading loads constants. They are distinct. |
| Production boot proves all code works. | It validates many load/name contracts, not every route, secret, query, or downstream runtime path. |

---

## 2. Core Concepts

### Separate three lifecycles

1. **Process lifecycle:** Ruby starts, code loads, process exits. A server may fork workers after preloading.
2. **Application lifecycle:** Rails.application is configured, initialized, optionally eager loaded, and may reload code in development.
3. **Work lifecycle:** Rack requests and jobs run repeatedly inside executor/reloader boundaries.

A boot-time remote call is a deployment dependency. A request-time remote call is a traffic dependency. Do not blur them.

### Ruby loading

- require searches $LOAD_PATH and evaluates a feature once per process, recording it in $LOADED_FEATURES. Use it for gems, standard library, and intentionally non-reloadable files.
- require_relative resolves relative to the current source file. Generated Rails entrypoints use it.
- load evaluates a file every time and is rarely appropriate.
- Ruby constant lookup is a language feature. Modern Rails delegates app constant autoloading/reloading/eager loading to Zeitwerk.

Zeitwerk’s convention is path-to-constant. In an autoload root:

~~~ruby
# app/services/payments/capture.rb
module Payments
  class Capture
  end
end
~~~

A file at app/models/user.rb should define User. Do not require an app file managed by Zeitwerk: it bypasses loader ownership and can make reload behavior inconsistent.

### Bundler and dependency resolution

config/boot.rb conventionally sets BUNDLE_GEMFILE then requires bundler/setup. Bundler uses Gemfile.lock’s resolved dependency graph and changes $LOAD_PATH. This makes require "rails" and require "pg" resolve to selected versions.

Bundler setup does not necessarily require every gem. Bundler.require(*Rails.groups) commonly requires group entrypoints later. A gem may have a Railtie that Rails discovers; another may require its own config block; another is just a library.

### Railties, engines, and Rails.application

A Railtie is Rails’ integration contract. Rails framework components and many gems provide one. It can contribute configuration, paths, generators, rake tasks, middleware, and initializers.

An Engine is a richer Railtie with app-like paths, routes, and possibly an isolated namespace. Rails::Application is the top-level Engine for the host app. Rails gathers initializers from the application, engines, and Railties into a unified graph.

### Configuration sources

| Source | Purpose | Timing |
|---|---|---|
| config/boot.rb | Bundler/Bootsnap | earliest |
| config/application.rb | global defaults, selected framework components | application class evaluation |
| config/environments/*.rb | environment overrides | before initializer execution |
| config/initializers/*.rb | integration setup | initializer graph phase |
| ENV / credentials / YAML | values consumed by config | process start or component-specific |
| config.ru / server config | Rack/server behavior | entrypoint/server phase |

Environment configuration usually wins because it is evaluated later. config.load_defaults selects versioned framework defaults; it changes behavior and is not cosmetic.

### Initializers and lifecycle hooks

An initializer is a named unit of boot work with optional before: and after: dependencies. Rails performs a topological ordering. Cycles are errors, not a reason to rely on filenames.

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

after_initialize does not guarantee a listening socket, a worker fork, database reachability, or first request.

### Reloading and to_prepare

Rails’ main Zeitwerk loader owns reloadable app code. In development, after a file change Rails unloads those constants before the next work boundary and installs autoloads again. A long-lived non-reloadable registry that stores a reloadable class now holds a stale class object.

Use Rails.application.reloader.to_prepare for repeat-safe wiring that references reloadable classes. It runs on boot and after reload. It must be idempotent: assignment/replacement is good; appending duplicate middleware, subscribers, callbacks, or routes is not.

### Autoloading, eager loading, and once paths

Rails has two important loaders:

- Rails.autoloaders.main: reloadable application and engine code.
- Rails.autoloaders.once: code in autoload_once_paths, which can autoload but is not unloaded.

With config.eager_load = true, Rails loads eager-load paths at boot. Benefits: deploy-time filename/constant validation, no first-use loading latency, and copy-on-write potential. Costs: longer boot and greater initial memory. File order is explicitly undefined; never make a design depend on eager-load order.

### Rack middleware and routes

Rack requires call(env) returning [status, headers, body]. Rails builds a deferred middleware declaration into nested callables around its endpoint. If you add A then B, the result is conceptually A(B(app)); A sees a request first and a response last.

Routes are loaded as part of application setup. The route set supports recognition, URL generation, helpers, mounted engines, and development reloading.

### Executor and reloader

The executor surrounds application work and maintains/clears execution context such as query cache and connection behavior. The reloader decides whether changed code should be unloaded before work. Request/job integrations use these boundaries; hand-created threads need deliberate lifecycle integration.

### Spring, Bootsnap, preload, and forking

- Spring keeps a development process warm for commands. Restart it when diagnosing boot changes.
- Bootsnap caches costly require resolution and often compilation work; it changes speed, not semantic order.
- Puma preload_app!, Unicorn, and Passenger can load in a parent then fork workers. Immutable loaded code may share copy-on-write pages. Open DB connections, sockets, threads, and mutable global state must be reset/re-established in worker hooks.

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

A conventional boot file:

~~~ruby
ENV["BUNDLE_GEMFILE"] ||= File.expand_path("../Gemfile", __dir__)
require "bundler/setup"
require "bootsnap/setup"
~~~

Rails command dispatch varies by release. The architectural point is that Bundler establishes loadability first; an app-aware command then loads the Rails environment and initializes it.

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

rails/all loads standard framework Railties. An intentionally slim API can require only needed components:

~~~ruby
require "rails"
require "active_model/railtie"
require "active_job/railtie"
require "active_record/railtie"
require "action_controller/railtie"
require "action_mailer/railtie"
~~~

When the Application subclass is evaluated, Rails establishes configuration and paths. Rails.application is a singleton application object. Rails.env normally derives from RAILS_ENV, then RACK_ENV, otherwise development.

### Initializer graph mechanics

Railties expose initializer(name, options, &block). Rails gets initializers from each contributor, builds a dependency graph using TSort, and runs a valid ordering.

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

The exact anchor names are version-sensitive: inspect the framework/gem version before binding to one. Prefer a public config hook over an internal initializer name.

Useful phases to articulate:

1. Bootstrap: load paths, Active Support/logging/config basics.
2. Railtie and Engine configuration: components establish paths and register behavior.
3. Application initialization: config/initializers and engine contributions are loaded at the appropriate graph phase.
4. Finish: routes/middleware and final structures complete; after_initialize callbacks and eager loading occur at their configured phases.

Do not win an interview by reciting every initializer name. Explain graph ordering, ownership, and how you inspect the target version.

### Zeitwerk internals

Zeitwerk scans loader roots, maps paths to expected constants, and sets Ruby autoloads. Accessing Payments::Capture causes Ruby/Zeitwerk to load the registered file. Rails manages a reloadable main loader and optional once loader.

Eager load asks each loader to load eligible files. A file app/clients/stripe_client.rb that defines StripeAPI raises a Zeitwerk naming error during eager loading. This is valuable deploy-time validation. It does not authorize reliance on arbitrary load order.

### PostgreSQL and Active Record

Active Record’s Railtie reads database config and creates connection handling/pool configuration. Rails boot does not necessarily open a PostgreSQL connection: checkout is normally lazy when a thread first needs a connection. Any initializer/model class body/health check that runs a query can force a connection.

When checkout occurs, the pg gem performs PostgreSQL protocol authentication/TLS as configured; Active Record wraps an adapter connection and pools it. With roles, replicas, shards, and multiple process types, the connection budget multiplies. Pool is per process and usually per role/shard—not a fleet-global number.

A dangerous scenario: an initializer runs User.count. During a rolling deployment every newly booting web/job process connects while old capacity drains; max_connections may be exhausted. Do not create a startup dependency accidentally.

### Middleware/server handoff

Middleware declarations collect during configuration. Rails later builds/instantiates the nested stack around the endpoint. The Rack handler binds a socket and calls the final Rack callable per request. Development adds reloader/executor integration around work.

### Entrypoint variants

| Entrypoint | Difference |
|---|---|
| bin/rails server | Initializes then starts Rack server; server may preload/fork. |
| config.ru/rackup | Rack host requires environment and runs Rails.application; no bin/rails dispatch necessarily. |
| bin/rails console | Initializes then enters console; no active web request file watcher. |
| bin/rails runner | Initializes, evaluates code, exits; excellent boot smoke test. |
| Rails/Rake task | Loads tasks/app as needed; config.rake_eager_load matters. |
| Sidekiq/worker | Requires Rails env, then has a worker loop and job lifecycle. |
| test | Boots test environment; may intentionally not eager load unless CI asserts it. |

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

Rails bridges Ruby loading/runtime and Rack’s HTTP interface. Railties are the inward plugin architecture; middleware is the outward HTTP composition architecture. An engine contributes to the host app’s unified initialization graph; it does not boot a second app.

---

## 5. Real Production Examples

### Production-only eager-load failure

~~~ruby
# app/clients/stripe_client.rb
class StripeAPI
end
~~~

Lazy development can appear fine if callers use StripeAPI. Production eager loading expects StripeClient and fails before pods are ready. Fix path/constant alignment or a deliberate inflector rule. Do not disable eager load to hide a deploy correctness check.

### Multi-tenant SaaS connection storm

A warmup initializer reads tenants. The app has 12 workers, a job fleet, primary/replica roles, and rolling deploy overlap. Each fresh process opens connections, possibly before fork; PostgreSQL receives a burst while old pods still drain.

Better: boot static tenant configuration only; lazy-load data with bounded cache after readiness or use an explicit warmup with a connection budget. Count pools per process and role, including migrations, consoles, monitors, and deploy overlap.

### Reloadable policy registered once

~~~ruby
# Bad: config/initializers/auth.rb
AuthGateway.policy_class = AccountPolicy
~~~

AccountPolicy is reloadable; AuthGateway may retain the old class after a development reload.

~~~ruby
# Good
Rails.application.reloader.to_prepare do
  AuthGateway.policy_class = AccountPolicy
end
~~~

The callback must replace state rather than append duplicate registrations.

### Preload/fork client hazard

~~~ruby
# Bad in a preloaded parent
PAYMENTS = Faraday.new(url: ENV.fetch("PAYMENTS_URL"))
~~~

The client may retain sockets, mutable state, or threads across fork.

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

Also use framework/server-supported connection lifecycle handling for Active Record.

### Feature flags

~~~ruby
# Bad: hidden deploy-time dependency
FeatureFlags.sync_from_remote!
~~~

Configure the client during boot; refresh lazily or by explicit observable warmup. If policy says no pod may serve without current flags, make that a first-class readiness contract: deadlines, cached last-known-good data, failure policy, metrics, rate-limit/capacity planning, and a runbook.

---

## 6. Common Mistakes

### Junior

- Query/write models or perform remote I/O in an initializer.
- require an autoloadable app file to solve a missing constant.
- Scatter Rails.env conditionals in initializers instead of using environment config.
- Depend on alphabetic initializer filenames.
- Put secrets in code or log configuration/ENV wholesale.

### Mid-level

- Hold reloadable class objects in long-lived global registries.
- Add all lib to autoload/eager-load paths without ignoring tasks, templates, generators, and artifacts.
- Disable production eager loading to shorten boot.
- Insert middleware without verifying the actual stack/order.
- Set database pool to machine-wide traffic concurrency rather than per-process DB concurrency.
- Call Rails.application.initialize! manually from scripts.

### Senior

- Treat Rails/gem private initializer names as a stable public API.
- Preload/fork with inherited sockets, DB connections, threads, file descriptors, or mutable singleton state.
- Couple boot to third-party availability without deadlines, fallback, capacity plan, and alerting.
- Create global registries that require fleet restart to reflect normal configuration change.
- Put expensive allocations/top-level side effects in code that eager loads.

### Staff-level blind spot

Treating startup as a local developer concern. Startup duration, memory, connection fan-out, crash-loop behavior, and external dependency policy affect deploy speed, autoscaling during incidents, and availability. Set a startup SLO and enforce it.

---

## 7. Performance Considerations

### Measure

Track:

- process start to ready-to-serve;
- duration of expensive initializers;
- memory before/after eager load and after fork;
- connection counts created during boot;
- boot/restart failure rate; and
- first-request latency versus startup cost.

time bin/rails runner 'puts :ok' is a coarse baseline. Benchmark the production image, filesystem, environment, and server fork mode.

### Levers

| Lever | Benefit | Trade-off |
|---|---|---|
| Bootsnap | Faster require lookup/compilation cache | Cache troubleshooting; measure actual deploy storage. |
| Eager load | No first-use loading; deploy-time validation; CoW-friendly | Longer boot/more memory; top-level side effects become boot work. |
| preload + fork | Shares immutable pages | Must reset connections/clients/threads post-fork. |
| add_autoload_paths_to_load_path = false | Fewer Ruby path lookups/Bootsnap indexes | Must follow Zeitwerk rather than require app files. |
| Slim framework selection | Less code/middleware | You own omitted component integration. |
| Lazy clients/caches | Faster, less fragile boot | First-use latency; still needs bounded failure behavior. |
| Background warmup | Removes noncritical critical-path work | May stampede dependencies; readiness semantics must be clear. |

### Copy-on-write

A forked child shares parent pages until it writes. Eagerly load immutable code/tables before fork to improve sharing. Per-worker mutations of global hashes, memoized registries, logger buffers, or caches dirty pages. Use PSS/USS, not RSS alone, to judge benefit. Preload is not universally beneficial for one-worker processes or mutable/JIT-heavy workloads.

### Avoid boot I/O

Network calls add DNS, TLS, vendor rate limits, and credential dependency to deploy critical path. If unavoidable: short connect/read deadlines, bounded retry with jitter, explicit fallback or hard-fail policy, structured outcome telemetry, and a readiness contract. Never unbounded-retry in an initializer.

### Pool math

~~~text
maximum possible PostgreSQL connections
≈ web processes × pool per web process
 + job processes × pool per job process
 + migration/console/monitor/deploy headroom
~~~

Roles/shards multiply this. Pool size should reflect concurrent DB users per process, not generic request concurrency. Rolling deploy overlap and initializer queries change demand timing.

---

## 8. Security Considerations

- Boot executes privileged application code. Dependency/supply-chain review matters: a gem runs with app credentials.
- Do not log full ENV, credential objects, database URLs, or tokens; boot exception logs are widely retained.
- Required secrets should fail boot with a precise non-secret message. Optional services need an explicit degraded implementation/status.
- Environment values are untrusted strings at the boundary: parse/validate URLs, hosts, integers, booleans; never eval or shell-interpolate them.
- Protect health/readiness endpoints and do not leak topology or error internals.
- Understand sensitive descriptor/token/client inheritance across fork and rotation behavior.
- Separate build-time from runtime secret requirements. Do not make assets/CI require production credentials unless necessary.

---

## 9. Debugging

### Classify before changing things

| Symptom | Likely phase | First move |
|---|---|---|
| cannot load such file before app boot | Bundler/load path/Bootsnap | Check Ruby/Bundler/Gemfile.lock, bundle exec, cache. |
| Error in config/application.rb | config evaluation | Read first app stack frame; remove premature constant/I/O. |
| Error in config/initializers | initializer graph | Identify block, ordering, I/O, and reproduce with runner. |
| Zeitwerk::NameError in CI/prod | eager load | Run zeitwerk:check; fix name/path/inflector. |
| Works until source edit | reloader/stale class | Inspect to_prepare and cached class references. |
| Pods slow/not ready | I/O/eager work/pool | Time phases; count connections/dependencies. |
| rails s works, server host fails | entrypoint/preload/fork | Compare command, env, server hooks. |

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

Use safe non-production secrets/config when simulating production. Avoid a local production command that contacts real production services.

### Inspect from console

~~~ruby
Rails.application.config.eager_load
Rails.application.config.autoload_paths
Rails.application.config.eager_load_paths
Rails.autoloaders.main.dirs
Rails.autoloaders.once.dirs
Rails.application.middleware.map(&:klass)
~~~

Middleware entry shape is version-dependent; do not build untested production tooling on this exact snippet.

### Time initialization

~~~ruby
# config/initializers/boot_timing.rb
started_at = Process.clock_gettime(Process::CLOCK_MONOTONIC)

Rails.application.config.after_initialize do
  elapsed = Process.clock_gettime(Process::CLOCK_MONOTONIC) - started_at
  Rails.logger.info(event: "rails.initialized", elapsed_seconds: elapsed.round(3))
end
~~~

For an individual block, use monotonic timing or ActiveSupport::Notifications. Log duration/outcome—not secrets. Better diagnostics preserve behavior; do not introduce query/network work merely to inspect it.

### Autoload diagnostics

~~~ruby
# temporary, noisy diagnostic
Rails.autoloaders.log!

# after Rails.logger exists
Rails.autoloaders.logger = Rails.logger
~~~

Then inspect expected constant names, acronym inflections, ignored/collapsed paths, and manual requires. zeitwerk:check is the best CI guardrail.

### Incident procedure

1. Capture full exception, first app frame, command, versions, RAILS_ENV, server mode, revision.
2. Reproduce with smallest matching entrypoint: runner for Rails init, real server for preload/fork behavior.
3. Isolate application config, suspect initializer, eager load, then worker boot using timestamps.
4. For external failures, verify DNS, credentials, firewall, quotas, and concurrent process starts from the deploy environment.
5. Fix lifecycle ownership; add regression coverage; remove temporary diagnostics.

After Gemfile, boot config, application config, or initializer edits, restart Spring/server. If evidence suggests stale cache, clear the project-specific Bootsnap cache using the project’s safe documented procedure.

---

## 10. Best Practices

1. Make boot deterministic, fast, idempotent, and observable.
2. Put global declarative defaults in application.rb and environment deltas in environment files.
3. Keep initializers thin. Move business logic into ordinary testable objects.
4. Avoid DB writes/queries, remote I/O, long work, background threads, and irreversible side effects.
5. Run production-like eager-loading checks in CI.
6. Follow Zeitwerk naming; configure inflections; never require reloadable app code.
7. Use to_prepare only for idempotent reload-aware wiring.
8. Define pre-fork/post-fork ownership for every connection/client/thread.
9. Publish boot duration, memory, and connection budgets.
10. Test with the actual production server/process model, not only rails server locally.

---

## 11. Anti-patterns

### Initializer junk drawer

~~~ruby
# config/initializers/setup_everything.rb
User.find_each { |u| SearchIndex.sync(u) }
Faraday.get(ENV.fetch("CONFIG_URL"))
Thread.new { loop { Metrics.flush } }
~~~

Every command now needs DB/vendor availability; work repeats on every process boot; thread/fork/shutdown behavior is undefined. Use migrations/jobs/deployment tasks, explicit client policies, and supervised worker architecture.

### Top-level side effects in autoloaded code

~~~ruby
# app/services/rates.rb
Rates.refresh!
class Rates; end
~~~

This executes when the file autoloads, eager loads, or reloads. File evaluation is not a lifecycle hook and eager-load order is undefined.

### Filename ordering

Files named 01_config.rb and 99_patch.rb encode a hidden dependency graph. Prefer a public config API or documented named dependency when unavoidable.

### Global singleton with hidden lifecycle

~~~ruby
CLIENT = Vendor::Client.new(token: Rails.application.credentials.vendor_token)
~~~

It is hard to rotate, test, fork, reload, or configure per tenant. Prefer a narrow factory/injected config with explicit reset semantics.

### Rescuing into a half-initialized app

~~~ruby
begin
  CriticalService.configure!
rescue StandardError => e
  Rails.logger.error(e.message)
end
~~~

For truly critical setup, fail fast. For optional setup, install an intentional no-op/degraded path and expose its status. A half-initialized process serving traffic is worse than a clear boot failure.

---

## 12. Interview Questions

### Basic

1. **What runs after bin/rails server?** Ruby loads boot.rb, Bundler/Bootsnap, command dispatch, app/environment, Rails.application.initialize!, then Rack server runtime.
2. **boot.rb vs environment.rb?** boot.rb sets dependency-loading/cache groundwork. environment.rb loads application and initializes it.
3. **What is an initializer?** A named dependency-orderable unit of framework/app/engine boot setup.
4. **Why eager-load production?** Earlier correctness failure, no first constant latency, and CoW potential.

### Intermediate

5. **How do engines participate?** They are Railties with paths/config/routes; their initializers join host graph and paths join loader management.
6. **Why can one-shot setup fail after development reload?** It retained a class object that Rails unloaded/redefined; use idempotent to_prepare/lazy name resolution.
7. **Does boot always connect PostgreSQL?** No: pool configuration is created, connection is commonly lazy, but initializer/model code may force checkout.
8. **What does Bootsnap change?** Startup speed through caching, not lifecycle semantics.
9. **How does middleware order work?** Nested Rack wrappers; outer layer sees request first and response last. Inspect actual stack.

### Senior

10. **How reduce a 45-second boot?** Measure; remove I/O/data work; defer noncritical clients; inspect eager-load side effects/require cost; test Bootsnap/preload; validate production memory/first-request trade-offs.
11. **preload_app! vs eager_load?** Parent-before-fork server lifecycle versus constant-loading policy. Combined CoW needs post-fork reconnect/reset.
12. **Production eager-load error but dev works?** Reproduce production config, run zeitwerk:check, repair name/path/inflector/manual-require issue, keep check in CI.
13. **A gem needs an app class in config?** Use its public API; if class reloads, assign idempotently in to_prepare or use class name/lazy resolution.

### Staff

14. **Startup strategy for 200 services?** Shared startup SLO, no undocumented boot I/O, eager-load CI, phase telemetry, standard pre/post-fork hooks, clear readiness contracts, rollout connection math, templates/linting.
15. **Feature flags must be current at startup?** First define safety requirement. If mandatory, explicit readiness dependency with deadline, cached data, fallback, capacity and runbook. If not, decouple with bounded refresh. Never hide it in an initializer.
16. **How evolve initializer contracts across engines?** Expose public config/hooks, minimize private anchors/global mutation, make idempotent, test host combinations/reload, version/document contracts.

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

This supports environment overrides and test injection without performing work during boot. Parse invalid environment values early and clearly.

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

The integration owns its setup and exposes config. Verify private-looking anchors against your target Rails and document/test them if publishing a gem.

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

For immutable configuration store an identifier in config. Create required data through migrations/seeds/deploy orchestration; validate it through targeted health policy, not every boot.

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

This removes socket construction from boot. It still needs error handling, idempotency-aware retry, metrics, and post-fork reset policy.

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

Or use a dedicated CI job:

~~~bash
RAILS_ENV=production SECRET_KEY_BASE=dummy bin/rails zeitwerk:check
~~~

Use safe dummy configuration only for values needed to load code; never point CI boot at production services.

### G. Explain middleware placement

~~~ruby
# config/application.rb
config.middleware.insert_before Rack::Head, RequestCorrelationId
~~~

State the reason: a correlation ID needs to be available early for downstream logs and returned on the response. Validate with bin/rails middleware because the stack varies by version, mode, and gems.

---

## 14. Edge Cases

| Edge case | Why it surprises teams | Correct response |
|---|---|---|
| runner boots, server fails | socket binding/server config/preload occurs later | Test real server command and hooks. |
| config.ru differs from bin/rails | host/command/env/config differs | Compare command args, env, directory, server config. |
| console sees old code | no web request file-watcher loop | Reload/restart deliberately. |
| Worker starts before migrations | app boots against old schema | Coordinate releases with expand-contract migrations. |
| STI subclasses missing in dev | lazy load cannot enumerate unknown hierarchy | Eager load/preload hierarchy with documented pattern. |
| Acronym only fails CI | file maps to VatReport but defines VATReport | Configure inflector or rename; run zeitwerk:check. |
| setup runs twice | to_prepare/reload/test/manual init | Make setup idempotent. |
| lib/tasks defines constants | broad lib autoload makes task files loader inputs | autoload_lib with ignores or separate runtime code. |
| Rails.logger unavailable | logging init is later | Move code late or use appropriate early logger. |
| credentials absent at build | build and run contracts differ | Separate requirements; avoid runtime-secret load at build. |
| SIGTERM during slow boot | orchestrator kills before ready | Short/idempotent boot; no partial side effects. |
| connection inherited after fork | parent queried pre-fork | reset/reconnect in supported worker hook. |
| circular initializer graph | no valid ordering | Remove coupling/create clear phase. |

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

- Boot makes a Ruby process ready to execute a Rails application; it is separate from request handling.
- Typical path: bin/rails -> boot.rb -> Rails command/Rack host -> application/environment -> initialize! -> middleware/routes/autoloaders -> server runtime.
- Railties/engines contribute configuration and dependency-ordered initializers.
- Put declarative settings in app/environment config; keep initializers thin, fast, and side-effect-free.
- Zeitwerk owns app code. Match paths/constants, avoid manual require, and test eager loading in CI.
- Eager load gives deploy-time validation and CoW potential but costs time/memory; its order is undefined.
- to_prepare runs at boot and reload boundaries. Make it idempotent.
- Active Record configures pools during boot; code may force real PostgreSQL connections. Count connections during rollout.
- Preload/fork differs from eager load. Reset/reconnect clients, pools, and threads after fork.
- Startup duration, memory, connection fan-out, and dependency policy are reliability concerns.

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
- [Rails Application Bootstrap](https://github.com/rails/rails/blob/main/railties/lib/rails/application/bootstrap.rb) and [Finisher](https://github.com/rails/rails/blob/main/railties/lib/rails/application/finisher.rb).
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

This turns memorized terminology into an architecture you can reason about under interview pressure.
