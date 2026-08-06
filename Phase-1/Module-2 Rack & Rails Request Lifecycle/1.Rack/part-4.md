# Part 4 — Internals, Concurrency, and Streaming

Welcome back.

In Part 1, you learned the Rack contract: `call(env) → [status, headers, body]`.

In Part 2, you learned the helper tools and middleware fundamentals.

In Part 3, you learned how Rails uses Rack — ActionDispatch, the full middleware stack, ordering, custom middleware, and production patterns.

Now we go deeper. Much deeper.

Part 4 answers questions like:

> "What does Rack's source code actually look like?"
> "How do multiple requests run at the same time?"
> "What happens if two threads touch the same middleware?"
> "Can Rails send data to the browser before the response is done?"
> "What is socket hijacking?"

These are the questions that separate a senior engineer from someone who just uses Rails.

---

# What You'll Learn in Part 4

1. **Internal Implementation** — how Rack is built inside
2. **Rack Source Code Walkthrough** — reading real Rack code, line by line
3. **Thread Safety** — what breaks when multiple threads share middleware
4. **Concurrency** — how Puma handles many requests at once
5. **Streaming** — sending response data piece by piece
6. **Hijacking** — taking over the raw TCP socket from Rack
7. **Async** — non-blocking responses
8. **HTTP/2 Considerations** — how Rack relates to HTTP/2

---

# 1. Internal Implementation

## How Big is Rack?

Rack is surprisingly small.

The entire Rack gem is about **50 files**. The core is maybe **10 files**. The rest are built-in middleware and utilities.

Compare that to Rails, which has thousands of files.

Rack is intentionally minimal. It defines a contract and provides helpers. That's it.

---

## The Key Files

Here are the most important files inside the Rack gem:

```
rack/
├── lib/
│   └── rack/
│       ├── builder.rb        ← Rack::Builder (use, run, map)
│       ├── request.rb        ← Rack::Request
│       ├── response.rb       ← Rack::Response
│       ├── utils.rb          ← Utility methods (parsing, escaping)
│       ├── mime_type.rb      ← MIME type detection
│       ├── mock_request.rb   ← For testing
│       ├── mock_response.rb  ← For testing
│       ├── common_logger.rb  ← Rack::CommonLogger middleware
│       ├── content_length.rb ← Rack::ContentLength middleware
│       ├── deflater.rb       ← Rack::Deflater middleware
│       ├── etag.rb           ← Rack::ETag middleware
│       ├── head.rb           ← Rack::Head middleware
│       ├── runtime.rb        ← Rack::Runtime middleware
│       ├── sendfile.rb       ← Rack::Sendfile middleware
│       ├── show_exceptions.rb← Rack::ShowExceptions middleware
│       ├── static.rb         ← Rack::Static middleware
│       ├── method_override.rb← Rack::MethodOverride middleware
│       ├── tempfile_reaper.rb← Rack::TempfileReaper middleware
│       └── lint.rb           ← Rack::Lint (validates your app)
```

That's most of Rack. Let's look inside the important ones.

---

## Rack's Core Architecture

Rack has three layers:

```
Layer 1: The Specification
  - Just a set of rules (not really code)
  - "call(env) must return [status, headers, body]"

Layer 2: Helper Classes
  - Rack::Request   (read env easily)
  - Rack::Response  (build responses easily)
  - Rack::Builder   (compose middleware)
  - Rack::Utils     (parsing utilities)

Layer 3: Built-in Middleware
  - Rack::CommonLogger
  - Rack::ContentLength
  - Rack::Runtime
  - Rack::ETag
  - ... and more
```

Layer 1 is just a document. Layer 2 and 3 are actual Ruby code.

---

## How Rack Boots Up

When you run `rackup` or start Puma, here's what happens:

```
1. Server reads config.ru
2. config.ru is executed inside a Rack::Builder context
3. Rack::Builder processes use/run/map directives
4. Rack::Builder builds the middleware chain (inside-out)
5. Server gets back one Rack app object
6. Server starts listening for HTTP connections
7. For each request: server builds env, calls app.call(env)
```

Let's look at step 3-4 in detail. That's `Rack::Builder`.

---

# 2. Rack Source Code Walkthrough

## Rack::Builder — The Real Code

This is simplified from the actual source, but captures the important parts:

```ruby
module Rack
  class Builder
    def initialize(&block)
      @use = []      # list of middleware classes
      @run = nil     # the inner app
      @map = nil     # path mappings

      instance_eval(&block) if block
    end

    def use(middleware, *args, &block)
      @use << [middleware, args, block]
    end

    def run(app)
      @run = app
    end

    def map(path, &block)
      @map ||= {}
      @map[path] = block
    end

    def to_app
      app = @run

      # Build inside-out: last use wraps the app first
      @use.reverse_each do |middleware_class, args, block|
        app = middleware_class.new(app, *args, &block)
      end

      app
    end
  end
end
```

Let's break this down.

---

### `initialize`

```ruby
def initialize(&block)
  @use = []
  @run = nil
  instance_eval(&block) if block
end
```

When you write:

```ruby
Rack::Builder.new do
  use Logger
  use Auth
  run MyApp.new
end
```

The block runs inside the Builder instance. So `use` and `run` are method calls on the builder.

`instance_eval(&block)` is the trick. It makes `self` inside the block be the Builder object. That's why `use` and `run` work without `builder.use`, `builder.run`.

---

### `use`

```ruby
def use(middleware, *args, &block)
  @use << [middleware, args, block]
end
```

Each `use` call stores the middleware class and any arguments. It doesn't create the middleware yet.

```ruby
use Logger              # @use = [[Logger, [], nil]]
use Auth, secret: "x"   # @use = [[Logger, [], nil], [Auth, [{secret: "x"}], nil]]
```

Just collecting. Building happens later.

---

### `run`

```ruby
def run(app)
  @run = app
end
```

Stores the innermost app. Nothing fancy.

---

### `to_app` — Where the Magic Happens

```ruby
def to_app
  app = @run

  @use.reverse_each do |middleware_class, args, block|
    app = middleware_class.new(app, *args, &block)
  end

  app
end
```

This is the heart of Rack.

Let's trace through an example:

```ruby
Rack::Builder.new do
  use A
  use B
  use C
  run MyApp.new
end
```

`@use` is: `[[A, [], nil], [B, [], nil], [C, [], nil]]`

`@run` is: `MyApp.new`

Now `to_app` runs:

```
Step 0: app = MyApp.new

reverse_each goes C, B, A:

Step 1: app = C.new(MyApp.new)
Step 2: app = B.new(C.new(MyApp.new))
Step 3: app = A.new(B.new(C.new(MyApp.new)))
```

Final result:

```
A wraps B wraps C wraps MyApp
```

When the server calls `app.call(env)`, it hits A first.

A calls `@app.call(env)` → hits B.

B calls `@app.call(env)` → hits C.

C calls `@app.call(env)` → hits MyApp.

**Why `reverse_each`?**

Because the first `use` should be the outermost layer. If we iterated forward, the last `use` would be outermost. Reversing fixes the order.

Think of it like wrapping presents:

```
You wrap the gift (MyApp) first.
Then wrap that in paper C.
Then wrap that in paper B.
Then wrap that in paper A.

When someone opens it, they see A first.
```

---

## Rack::Request — The Real Code

Simplified version:

```ruby
module Rack
  class Request
    def initialize(env)
      @env = env
    end

    def request_method
      @env["REQUEST_METHOD"]
    end

    def path_info
      @env["PATH_INFO"]
    end

    def path
      path_info
    end

    def query_string
      @env["QUERY_STRING"]
    end

    def get?
      request_method == "GET"
    end

    def post?
      request_method == "POST"
    end

    def params
      # Merge query params and form body params
      GET.merge(POST)
    end

    def GET
      # Parse query string: "page=2&sort=name" → {"page" => "2", "sort" => "name"}
      Rack::Utils.parse_query(query_string)
    end

    def POST
      # Parse form body if content type is form data
      # Returns {} for JSON bodies
      Rack::Utils.parse_query(body.read).tap { body.rewind }
    end

    def body
      @env["rack.input"]
    end

    def cookies
      Rack::Utils.parse_cookies(@env)
    end

    def ip
      @env["REMOTE_ADDR"]
    end

    def user_agent
      @env["HTTP_USER_AGENT"]
    end

    def env
      @env
    end
  end
end
```

The key insight: **every method just reads from `@env`**.

`Rack::Request` is not magic. It's a thin wrapper that makes `env` hash keys easier to access.

When you call `request.path`, it's the same as `env["PATH_INFO"]`.

When you call `request.get?`, it's the same as `env["REQUEST_METHOD"] == "GET"`.

---

## Rack::Response — The Real Code

Simplified:

```ruby
module Rack
  class Response
    attr_accessor :status
    attr_reader :headers, :body

    def initialize(body = nil, status = 200, headers = {})
      @status = status
      @headers = headers
      @body = []
      @body << body if body
    end

    def [](key)
      @headers[key]
    end

    def []=(key, value)
      @headers[key] = value
    end

    def write(chunk)
      @body << chunk
      @headers["Content-Length"] = @body.sum(&:bytesize).to_s
      chunk.bytesize
    end

    def set_cookie(name, value)
      cookie_header = Rack::Utils.set_cookie_header(name, value)
      # Append to Set-Cookie header
      add_header("set-cookie", cookie_header)
    end

    def redirect(target, status = 302)
      @status = status
      @headers["location"] = target
    end

    def finish
      [@status, @headers, @body]
    end
  end
end
```

Key insight: `finish` just returns the standard `[status, headers, body]` array.

Everything else — `write`, `set_cookie`, `redirect` — is just helpers that build up `@status`, `@headers`, and `@body` so you don't have to construct the array by hand.

---

## Rack::Utils — Parsing Helpers

`Rack::Utils` does the grunt work:

```ruby
module Rack
  module Utils
    # Parse query strings
    def self.parse_query(query_string)
      # "page=2&sort=name" → {"page" => "2", "sort" => "name"}
      params = {}
      (query_string || "").split("&").each do |pair|
        key, value = pair.split("=", 2)
        params[unescape(key)] = unescape(value || "")
      end
      params
    end

    # URL encoding/decoding
    def self.escape(s)
      URI.encode_www_form_component(s)
    end

    def self.unescape(s)
      URI.decode_www_form_component(s)
    end

    # Parse cookies
    def self.parse_cookies(env)
      # "session=abc; theme=dark" → {"session" => "abc", "theme" => "dark"}
      cookie_header = env["HTTP_COOKIE"] || ""
      cookies = {}
      cookie_header.split(";").each do |pair|
        key, value = pair.strip.split("=", 2)
        cookies[key] = value
      end
      cookies
    end

    # Build Set-Cookie header
    def self.set_cookie_header(name, value)
      if value.is_a?(Hash)
        cookie = "#{name}=#{value[:value]}"
        cookie << "; path=#{value[:path]}" if value[:path]
        cookie << "; HttpOnly" if value[:httponly]
        cookie << "; Secure" if value[:secure]
        cookie
      else
        "#{name}=#{value}"
      end
    end
  end
end
```

Nothing mysterious. String parsing, URL encoding, cookie formatting.

The reason this lives in `Utils` is so both `Rack::Request` and `Rack::Response` can use the same parsing code.

---

## Rack::Lint — The Spec Enforcer

`Rack::Lint` is a middleware that validates your app follows the Rack spec:

```ruby
module Rack
  class Lint
    def initialize(app)
      @app = app
    end

    def call(env)
      # Check env is a Hash
      raise "env must be a Hash" unless env.is_a?(Hash)

      # Check required keys
      %w[REQUEST_METHOD PATH_INFO SERVER_NAME SERVER_PORT].each do |key|
        raise "Missing #{key}" unless env.key?(key)
      end

      # Check rack.input
      raise "rack.input missing" unless env["rack.input"]

      status, headers, body = @app.call(env)

      # Check status is an integer
      raise "Status must be Integer" unless status.is_a?(Integer)
      raise "Status must be >= 100" unless status >= 100

      # Check headers
      raise "Headers must be a Hash" unless headers.is_a?(Hash)

      # Check body responds to each
      raise "Body must respond to each" unless body.respond_to?(:each)

      [status, headers, body]
    end
  end
end
```

Use it during development to catch spec violations:

```ruby
# config.ru
use Rack::Lint   # catches mistakes early
run MyApp.new
```

If your app returns a string instead of an array for the body:

```ruby
[200, {}, "Hello"]   # WRONG — string, not array
```

Rack::Lint catches it and raises a clear error.

**Don't use in production.** It adds overhead because it checks every single request.

---

## Interview Questions — Internal Implementation

**Q: How does Rack::Builder build the middleware chain?**
A: It collects middleware classes with `use`, stores the app with `run`, then in `to_app` it iterates in reverse, wrapping each middleware around the previous layer. The result is one nested app object.

**Q: Why does Rack::Builder iterate in reverse?**
A: Because the first `use` should be the outermost middleware. Reverse iteration ensures the first declared middleware wraps everything else.

**Q: What is Rack::Lint?**
A: A development middleware that validates your app follows the Rack spec. It checks that `env` has required keys, that status is an integer, that headers are a hash, and that body responds to `each`.

**Q: Is Rack::Request a separate object from env?**
A: No. It wraps the same `env` hash. Every method on `Rack::Request` just reads from `env`. No copying.

---

# 3. Thread Safety

## Why Thread Safety Matters

In production, Puma runs multiple threads. Each thread handles a different request at the same time.

```
Puma
├── Thread 1 → handling GET /users     (User A)
├── Thread 2 → handling POST /orders   (User B)
├── Thread 3 → handling GET /products  (User C)
├── Thread 4 → handling DELETE /cart    (User D)
└── Thread 5 → handling GET /users     (User E)
```

All 5 threads share the **same middleware objects**.

Read that again.

The **same** middleware instance handles requests from different users at the same time.

---

## The Problem

Consider this middleware:

```ruby
class BadMiddleware
  def initialize(app)
    @app = app
    @request_count = 0   # instance variable shared across threads!
  end

  def call(env)
    @request_count += 1
    env["request_number"] = @request_count
    @app.call(env)
  end
end
```

What happens when two threads run `@request_count += 1` at the exact same time?

```
Thread 1: reads @request_count → 5
Thread 2: reads @request_count → 5    (same value!)
Thread 1: writes @request_count = 6
Thread 2: writes @request_count = 6   (should be 7!)
```

This is a **race condition**. Two threads read the same value before either writes. One increment is lost.

The count is now 6, but it should be 7.

---

## The Golden Rule of Thread-Safe Middleware

> **Never store per-request state in instance variables.**

Instance variables (`@something`) are shared across all threads. If you write to them during a request, other threads will see (and overwrite) those values.

---

## What IS Safe

### 1. Read-only instance variables (set once in initialize)

```ruby
class SafeMiddleware
  def initialize(app, header_name: "X-Custom")
    @app = app
    @header_name = header_name   # set once, never changes
  end

  def call(env)
    status, headers, body = @app.call(env)
    headers[@header_name] = "some-value"
    [status, headers, body]
  end
end
```

`@app` and `@header_name` are set in `initialize` and never change. Multiple threads can read them safely.

### 2. Local variables inside call

```ruby
class SafeMiddleware
  def initialize(app)
    @app = app
  end

  def call(env)
    start_time = Time.now     # local variable — each thread has its own
    status, headers, body = @app.call(env)
    duration = Time.now - start_time
    headers["X-Runtime"] = duration.to_s
    [status, headers, body]
  end
end
```

`start_time` and `duration` are local variables. Each thread has its own copy. They don't interfere with each other.

### 3. The env hash

```ruby
def call(env)
  env["my_app.start_time"] = Time.now   # safe — each request has its own env
  @app.call(env)
end
```

Each request gets its own `env` hash. Writing to `env` is safe.

---

## What is NOT Safe

### 1. Writing to instance variables during call

```ruby
# DANGEROUS
def call(env)
  @current_user = find_user(env)   # Thread A sets User A
                                    # Thread B overwrites with User B
                                    # Thread A now sees User B!
  @app.call(env)
end
```

### 2. Mutating shared data structures

```ruby
# DANGEROUS
class BadMiddleware
  def initialize(app)
    @app = app
    @cache = {}   # shared hash
  end

  def call(env)
    path = env["PATH_INFO"]
    @cache[path] = Time.now   # multiple threads write to the same hash
    @app.call(env)
  end
end
```

Ruby's Hash is not thread-safe for concurrent writes. You can get corrupted data, lost entries, or even crashes.

### 3. Lazy initialization without locks

```ruby
# DANGEROUS
class BadMiddleware
  def initialize(app)
    @app = app
  end

  def call(env)
    @db ||= Database.connect   # race condition!
    # Both threads might create separate connections
    @app.call(env)
  end
end
```

Two threads might both see `@db` as nil and both create connections. Or worse, one thread might see a half-initialized object.

---

## How to Fix Thread Safety Issues

### Fix 1: Use local variables

```ruby
# SAFE
def call(env)
  current_user = find_user(env)   # local variable, thread-local
  env["current_user"] = current_user
  @app.call(env)
end
```

### Fix 2: Use env for per-request state

```ruby
# SAFE
def call(env)
  env["my_app.start_time"] = Process.clock_gettime(Process::CLOCK_MONOTONIC)
  status, headers, body = @app.call(env)
  duration = Process.clock_gettime(Process::CLOCK_MONOTONIC) - env["my_app.start_time"]
  headers["X-Duration"] = duration.to_s
  [status, headers, body]
end
```

### Fix 3: Use Mutex for shared mutable state

```ruby
class ThreadSafeCounter
  def initialize(app)
    @app = app
    @mutex = Mutex.new
    @count = 0
  end

  def call(env)
    @mutex.synchronize do
      @count += 1
    end

    env["request_number"] = @count
    @app.call(env)
  end
end
```

A `Mutex` ensures only one thread can run the code inside `synchronize` at a time.

But be careful — mutexes make requests wait for each other. Too many mutexes = slow app.

### Fix 4: Use Concurrent::AtomicFixnum (from concurrent-ruby)

```ruby
require "concurrent"

class AtomicCounter
  def initialize(app)
    @app = app
    @count = Concurrent::AtomicFixnum.new(0)
  end

  def call(env)
    env["request_number"] = @count.increment
    @app.call(env)
  end
end
```

`Concurrent::AtomicFixnum` is thread-safe without explicit locks. It uses CPU-level atomic operations. Faster than Mutex for simple counters.

### Fix 5: Use Thread.current for thread-local storage

```ruby
def call(env)
  Thread.current[:request_id] = SecureRandom.uuid
  status, headers, body = @app.call(env)
  Thread.current[:request_id] = nil   # clean up!
  [status, headers, body]
end
```

`Thread.current` is a hash that belongs to the current thread only. Other threads can't see it.

**Warning:** Always clean up `Thread.current` values after the request. Puma reuses threads, so leftover data from one request can leak to the next.

---

## Thread Safety Checklist

| Pattern | Safe? | Why |
|---------|-------|-----|
| `@app` in middleware | ✅ | Set once in initialize, never changes |
| `@config` set in initialize | ✅ | Read-only after initialize |
| Local variable in `call` | ✅ | Each thread has its own stack |
| Writing to `env` hash | ✅ | Each request has its own env |
| Writing to `@cache` in `call` | ❌ | Shared across threads |
| `@current_user = ...` in `call` | ❌ | Overwritten by other threads |
| `@count += 1` in `call` | ❌ | Race condition |
| `Mutex.synchronize { @count += 1 }` | ✅ | Mutex prevents races |
| `Thread.current[:x] = ...` | ✅* | Safe, but clean up after request |

---

## How Rails Handles Thread Safety

Rails itself is thread-safe (since Rails 4). But there are rules:

1. **Controllers are created per-request.** Each request gets a new controller instance. So `@user` in a controller is safe — it's not shared.

2. **Middleware is shared.** One instance handles all requests. That's why middleware must be thread-safe.

3. **ActiveRecord connections are managed per-thread.** The connection pool gives each thread its own database connection.

4. **Global state is your responsibility.** If you write to a class variable or global variable, it's shared across all threads.

```ruby
# DANGEROUS in threaded Puma
class UsersController < ApplicationController
  @@request_count = 0   # class variable shared across all threads!

  def index
    @@request_count += 1   # race condition
  end
end
```

---

## Interview Questions — Thread Safety

**Q: Are Rack middleware instances shared across threads?**
A: Yes. Puma creates middleware instances once at boot. All threads share the same instances. That's why you must not store per-request state in instance variables.

**Q: What's safe to store in middleware instance variables?**
A: Only read-only data set during `initialize` (config, the inner app). Never write to instance variables during `call`.

**Q: How do you store per-request data in middleware?**
A: Use local variables inside `call`, or write to the `env` hash. Both are safe because each request has its own.

**Q: What's a race condition?**
A: When two threads read and write shared data at the same time, producing incorrect results. Example: two threads increment a counter and one increment is lost.

**Q: How do you safely share mutable state in middleware?**
A: Use a `Mutex` to ensure only one thread accesses the data at a time. Or use thread-safe data structures from `concurrent-ruby`.

---

# 4. Concurrency

## How Puma Handles Multiple Requests

Puma uses a **multi-process, multi-thread** model:

```
Puma
├── Worker Process 1
│   ├── Thread 1 → handling request
│   ├── Thread 2 → handling request
│   ├── Thread 3 → handling request
│   ├── Thread 4 → idle (waiting for request)
│   └── Thread 5 → handling request
│
├── Worker Process 2
│   ├── Thread 1 → handling request
│   ├── Thread 2 → idle
│   ├── Thread 3 → handling request
│   ├── Thread 4 → handling request
│   └── Thread 5 → idle
│
└── Worker Process 3
    ├── Thread 1 → handling request
    ├── Thread 2 → handling request
    ├── Thread 3 → idle
    ├── Thread 4 → handling request
    └── Thread 5 → handling request
```

Default Puma config:

```ruby
# config/puma.rb
workers 3        # 3 processes
threads 5, 5     # 5 threads per process
```

Total concurrent requests:

```
3 workers × 5 threads = 15 requests at the same time
```

---

## Process vs Thread — What's the Difference?

### Processes

```
Process 1                Process 2
┌─────────────┐          ┌─────────────┐
│ Own memory  │          │ Own memory  │
│ Own env     │          │ Own env     │
│ Own code    │          │ Own code    │
│ Own objects │          │ Own objects │
└─────────────┘          └─────────────┘
      ↕                        ↕
  Completely                Completely
  independent              independent
```

Processes don't share memory. If Process 1 crashes, Process 2 keeps running.

### Threads

```
Process 1
┌──────────────────────────────────┐
│  Thread 1    Thread 2    Thread 3│
│     │           │           │   │
│     └───────────┼───────────┘   │
│           Shared Memory         │
│         (same objects,          │
│          same middleware,       │
│          same classes)          │
└──────────────────────────────────┘
```

Threads within the same process share memory. That's why thread safety matters — threads can see and modify each other's data.

---

## The GVL (Global VM Lock) — Ruby's Constraint

Ruby (MRI/CRuby) has the **Global VM Lock** (historically called GIL — Global Interpreter Lock).

The GVL means:

> **Only one thread in a Ruby process can execute Ruby code at a time.**

Wait — then what's the point of threads?

### The Point: I/O Waiting

When a thread is waiting for I/O (database query, HTTP request, file read), the GVL is released. Other threads can run Ruby code while one thread waits.

```
Thread 1: Running Ruby code ████████
Thread 2:                            Waiting for DB ░░░░░░░  Running Ruby ███
Thread 3:                   Waiting for API ░░░░░░░░░░░░░░░  Running Ruby ██
```

During I/O wait (`░░░`), the GVL is free. Other threads can use it.

### Why This Matters for Web Apps

Most web requests spend their time waiting:

```
Typical request breakdown:
  5%  - Ruby code (parsing, logic)
  50% - Database queries (waiting for PostgreSQL)
  30% - External API calls (waiting for Stripe, etc.)
  15% - Other I/O (Redis, file reads)
```

So even with the GVL, threads give you a big boost because most time is spent in I/O.

---

## How a Request Flows Through Puma's Concurrency Model

```
1. Client connects to Puma

2. Puma's main thread accepts the connection

3. Puma assigns the connection to a worker process

4. Worker's thread pool picks an available thread

5. That thread runs:
   app.call(env)
   
6. The thread processes the entire middleware chain + app

7. When the app does a DB query:
   - Thread releases GVL
   - Waits for PostgreSQL
   - Other threads run Ruby code
   - PostgreSQL responds
   - Thread reacquires GVL
   - Continues processing

8. Thread returns [status, headers, body]

9. Puma sends the HTTP response

10. Thread becomes available for the next request
```

---

## Rack's Role in Concurrency

Rack itself doesn't manage concurrency. It doesn't create threads or processes.

Rack just defines the interface: `call(env) → [status, headers, body]`.

The **server** (Puma, Unicorn, Falcon) decides how to handle concurrency:

| Server | Model | How |
|--------|-------|-----|
| **Puma** | Multi-process + multi-thread | Workers × Threads |
| **Unicorn** | Multi-process only | Workers (no threads) |
| **Falcon** | Async (fibers) | Event loop + fibers |
| **Thin** | Event loop (single thread) | EventMachine |

Your Rack app doesn't need to know which model the server uses. It just implements `call(env)` and the server handles the rest.

That's the beauty of Rack's design — the contract is so simple that it works with any concurrency model.

---

## Concurrency and the Middleware Chain

Here's what's happening inside Puma when 3 requests arrive:

```
Thread 1                    Thread 2                    Thread 3
─────────                   ─────────                   ─────────
Middleware A.call           Middleware A.call           
  Middleware B.call           Middleware B.call         Middleware A.call
    App.call                    App.call                 Middleware B.call
    (DB query - waiting)        (processing)               App.call
    ...                         return [200, ...]          (processing)
    (DB responds)             Middleware B (after)         return [200, ...]
    return [200, ...]         Middleware A (after)        Middleware B (after)
  Middleware B (after)      → Response sent              Middleware A (after)
  Middleware A (after)                                  → Response sent
→ Response sent
```

Notice:

- Each thread runs the **same** middleware objects (A and B)
- But each thread has its own `env` hash and local variables
- Threads can be at different stages simultaneously
- When Thread 1 waits for DB, Threads 2 and 3 can run

---

## Connection Pool — How ActiveRecord Handles Concurrency

Each thread needs its own database connection. But creating a new connection for every thread is expensive.

Solution: **connection pool**.

```ruby
# config/database.yml
production:
  pool: 10   # keep 10 connections ready
```

```
Thread 1: "I need a DB connection"
  → Pool gives Thread 1 Connection A
  
Thread 2: "I need a DB connection"  
  → Pool gives Thread 2 Connection B

Thread 1: "I'm done with DB"
  → Pool takes Connection A back

Thread 3: "I need a DB connection"
  → Pool gives Thread 3 Connection A (reused!)
```

**Important rule:**

```
pool size >= threads per worker
```

If you have 5 threads but only 3 connections, 2 threads will wait (and might timeout):

```ruby
# Bad:
threads 5, 5      # 5 threads
pool: 3           # only 3 connections → 2 threads blocked!

# Good:
threads 5, 5      # 5 threads
pool: 5           # 5 connections → no waiting
```

---

## Interview Questions — Concurrency

**Q: How does Puma handle multiple requests?**
A: Multi-process (workers) + multi-thread. Each worker has a thread pool. Total capacity = workers × threads.

**Q: What is the GVL?**
A: Ruby's Global VM Lock. Only one thread can execute Ruby code at a time within a process. But threads waiting for I/O release the GVL, allowing other threads to run.

**Q: Why are threads useful despite the GVL?**
A: Web requests spend most time waiting for I/O (database, APIs, Redis). During I/O waits, the GVL is released and other threads can process requests.

**Q: Does Rack handle concurrency?**
A: No. Rack just defines the `call(env)` interface. The server (Puma, Unicorn, Falcon) handles concurrency. Rack's simple interface works with any concurrency model.

**Q: What happens if the connection pool is too small?**
A: Threads wait for a database connection. If they wait too long, `ActiveRecord::ConnectionTimeoutError` is raised.

---

# 5. Streaming

## What is Streaming?

Normally, the server waits for the entire response to be ready, then sends it all at once:

```
Normal response:
  Controller builds entire body
  → all 10,000 users serialized to JSON
  → all loaded into memory
  → sent to browser in one shot
```

Streaming sends data piece by piece as it becomes available:

```
Streaming response:
  Controller starts sending
  → first 100 users → browser receives
  → next 100 users → browser receives
  → next 100 users → browser receives
  → ... until done
```

---

## Why Stream?

### 1. Memory

Without streaming:

```ruby
# All 500MB in memory at once
body = generate_huge_csv  # 500MB string
[200, headers, [body]]
```

With streaming:

```ruby
# Only a few KB in memory at a time
body = Enumerator.new do |yielder|
  User.find_each do |user|
    yielder << user.to_csv_row   # send one row, free memory
  end
end
[200, headers, body]
```

### 2. Time to First Byte (TTFB)

Without streaming:

```
Server: process... process... process... (5 seconds) → SEND ALL
Browser: waiting... waiting... waiting... got it!
```

With streaming:

```
Server: process → SEND → process → SEND → process → SEND
Browser: got some! → got more! → got more! → done!
```

The browser starts receiving data immediately. Users see content faster.

### 3. Large Files

Sending a 2GB file without streaming = 2GB in memory.

Streaming = read and send chunks. Maybe 64KB at a time.

---

## How Streaming Works in Rack

Remember the body contract from Part 1:

```ruby
[status, headers, body]
```

The body must respond to `each`. Rack doesn't say it must be an Array. It can be anything with `each`.

### Streaming with an Enumerator

```ruby
class StreamingApp
  def call(env)
    body = Enumerator.new do |yielder|
      yielder << "Line 1\n"
      sleep 1
      yielder << "Line 2\n"
      sleep 1
      yielder << "Line 3\n"
    end

    [200, { "Content-Type" => "text/plain" }, body]
  end
end
```

When Puma calls `body.each`, the Enumerator yields chunks one at a time. Each chunk is sent to the browser as it's yielded.

The browser sees:

```
(immediately)  Line 1
(after 1 sec)  Line 2
(after 1 sec)  Line 3
```

### Streaming with a Custom Body Object

```ruby
class CsvBody
  def initialize(scope)
    @scope = scope
  end

  def each
    yield "id,name,email\n"   # header row

    @scope.find_each(batch_size: 1000) do |user|
      yield "#{user.id},#{user.name},#{user.email}\n"
    end
  end
end

class CsvExportApp
  def call(env)
    body = CsvBody.new(User.all)

    headers = {
      "Content-Type" => "text/csv",
      "Content-Disposition" => 'attachment; filename="users.csv"'
    }

    [200, headers, body]
  end
end
```

This streams a CSV of all users. Even if you have a million users, only one batch (1000 records) is in memory at a time.

---

## Streaming in Rails

Rails has built-in streaming support:

### 1. ActionController::Live

```ruby
class ReportsController < ApplicationController
  include ActionController::Live

  def export
    response.headers["Content-Type"] = "text/csv"
    response.headers["Content-Disposition"] = 'attachment; filename="report.csv"'

    response.stream.write "id,name,email\n"

    User.find_each do |user|
      response.stream.write "#{user.id},#{user.name},#{user.email}\n"
    end
  ensure
    response.stream.close   # MUST close the stream!
  end
end
```

**Important:** You MUST close the stream. If you forget, the connection hangs forever.

### 2. send_data with streaming

```ruby
def export
  headers["Content-Type"] = "text/csv"
  headers["Content-Disposition"] = 'attachment; filename="users.csv"'

  # This uses a self-closing body
  self.response_body = Enumerator.new do |yielder|
    yielder << CSV.generate_line(["id", "name", "email"])
    User.find_each do |user|
      yielder << CSV.generate_line([user.id, user.name, user.email])
    end
  end
end
```

### 3. Server-Sent Events (SSE)

```ruby
class NotificationsController < ApplicationController
  include ActionController::Live

  def stream
    response.headers["Content-Type"] = "text/event-stream"
    response.headers["Cache-Control"] = "no-cache"

    sse = SSE.new(response.stream)

    10.times do |i|
      sse.write({ message: "Notification #{i}" })
      sleep 2
    end
  ensure
    sse.close
  end
end
```

Browser receives:

```
data: {"message":"Notification 0"}

data: {"message":"Notification 1"}

data: {"message":"Notification 2"}
...
```

---

## Streaming and Middleware — The Catch

Here's something important that trips people up:

**Most middleware buffers the entire response.**

Remember middleware from Part 2:

```ruby
def call(env)
  status, headers, body = @app.call(env)
  # at this point, body is already "done" if it's a normal response
  # but with streaming, body is an enumerator that hasn't run yet!
  [status, headers, body]
end
```

If middleware tries to read the body:

```ruby
def call(env)
  status, headers, body = @app.call(env)
  
  # This BREAKS streaming!
  full_body = ""
  body.each { |chunk| full_body << chunk }
  
  headers["Content-Length"] = full_body.bytesize.to_s
  [status, headers, [full_body]]   # entire body in memory
end
```

This defeats the purpose of streaming — the middleware collects all chunks before sending.

**Which middleware does this?**

```
Rack::ContentLength  → calculates body size (must read entire body)
Rack::ETag           → calculates hash of body (must read entire body)
```

That's why streaming responses usually:

1. Don't set `Content-Length` (use `Transfer-Encoding: chunked` instead)
2. Don't get ETags

Rails handles this automatically when you use `ActionController::Live`.

---

## Transfer-Encoding: chunked

When streaming, the server doesn't know the total body size upfront. So instead of `Content-Length`, it uses:

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked

5\r\n
Hello\r\n
6\r\n
 World\r\n
0\r\n
\r\n
```

Each chunk has its size, then the data. A chunk of size 0 means "done."

The browser knows how to reassemble chunked responses.

Puma handles this automatically when you stream.

---

## Interview Questions — Streaming

**Q: How does streaming work in Rack?**
A: The body can be any object that responds to `each`. Instead of an array with one big string, you return an Enumerator or custom object that yields chunks one at a time.

**Q: Why is streaming useful?**
A: Three reasons: lower memory usage (don't load entire response into memory), faster time-to-first-byte (browser gets data immediately), and handling large files without running out of memory.

**Q: What middleware can break streaming?**
A: Middleware that reads the entire body, like `Rack::ContentLength` or `Rack::ETag`. They buffer everything to calculate size or hash.

**Q: How do you stream in Rails?**
A: Include `ActionController::Live` in the controller. Write to `response.stream`. Always close the stream in an `ensure` block.

**Q: What happens if you forget to close the stream?**
A: The connection stays open forever. The thread is tied up and can't serve other requests. Eventually you run out of threads.

---

# 6. Hijacking

## What is Hijacking?

Hijacking is the most advanced Rack feature. It lets your app take over the raw TCP socket from the server.

Normal Rack:

```
Puma owns the socket
  → Your app returns [status, headers, body]
  → Puma sends HTTP response
  → Puma closes the connection
```

Hijacking:

```
Puma gives you the socket
  → You talk directly to the client
  → You send whatever you want (HTTP, WebSocket, raw bytes)
  → You close the connection when you're done
  → Puma doesn't touch it
```

---

## Why Would You Do This?

Rack's normal response model is:

```
Request → Response → Done
```

But some protocols need persistent, two-way communication:

- **WebSockets** — real-time chat, live updates
- **HTTP/2 Server Push** — push resources without the client asking
- **Custom protocols** — speaking non-HTTP over the connection

For these, the normal `[status, headers, body]` pattern doesn't work. You need the raw socket.

---

## Two Types of Hijacking

### Full Hijacking

Your app takes the socket before the response is sent:

```ruby
class WebSocketApp
  def call(env)
    if env["rack.hijack?"]
      # Tell Rack we're taking over
      env["rack.hijack"].call
      
      # Get the raw IO socket
      io = env["rack.hijack_io"]
      
      # Now we own the socket
      # Send a WebSocket handshake, read/write frames, etc.
      io.write "HTTP/1.1 101 Switching Protocols\r\n"
      io.write "Upgrade: websocket\r\n"
      io.write "Connection: Upgrade\r\n"
      io.write "\r\n"
      
      # Read and write WebSocket frames...
      handle_websocket(io)
      
      # Clean up
      io.close
      
      # Return a dummy response (Puma ignores it)
      [-1, {}, []]
    else
      [200, {}, ["WebSocket not supported"]]
    end
  end
end
```

### Partial Hijacking (Response Hijacking)

The server sends the status and headers. Then your app takes the socket for the body:

```ruby
class StreamApp
  def call(env)
    body = proc do |io|
      # io is the raw socket, after headers are sent
      100.times do |i|
        io.write "chunk #{i}\n"
        io.flush
        sleep 0.1
      end
      io.close
    end

    [200, { "rack.hijack" => body, "Content-Type" => "text/plain" }, []]
  end
end
```

The server sends `200 OK` with headers, then hands the socket to your proc for the body.

---

## Hijacking and Servers

Not all servers support hijacking:

| Server | Full Hijack | Partial Hijack |
|--------|-------------|----------------|
| Puma | ✅ | ✅ |
| Unicorn | ❌ | ❌ |
| Falcon | ✅ | ✅ |
| Thin | ✅ | ✅ |

Check `env["rack.hijack?"]` to know if the server supports it.

---

## Real-World Use: Action Cable

Rails' Action Cable (WebSockets) uses hijacking under the hood.

When a client connects to `/cable`:

```
1. Browser sends WebSocket upgrade request
2. Request goes through Rack middleware
3. Action Cable middleware intercepts /cable requests
4. Hijacks the socket
5. Performs WebSocket handshake
6. Manages the persistent connection
7. Puma's thread is released
```

You never write hijacking code directly for Action Cable. But knowing it uses hijacking helps you understand why WebSockets work in Rails.

---

## Dangers of Hijacking

1. **You own the socket.** If you don't close it, it leaks. If you don't handle errors, it crashes.

2. **You bypass middleware.** Once you hijack, response middleware doesn't run. No logging, no headers, no compression.

3. **Thread management.** If you hijack and do blocking work, you tie up a Puma thread.

4. **Not portable.** Not all servers support hijacking. Your app becomes server-specific.

---

## When to Use Hijacking

| Use Case | Should You Hijack? |
|----------|-------------------|
| Normal HTTP requests | No — use `[status, headers, body]` |
| File downloads | No — use streaming |
| WebSockets | Maybe — or use Action Cable (which hijacks for you) |
| Server-Sent Events | No — use streaming with `ActionController::Live` |
| Custom binary protocol | Yes — hijacking is the only option |

**Most developers never need to hijack directly.** Action Cable and libraries like Faye handle it for you.

---

## Interview Questions — Hijacking

**Q: What is Rack hijacking?**
A: It lets your app take over the raw TCP socket from the server. Instead of returning `[status, headers, body]`, you communicate directly with the client over the socket.

**Q: When would you use hijacking?**
A: For protocols that need persistent, two-way communication like WebSockets. Or custom binary protocols. For most use cases, streaming or Action Cable is better.

**Q: How does Action Cable relate to hijacking?**
A: Action Cable uses Rack hijacking internally to take over the socket for WebSocket connections. But you interact with Action Cable through its high-level API, not raw hijacking.

**Q: What's the difference between full and partial hijacking?**
A: Full hijacking takes the socket before any response is sent — you handle everything. Partial hijacking lets the server send headers first, then you take over for the body.

---

# 7. Async

## What is Async in Rack?

Async means **the server doesn't wait for your app to finish before handling other requests**.

In the normal (synchronous) model:

```
Thread 1:
  Start request → process → DB query (wait) → process → send response
  ████████████████░░░░░░░░░░████████████████████████████
  
  Thread is BLOCKED during DB wait
```

In the async model:

```
Thread 1:
  Start request → process → DB query (don't wait!) → work on other stuff
  ████████████████→ pause → work on request 2 → DB responds → resume request 1
```

The thread doesn't sit idle during I/O.

---

## Why Async Matters

Consider a typical Puma setup:

```
3 workers × 5 threads = 15 concurrent requests max
```

If each request waits 100ms for the database, each thread is doing nothing for 100ms.

With async, that thread can handle other work during those 100ms.

Result: more requests with fewer threads.

---

## Async in Ruby — Fibers

Ruby 3.0+ introduced the Fiber Scheduler, which enables non-blocking I/O without changing your code.

A Fiber is like a lightweight thread:

```
Thread:  heavy, managed by the OS, expensive to create
Fiber:   light, managed by Ruby, cheap to create
```

```ruby
# Conceptually:
Fiber 1: start → I/O wait → suspended
Fiber 2: start → I/O wait → suspended
Fiber 3: start → process → done

# When Fiber 1's I/O completes → resume Fiber 1
```

All fibers run on one thread. No thread-safety issues. No GVL concerns.

---

## Falcon — The Async Rack Server

**Falcon** is a Rack-compatible server built on `async` (a Ruby async I/O library).

```ruby
# Gemfile
gem "falcon"
```

```bash
falcon serve --count 1
```

Falcon uses fibers instead of threads:

```
Falcon Process
├── Fiber 1 → handling request (I/O wait → suspended)
├── Fiber 2 → handling request (processing)
├── Fiber 3 → handling request (I/O wait → suspended)
├── Fiber 4 → handling request (processing)
└── ... (thousands of fibers possible)
```

One Falcon process can handle thousands of concurrent connections using fibers. Much more efficient than Puma's thread model for I/O-heavy workloads.

---

## Rack and Async — How It Works

The Rack spec doesn't have special async support. But it works because:

1. **The body can be lazy.** An Enumerator doesn't run until iterated.
2. **Hijacking lets you take the socket.** For persistent connections.
3. **The server decides the concurrency model.** Rack's `call(env)` works the same whether the server uses threads, processes, or fibers.

### Async Streaming Example (with Falcon)

```ruby
class AsyncStreamApp
  def call(env)
    body = Enumerator.new do |yielder|
      10.times do |i|
        # In Falcon, this sleep is non-blocking
        # The fiber is suspended, and other requests are handled
        sleep 1
        yielder << "Event #{i}\n"
      end
    end

    [200, { "Content-Type" => "text/plain" }, body]
  end
end
```

With Falcon, the `sleep 1` suspends the fiber. Other fibers handle other requests. When the sleep is done, this fiber resumes.

With Puma, the same code blocks the thread for 1 second. During that second, the thread can't handle other requests.

---

## Puma vs Falcon

| Feature | Puma | Falcon |
|---------|------|--------|
| Concurrency model | Threads + Processes | Fibers (async) |
| Memory per connection | Higher (thread stack) | Lower (fiber stack) |
| Max concurrent connections | threads × workers (10-100) | Thousands |
| Thread safety needed | Yes | No (single-threaded fibers) |
| Maturity | Very mature, battle-tested | Newer, growing |
| Rails default | Yes | No |
| Best for | General web apps | I/O-heavy, real-time apps |

**Most Rails apps use Puma.** It's mature, well-tested, and handles typical web workloads well.

**Consider Falcon when:** You have many concurrent connections (WebSockets, long-polling, streaming) and want better I/O efficiency.

---

## Rack 3 and Async

Rack 3 (released 2023) improved async support:

1. **Response body can be any object with `each` or `call`.** A callable body enables push-based streaming.

2. **Headers are now arrays of arrays.** This allows multiple headers with the same name (important for `Set-Cookie`).

3. **Better streaming support.** Rack 3 is more explicit about streaming expectations.

But the core contract remains the same:

```ruby
call(env) → [status, headers, body]
```

Async doesn't change the interface. It changes how the server schedules work.

---

## Interview Questions — Async

**Q: What is async in the context of Rack?**
A: Instead of blocking a thread while waiting for I/O, the runtime suspends the work (using fibers) and handles other requests. When I/O completes, the work resumes. This allows handling many more concurrent connections.

**Q: What is Falcon?**
A: An async Rack-compatible web server using Ruby fibers. It handles thousands of concurrent connections on a single process, unlike Puma which uses threads.

**Q: Do you need to change your Rack app for async?**
A: No. The `call(env)` interface is the same. The server handles async scheduling. Your app code doesn't need to change (though it benefits more if it uses non-blocking I/O).

**Q: When would you choose Falcon over Puma?**
A: For workloads with many concurrent long-lived connections: WebSockets, SSE, long-polling, real-time features. For typical request-response web apps, Puma is simpler and more battle-tested.

---

# 8. HTTP/2 Considerations

## HTTP/1.1 vs HTTP/2 — Quick Recap

HTTP/1.1 (what most Rails apps use):

```
One request, one response, one at a time per connection.

Browser: "Give me page.html"
Server:  "Here's page.html"
Browser: "Give me style.css"
Server:  "Here's style.css"
Browser: "Give me app.js"
Server:  "Here's app.js"
```

HTTP/2:

```
Multiple requests and responses at the same time on ONE connection.

Browser: "Give me page.html AND style.css AND app.js"
Server:  [sends all three simultaneously as interleaved frames]
```

Key HTTP/2 features:

| Feature | HTTP/1.1 | HTTP/2 |
|---------|----------|--------|
| Multiplexing | No (one request per connection) | Yes (many requests per connection) |
| Header compression | No | Yes (HPACK) |
| Server Push | No | Yes (server can send resources before asked) |
| Binary protocol | No (text-based) | Yes (binary frames) |
| Stream prioritization | No | Yes |

---

## Does Rack Support HTTP/2?

**Short answer:** Not fully. Rack was designed for HTTP/1.1's request-response model.

**Longer answer:**

The core Rack interface:

```ruby
call(env) → [status, headers, body]
```

This is fundamentally a **one-request, one-response** model. HTTP/2's multiplexing and server push don't fit neatly into this.

### What Works

**Basic HTTP/2 requests work fine.** A regular GET request over HTTP/2 still maps to `call(env) → [status, headers, body]`. Each HTTP/2 stream becomes one Rack `call`.

```
HTTP/2 stream 1: GET /users     → app.call(env1) → [200, headers, body]
HTTP/2 stream 3: GET /products  → app.call(env2) → [200, headers, body]
HTTP/2 stream 5: POST /orders   → app.call(env3) → [201, headers, body]
```

Each stream is handled independently. Rack doesn't know (or care) that they're multiplexed.

### What Doesn't Work (Natively)

**Server Push.** HTTP/2 lets the server send resources before the client asks. For example, when the client requests `index.html`, the server can push `style.css` and `app.js` along with it.

Rack has no built-in way to say "also send these other responses." The `call(env)` contract returns one response per call.

Some servers work around this with Rack extensions:

```ruby
def call(env)
  # Some servers support this (non-standard Rack extension)
  if push_promise = env["rack.early_hints"]
    push_promise.call("link" => "</style.css>; rel=preload; as=style")
  end

  [200, headers, body]
end
```

Rails supports Early Hints (a related feature):

```ruby
class ApplicationController < ActionController::Base
  def index
    early_hints("link" => "</style.css>; rel=preload; as=style")
    render :index
  end
end
```

But this is limited compared to full HTTP/2 Server Push.

---

## Where HTTP/2 Actually Happens

In most production setups, HTTP/2 is handled by the reverse proxy (Nginx, CloudFlare), not by Rails or Puma:

```
Browser ←─ HTTP/2 ─→ Nginx/CloudFlare ←─ HTTP/1.1 ─→ Puma ←─ Rack ─→ Rails
```

The browser talks HTTP/2 with Nginx. Nginx translates to HTTP/1.1 and talks to Puma. Puma and Rack never see HTTP/2.

This is the most common setup and it works well because:

1. Nginx handles HTTP/2's complexity (multiplexing, header compression, prioritization)
2. Puma handles the simple HTTP/1.1 request-response cycle
3. Rails doesn't need to change

---

## Puma and HTTP/2

Puma currently speaks HTTP/1.1 only. It doesn't handle HTTP/2 directly.

For most apps, this is fine because Nginx or a CDN (CloudFlare, Fastly) handles HTTP/2 termination.

---

## Falcon and HTTP/2

Falcon supports HTTP/2 natively:

```bash
falcon serve --bind https://localhost:9292
```

With Falcon, the entire chain can be HTTP/2:

```
Browser ←─ HTTP/2 ─→ Falcon ←─ Rack ─→ Your App
```

No Nginx needed (though you might still want it for other reasons).

Falcon maps HTTP/2 streams to Rack `call` invocations. Each stream gets its own `call(env)`.

---

## HTTP/3 and QUIC

HTTP/3 (over QUIC) is even newer. It replaces TCP with UDP for better performance.

Rack's position: same as HTTP/2. The `call(env)` interface doesn't change. Servers and reverse proxies handle the protocol details.

```
Browser ←─ HTTP/3/QUIC ─→ CloudFlare ←─ HTTP/1.1 ─→ Puma ←─ Rack ─→ Rails
```

---

## How to Actually Use HTTP/2 with Rails

### Option 1: Nginx Termination (Most Common)

```nginx
# nginx.conf
server {
    listen 443 ssl http2;
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://localhost:3000;  # Puma on HTTP/1.1
    }
}
```

Browser ↔ Nginx uses HTTP/2.
Nginx ↔ Puma uses HTTP/1.1.

### Option 2: CDN Termination

CloudFlare, Fastly, AWS CloudFront all support HTTP/2 automatically.

```
Browser ←─ HTTP/2 ─→ CloudFlare ←─ HTTP/1.1 ─→ Your Server ←─ Puma
```

You don't configure anything in Rails. The CDN handles it.

### Option 3: Falcon (Full HTTP/2)

```ruby
# Gemfile
gem "falcon"
```

```bash
falcon serve --bind https://localhost:9292
```

Your Rack app speaks HTTP/2 end-to-end. No reverse proxy needed.

---

## Rack's Future with HTTP/2

The Rack community is aware of HTTP/2's limitations in the current spec. Some ideas being explored:

1. **Extended `env` keys** for HTTP/2 metadata (stream ID, priority)
2. **Push promise API** as a standard part of Rack
3. **Better streaming primitives** that work naturally with multiplexing

But for now, the practical approach is:

```
Use Nginx/CDN for HTTP/2 ←→ Browser
Use Puma/Rack for HTTP/1.1 ←→ App
```

This works great for the vast majority of Rails applications.

---

## Interview Questions — HTTP/2

**Q: Does Rack support HTTP/2?**
A: Partially. Basic HTTP/2 requests work fine (each stream maps to a `call`). But HTTP/2-specific features like Server Push don't have native Rack support. Most apps handle HTTP/2 at the Nginx/CDN layer.

**Q: How does a typical Rails production setup handle HTTP/2?**
A: Nginx or a CDN terminates HTTP/2 from the browser, then proxies HTTP/1.1 to Puma. Rails and Rack never see HTTP/2 directly.

**Q: Can Rails do Server Push?**
A: Rails supports Early Hints (a limited form), but full HTTP/2 Server Push requires server support. In practice, CDNs handle push better than application servers.

**Q: What's the difference between Puma and Falcon regarding HTTP/2?**
A: Puma only speaks HTTP/1.1. Falcon supports HTTP/2 natively, so your app can speak HTTP/2 end-to-end without a reverse proxy.

---

# Mental Model — Part 4

```
Rack Internals
├── Builder builds chain (inside-out, reverse_each)
├── Request wraps env (thin wrapper, no copying)
├── Response builds [status, headers, body]
└── Lint validates the spec

Concurrency
├── Puma: processes × threads
├── Middleware instances shared across threads
├── GVL: only one thread runs Ruby, but I/O releases it
├── Falcon: fibers, thousands of connections
└── Connection pool: threads need DB connections

Streaming
├── Body responds to each
├── Enumerator yields chunks
├── ActionController::Live for Rails
├── Middleware can break streaming (ContentLength, ETag)
└── Transfer-Encoding: chunked

Hijacking
├── Take raw TCP socket from server
├── Full: before response, you handle everything
├── Partial: server sends headers, you send body
├── Action Cable uses this internally
└── Most developers never need it directly

Async
├── Fibers: lightweight threads
├── Non-blocking I/O
├── Falcon: async Rack server
├── Same call(env) interface
└── Better for I/O-heavy, many-connection workloads

HTTP/2
├── Multiplexing, header compression, server push
├── Rack designed for HTTP/1.1 request-response
├── Basic HTTP/2 requests work (one stream = one call)
├── Server Push not natively supported
├── Most apps: Nginx/CDN handles HTTP/2, Puma uses HTTP/1.1
└── Falcon supports HTTP/2 natively
```

---

# Key Takeaways — Part 4

1. **Rack is small** — ~50 files. The core is just Builder, Request, Response, and Utils.
2. **Builder builds inside-out** — `reverse_each` ensures first `use` is the outermost middleware.
3. **Thread safety** — middleware instances are shared across threads. Never write to instance variables in `call`. Use local variables or `env`.
4. **Puma concurrency** — processes × threads. GVL limits parallel Ruby execution but I/O releases it.
5. **Streaming** — return a body that yields chunks. Reduces memory and speeds up time-to-first-byte.
6. **Hijacking** — take the raw socket for protocols like WebSocket. Action Cable does this internally.
7. **Async** — Fibers enable non-blocking I/O. Falcon handles thousands of connections per process.
8. **HTTP/2** — Rack works for basic HTTP/2 but full features need server/proxy support. Nginx handles HTTP/2 termination in most setups.

---

# 🎉 Part 4 Complete

You now understand:

* ✅ Internal Implementation — how Rack is built (Builder, Request, Response, Utils, Lint)
* ✅ Source Code Walkthrough — real Rack code, line by line
* ✅ Thread Safety — what's safe, what's not, how to fix it
* ✅ Concurrency — Puma's model, GVL, connection pools, threads vs processes
* ✅ Streaming — chunked responses, Enumerators, ActionController::Live
* ✅ Hijacking — taking the socket, full vs partial, Action Cable
* ✅ Async — fibers, Falcon, non-blocking I/O
* ✅ HTTP/2 Considerations — what works, what doesn't, production setups

**Next up in Part 5:** Performance, Security, Debugging, Logging, Common Mistakes, Best Practices, and Anti-patterns.

---

## Progress Tracker — Rack Study Guide

| Part | Topics | Status |
|------|--------|--------|
| **Part 1** | Overview, History, Why Rack Exists, Problems Rack Solves, Rack Architecture, Rack Specification, Rack Application, Rack Environment, Hello World, Request Flow | 🟢 Complete |
| **Part 2** | Rack::Request, Rack::Response, Rack::Builder, config.ru (Rackup), Middleware Fundamentals, Middleware Chain, Complete Request Lifecycle | 🟢 Complete |
| **Part 3** | Rails + Rack, ActionDispatch, Rails Middleware Stack, Middleware Ordering, Custom Middleware, Production Examples | 🟢 Complete |
| **Part 4** | Internal Implementation, Source Code Walkthrough, Thread Safety, Concurrency, Streaming, Hijacking, Async, HTTP/2 | 🟢 Complete |
| **Part 5** | Performance, Security, Debugging, Logging, Common Mistakes, Best Practices, Anti-patterns | ⬜ Not Started |
| **Part 6** | Advanced Examples, Edge Cases, Comparison Tables, Interview Questions, Coding Exercises, Cheat Sheet, Summary, Resources | ⬜ Not Started |
