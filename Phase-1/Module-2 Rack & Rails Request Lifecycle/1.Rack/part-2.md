# Part 2 — Rack Utilities and Middleware

Welcome back.

In Part 1, you learned one simple rule:

```ruby
call(env) → [status, headers, body]
```

You also learned that `env` is a Hash. Puma builds it from the HTTP request and gives it to your app.

That works. But reading `env` directly gets messy fast:

```ruby
env["HTTP_USER_AGENT"]
env["HTTP_X_REQUEST_ID"]
env["QUERY_STRING"]  # you have to parse this yourself
```

Part 2 makes your life easier. Rack gives you helper tools. You also learn **middleware** — the thing Rails uses for sessions, cookies, logging, and much more.

Take it one section at a time. Don't rush.

---

# What You'll Learn in Part 2

1. **Rack::Request** — easy way to read incoming requests
2. **Rack::Response** — easy way to build outgoing responses
3. **Rack::Builder** — easy way to stack middleware
4. **config.ru** — the file that starts your app
5. **Middleware** — what it is and why it exists
6. **Middleware Chain** — how multiple middleware work together
7. **Complete Request Lifecycle** — one request, start to finish

---

# 1. Rack::Request

## The Problem

In Part 1, Puma gave you a raw Hash called `env`:

```ruby
env["REQUEST_METHOD"]   # => "GET"
env["PATH_INFO"]        # => "/users"
env["HTTP_USER_AGENT"]  # => "Mozilla/5.0..."
```

It works. But it's annoying:

- Headers look ugly: `env["HTTP_X_REQUEST_ID"]`
- Query params are a raw string: `"page=2&sort=name"` — you parse it yourself
- Cookies are a raw string too
- The request body lives in `rack.input` — you read it yourself

## The Solution: Rack::Request

Think of `Rack::Request` as a **helper that reads the env hash for you**.

Instead of digging through `env` manually, you write:

```ruby
request = Rack::Request.new(env)

request.get?           # is this a GET request?
request.path           # what path was requested?
request.params         # give me the params (already parsed!)
request.headers["User-Agent"]  # clean header access
```

That's it. Same data. Much easier to use.

## Important: It Does NOT Copy env

```ruby
request = Rack::Request.new(env)

request.env.equal?(env)  # => true (same object!)
```

`Rack::Request` just **wraps** the hash. It doesn't make a copy.

So if middleware changes `env` before your app runs, your `Rack::Request` sees those changes too.

---

## How to Use It

```ruby
class MyApp
  def call(env)
    request = Rack::Request.new(env)

    if request.get? && request.path == "/hello"
      [200, { "Content-Type" => "text/plain" }, ["Hello!"]]
    else
      [404, {}, ["Not Found"]]
    end
  end
end
```

Rails does the same thing. When you write `request.path` in a controller, Rails is using something built on top of `Rack::Request`.

---

## Reading the HTTP Method

```ruby
request.request_method  # => "GET", "POST", "PUT", etc.

request.get?      # true or false
request.post?     # true or false
request.put?      # true or false
request.delete?   # true or false
request.patch?    # true or false
```

Under the hood, `request.get?` just checks if `env["REQUEST_METHOD"] == "GET"`. Simple.

---

## Reading the Path and URL

```ruby
request.path_info     # => "/users/10"
request.path          # => "/users/10"
request.fullpath      # => "/users/10?page=2"
request.query_string  # => "page=2&sort=name"
request.url           # => "http://localhost:3000/users/10?page=2"
request.base_url      # => "http://localhost:3000"
```

**Two extra ones you might see:**

```ruby
request.script_name   # => "" or "/api"
```

When your app is mounted inside another app (like `/api`), Rack splits the path:

```
Full URL path: /api/users

script_name = "/api"    (the mount point)
path_info   = "/users"  (what your app sees)
```

You won't need this every day. But it comes up with mounted Rails engines and `Rack::Builder#map`.

---

## Reading Parameters (params)

This is the one you'll use most.

```ruby
request.params
# => { "page" => "2", "name" => "Alice" }
```

Rack parses the params for you. No manual string splitting.

**Where do params come from?**

```
URL query string     →  ?page=2&sort=name
Form body (POST)     →  name=Alice&email=test@example.com
```

Rack merges them into one hash.

**Example — GET request:**

```http
GET /users?page=2&sort=name
```

```ruby
request.GET    # => { "page" => "2", "sort" => "name" }
request.POST   # => {}
request.params # => { "page" => "2", "sort" => "name" }
```

**Example — POST form:**

```http
POST /users
Content-Type: application/x-www-form-urlencoded

name=Alice&email=alice@example.com
```

```ruby
request.POST   # => { "name" => "Alice", "email" => "alice@example.com" }
request.params # => { "name" => "Alice", "email" => "alice@example.com" }
```

### JSON Bodies — Important!

`Rack::Request` does **NOT** parse JSON.

If the client sends:

```http
POST /users
Content-Type: application/json

{"name": "Alice"}
```

Then:

```ruby
request.params       # => {}  (empty!)
request.body.read    # => '{"name": "Alice"}'  (raw string)
```

Rails parses JSON for you in controllers. Plain Rack does not. You parse it yourself:

```ruby
if request.media_type == "application/json"
  data = JSON.parse(request.body.read)
  request.body.rewind  # important! (explained below)
end
```

---

## Reading Headers

**The old way (ugly):**

```ruby
env["HTTP_USER_AGENT"]
env["HTTP_AUTHORIZATION"]
env["HTTP_X_REQUEST_ID"]
```

**The new way (clean):**

```ruby
request.headers["User-Agent"]
request.headers["Authorization"]
request.headers["X-Request-Id"]
```

Much better.

---

## Reading Cookies

```ruby
request.cookies
# => { "session_id" => "abc123" }

request.cookies["session_id"]
# => "abc123"
```

Rack reads the `Cookie` header and parses it for you.

**Note:** To **set** cookies, you use the **response** (`Rack::Response`), not the request.

---

## Reading the Request Body

```ruby
request.body.read    # reads the raw body as a string
request.body.rewind  # resets so you can read again
```

### Common Bug — Body Can Only Be Read Once

The body is like a tape — once you read it, the pointer moves forward.

```ruby
# Middleware reads the body
request.body.read   # => '{"name": "Alice"}'

# App tries to read params
request.params    # => {}  (body already consumed!)
```

**Fix:** Always rewind after reading:

```ruby
body = request.body.read
request.body.rewind   # reset pointer
# now other code can read the body too
```

---

## Other Useful Methods

```ruby
request.content_type   # => "application/json; charset=utf-8"
request.media_type     # => "application/json"  (without extra stuff)
request.host           # => "example.com"
request.port           # => 443
request.scheme         # => "https"
request.ssl?           # => true
request.ip             # => "127.0.0.1"
request.user_agent     # => "Mozilla/5.0..."
request.xhr?           # => true if AJAX request
```

---

## Simple Summary — Rack::Request

| What you want | How to get it |
|---------------|---------------|
| HTTP method | `request.get?`, `request.post?` |
| Path | `request.path` |
| Params | `request.params` |
| Headers | `request.headers["Name"]` |
| Cookies | `request.cookies["name"]` |
| Body | `request.body.read` |
| Full URL | `request.url` |

**Remember:** It's just a wrapper around `env`. Same data, easier access.

---

## Common Mistakes

**1. Expecting JSON in params**

```ruby
request.params["name"]  # nil for JSON body — won't work!
```

**2. Reading body without rewinding**

```ruby
request.body.read
# forgot to rewind — downstream code gets empty body
```

**3. Thinking it copies env**

It doesn't. Same hash, shared between middleware and app.

---

## Interview Questions — Rack::Request

**Q: Does Rack::Request copy the env hash?**  
A: No. It wraps the same hash. Changes to env are visible through the request object.

**Q: Does it parse JSON?**  
A: No. Only form data and query strings. JSON parsing is done by frameworks like Rails.

**Q: What happens if you read the body twice?**  
A: Second read is empty unless you call `request.body.rewind`.

**Q: How does Rails' request object relate to this?**  
A: `ActionDispatch::Request` inherits from `Rack::Request` and adds Rails features on top.

---

# 2. Rack::Response

## The Problem

Part 1 showed how to return a response:

```ruby
[
  200,
  { "Content-Type" => "text/html" },
  ["<h1>Hello</h1>"]
]
```

This works. But it gets messy when you need:

- Multiple headers
- Cookies
- Redirects
- Auto-calculated Content-Length

## The Solution: Rack::Response

Think of `Rack::Response` as a **helper that builds the response array for you**.

```ruby
response = Rack::Response.new
response.status = 200
response["Content-Type"] = "application/json"
response.write('{"message": "Hello"}')
response.finish   # returns [status, headers, body]
```

---

## Basic Usage

```ruby
class MyApp
  def call(env)
    response = Rack::Response.new
    response.write("Hello World")
    response.finish
  end
end
```

**Always call `finish` at the end.** That gives you the `[status, headers, body]` array Rack expects.

---

## Setting Status

```ruby
response.status = 200   # OK
response.status = 404   # Not Found
response.status = 500   # Server Error
```

Default is 200 if you don't set it.

---

## Setting Headers

```ruby
response["Content-Type"] = "application/json"
response["X-Custom"] = "my-value"
```

Or use helper methods:

```ruby
response.content_type = "application/json"
```

---

## Writing the Body

```ruby
response.write("Hello")
response.write(" World")
# body is now "Hello World"
```

Each `write` adds to the body. Like building a string piece by piece.

---

## Setting Cookies

```ruby
response.set_cookie("session_id", value: "abc123", path: "/", httponly: true)
```

Without `Rack::Response`, you'd write this by hand:

```ruby
{ "Set-Cookie" => "session_id=abc123; path=/; HttpOnly" }
```

Easy to get wrong. `set_cookie` handles the formatting.

To delete a cookie:

```ruby
response.delete_cookie("session_id")
```

---

## Redirects

```ruby
response.redirect("/login")       # 302 redirect
response.redirect("/login", 301)  # permanent redirect
```

That's it. No need to manually set status + Location header.

---

## finish — The Important Method

```ruby
status, headers, body = response.finish
# => [200, { "Content-Type" => "text/plain", "Content-Length" => "11" }, ["Hello World"]]
```

`finish` does two things:

1. Auto-sets `Content-Length` if needed
2. Returns the `[status, headers, body]` array

**Return this from your `call` method:**

```ruby
def call(env)
  res = Rack::Response.new
  res.write("Hello")
  res.finish
end
```

---

## Full Example

```ruby
class JsonApp
  def call(env)
    request = Rack::Request.new(env)

    res = Rack::Response.new

    if request.path == "/health"
      res.status = 200
      res.content_type = "application/json"
      res.write('{"status": "ok"}')
    else
      res.status = 404
      res.content_type = "application/json"
      res.write('{"error": "Not Found"}')
    end

    res.finish
  end
end
```

---

## Raw Array vs Rack::Response

| Task | Raw Array | Rack::Response |
|------|-----------|----------------|
| Set status | `[404, ...]` | `response.status = 404` |
| Set header | `{ "Content-Type" => "..." }` | `response["Content-Type"] = "..."` |
| Write body | `["Hello"]` | `response.write("Hello")` |
| Redirect | Manual 302 + Location | `response.redirect("/login")` |
| Set cookie | Manual Set-Cookie string | `response.set_cookie(...)` |

**When to use raw array:** Simple responses, middleware that returns early (like 401 Unauthorized).

**When to use Rack::Response:** App code with headers, cookies, or redirects.

---

## Interview Questions — Rack::Response

**Q: What does finish do?**  
A: Returns the `[status, headers, body]` array. Also sets Content-Length automatically.

**Q: How do redirects work?**  
A: Set status 301/302 and a Location header. Or just call `response.redirect("/path")`.

**Q: Can you change the response after finish?**  
A: Technically yes, but don't. The server may have already started sending it.

---

# 3. Rack::Builder

## The Problem

Real apps aren't just one class. They have layers:

```
Request → Logger → Auth → Your App → Response
```

Building this by hand is annoying:

```ruby
app = MyApp.new
app = AuthMiddleware.new(app)
app = LoggerMiddleware.new(app)
# hard to read, easy to get order wrong
```

## The Solution: Rack::Builder

`Rack::Builder` gives you a simple DSL to declare your stack:

```ruby
Rack::Builder.new do
  use LoggerMiddleware
  use AuthMiddleware
  run MyApp.new
end
```

Read top to bottom. First `use` = outermost layer. `run` = your actual app at the bottom.

---

## The Three Commands

### `use` — Add Middleware

```ruby
use MyMiddleware
use MyMiddleware, some_option: "value"
```

Each middleware wraps whatever comes below it.

---

### `run` — Set Your App

```ruby
run MyApp.new
```

This is the core application — the innermost layer. The thing that actually handles the request.

Only one `run` per builder block.

---

### `map` — Mount an App at a Path

```ruby
Rack::Builder.new do
  map "/admin" do
    run AdminApp.new
  end

  map "/api" do
    run ApiApp.new
  end

  run MainApp.new
end
```

- `/admin/*` goes to AdminApp
- `/api/*` goes to ApiApp
- everything else goes to MainApp

This is how mounted Rails engines work at the Rack level.

---

## How the Chain Gets Built

You write:

```ruby
Rack::Builder.new do
  use A
  use B
  run C.new
end
```

Rack builds it from the inside out:

```
Step 1:  app = C.new
Step 2:  app = B.new(app)
Step 3:  app = A.new(app)

Final:   A → B → C
```

When a request comes in:

```
Request hits A first → then B → then C
```

---

## Getting the Final App

```ruby
app = Rack::Builder.app do
  use Rack::ContentLength
  run MyApp.new
end

# Now you can call it:
app.call(env)
```

`Rack::Builder.app` is shorthand for "build the stack and give me the final app."

---

## Interview Questions — Rack::Builder

**Q: What order does middleware run?**  
A: Top to bottom in the file. First `use` sees the request first.

**Q: What's the difference between use and run?**  
A: `use` adds a middleware layer. `run` sets the core app at the bottom.

**Q: What does map do?**  
A: Mounts a separate app at a URL prefix like `/api` or `/admin`.

---

# 4. config.ru (Rackup)

## What is config.ru?

Every Rack app has a file called `config.ru` at the project root.

It's the **startup file**. It tells the server:

1. What middleware to use
2. What app to run

Think of it as the "main()" of your web app.

---

## Minimal config.ru

```ruby
require_relative "app"

run MyApp.new
```

That's a complete Rack app. Load your app class, run it. Done.

---

## Typical config.ru with Middleware

```ruby
require_relative "app"

use Rack::ContentLength
use Rack::CommonLogger
use MyCustomMiddleware

run MyApp.new
```

---

## Rails' config.ru

Open any Rails project. You'll see:

```ruby
require_relative "config/environment"

run Rails.application
```

Three lines. But those three lines boot the entire Rails app — routes, models, middleware, everything.

`Rails.application` is a Rack app. It responds to `call(env)`.

---

## What is rackup?

`rackup` is a command that reads `config.ru` and starts a web server:

```bash
rackup                  # starts on port 9292
rackup -p 3000          # custom port
rackup -E production    # production mode
```

In development, `rackup` is fine.

In production, you usually run Puma directly:

```bash
bundle exec puma -C config/puma.rb
```

Puma still reads `config.ru` to know what app to run. It just doesn't use the `rackup` command.

---

## How rackup Loads config.ru

Simple steps:

```
1. Find config.ru
2. Create a Rack::Builder
3. Run the code in config.ru (use, run, etc.)
4. Build the middleware chain
5. Start the web server with that app
```

That's why `use` and `run` work at the top level of config.ru — the file runs inside a `Rack::Builder` context.

---

## RACK_ENV

```ruby
# config.ru
case ENV["RACK_ENV"]
when "production"
  use Rack::CommonLogger
when "development"
  use Rack::ShowExceptions   # pretty error pages
end

run MyApp.new
```

| Value | When |
|-------|------|
| `development` | Local dev (default) |
| `production` | Live server |
| `test` | Running tests |

Rails uses `RAILS_ENV` but falls back to `RACK_ENV`.

---

## config.ru vs config/environment.rb (Rails)

| File | What it does |
|------|-------------|
| `config.ru` | Rack entry point — tells server what to run |
| `config/environment.rb` | Boots Rails — loads gems, routes, initializers |

In a plain Rack app, `config.ru` is the only boot file.

In Rails, `config.ru` just points to `Rails.application`. Rails does the rest.

---

## Interview Questions — config.ru

**Q: What is config.ru?**  
A: The Rack startup file. Defines middleware and the app. Entry point for web servers.

**Q: What's the difference between rackup and Puma?**  
A: `rackup` is a generic launcher. Puma is a production server that reads config.ru directly.

**Q: What does `run Rails.application` do?**  
A: Sets the fully booted Rails app as the Rack application.

---

# 5. Rack Middleware Fundamentals

## What is Middleware?

Here's the simplest definition:

> **Middleware is a Rack app that wraps another Rack app.**

That's it.

```ruby
class MyMiddleware
  def initialize(app)
    @app = app    # the next layer down
  end

  def call(env)
    # do stuff BEFORE the request reaches the app
    status, headers, body = @app.call(env)
    # do stuff AFTER the app responds
    [status, headers, body]
  end
end
```

Think of middleware like **checkpoints on the way to your app**:

```
Request → Checkpoint 1 → Checkpoint 2 → Your App → Response
```

Each checkpoint can:
- Look at the request
- Change the request
- Block the request (return 401, 403, etc.)
- Look at the response
- Change the response

---

## The @app Pattern

Every middleware stores the next layer in `@app`:

```ruby
def initialize(app)
  @app = app
end
```

When you write:

```ruby
use AuthMiddleware
use LoggerMiddleware
run MyApp.new
```

Rack builds:

```
AuthMiddleware wraps LoggerMiddleware wraps MyApp
```

Inside AuthMiddleware, `@app` is LoggerMiddleware.

Inside LoggerMiddleware, `@app` is MyApp.

---

## Three Things Middleware Can Do

### 1. Before — Run code before the app

```ruby
def call(env)
  env["start_time"] = Time.now    # add something to env
  @app.call(env)
end
```

Examples: start a timer, check auth, assign a request ID.

---

### 2. Delegate — Pass the request to the next layer

```ruby
status, headers, body = @app.call(env)
```

This sends the request deeper into the chain.

---

### 3. After — Run code after the app responds

```ruby
def call(env)
  status, headers, body = @app.call(env)
  headers["X-Runtime"] = "0.05"   # add a response header
  [status, headers, body]
end
```

Examples: add timing header, log the response, compress the body.

---

## Changing the Request (env)

Middleware can add data to `env` for downstream layers:

```ruby
class SetCurrentUser
  def initialize(app)
    @app = app
  end

  def call(env)
    request = Rack::Request.new(env)
    token = request.headers["Authorization"]

    env["current_user"] = find_user(token)

    @app.call(env)
  end
end
```

Now your app can read `env["current_user"]`.

**Tip:** Use a prefix to avoid name clashes:

```ruby
env["my_app.current_user"] = user   # good
env["current_user"] = user           # might clash with other middleware
```

---

## Short-Circuiting — Stopping Early

Middleware can return a response **without** calling the app:

```ruby
class AuthMiddleware
  def initialize(app)
    @app = app
  end

  def call(env)
    request = Rack::Request.new(env)

    unless logged_in?(request)
      return [401, { "Content-Type" => "application/json" },
              ['{"error": "Unauthorized"}']]
    end

    @app.call(env)
  end
end
```

If the user isn't logged in, the app **never runs**. Request stops at the middleware.

This is how auth, rate limiting, and maintenance mode work.

---

## Error Handling Middleware

```ruby
class ErrorHandler
  def initialize(app)
    @app = app
  end

  def call(env)
    @app.call(env)
  rescue StandardError => e
    puts "Error: #{e.message}"
    [500, { "Content-Type" => "application/json" },
     ['{"error": "Something went wrong"}']]
  end
end
```

If anything inside raises an error, this middleware catches it and returns a clean 500.

Put error handling middleware **outermost** (first `use`) so it catches errors from all inner layers.

---

## Built-in Rack Middleware (Quick Reference)

Rack ships with useful middleware:

| Middleware | What it does |
|-----------|-------------|
| `Rack::ContentLength` | Sets Content-Length header |
| `Rack::CommonLogger` | Logs requests (Apache-style) |
| `Rack::ShowExceptions` | Pretty error pages in dev |
| `Rack::Runtime` | Adds X-Runtime header (how long request took) |
| `Rack::Deflater` | Compresses response (gzip) |
| `Rack::Static` | Serves files from a folder |
| `Rack::Head` | Handles HEAD requests (empty body) |
| `Rack::MethodOverride` | Allows `_method=DELETE` in forms |
| `Rack::ETag` | Adds caching ETag headers |

You'll see Rails' own middleware in Part 3.

---

## Writing Your Own Middleware — Template

Copy this template:

```ruby
class MyMiddleware
  def initialize(app)
    @app = app
  end

  def call(env)
    # --- BEFORE ---
    start = Time.now

    # --- PASS TO APP ---
    status, headers, body = @app.call(env)

    # --- AFTER ---
    headers["X-Runtime"] = (Time.now - start).to_s

    [status, headers, body]
  end
end
```

Add it in config.ru:

```ruby
use MyMiddleware
run MyApp.new
```

---

## Interview Questions — Middleware

**Q: Is middleware a Rack app?**  
A: Yes. It has `call(env)` and returns `[status, headers, body]`. The only extra thing is it takes an inner app in `initialize`.

**Q: Can middleware stop a request before it reaches the app?**  
A: Yes. Return a response directly without calling `@app.call(env)`. This is called short-circuiting.

**Q: Can middleware change the request?**  
A: Yes. Modify the `env` hash before calling `@app.call(env)`.

**Q: Can middleware change the response?**  
A: Yes. Modify status, headers, or body after `@app.call(env)` returns.

---

# 6. Middleware Chain

## The Onion Model

The best way to understand middleware is an **onion**:

```
        ┌─────────────────────┐
        │   Middleware A      │  ← outermost (first to see request)
        │  ┌───────────────┐  │
        │  │ Middleware B  │  │
        │  │  ┌─────────┐  │  │
        │  │  │  App    │  │  │  ← innermost (handles the request)
        │  │  └─────────┘  │  │
        │  └───────────────┘  │
        └─────────────────────┘
```

**Request goes IN:** A → B → App  
**Response goes OUT:** App → B → A

Like peeling an onion layer by layer.

---

## Order Matters

```ruby
Rack::Builder.new do
  use MiddlewareA   # sees request FIRST
  use MiddlewareB   # sees request SECOND
  run MyApp.new     # handles request LAST
end
```

Request: `A → B → MyApp`  
Response: `MyApp → B → A`

**Why order matters — real example:**

```ruby
use AuthMiddleware    # must run BEFORE app (block bad users early)
use Rack::Deflater    # must run AFTER app (compress the response body)
run MyApp.new
```

Wrong order = bugs:

```ruby
use Rack::Deflater
use AuthMiddleware    # BAD — Deflater wraps Auth, wastes effort compressing 401 errors
run MyApp.new
```

---

## How the Chain is Built (Step by Step)

You write:

```ruby
use A
use B
run C.new
```

Rack builds inside-out:

```
1. app = C.new
2. app = B.new(app)
3. app = A.new(app)

Result: A → B → C
```

Server calls `app.call(env)` which starts at A.

---

## Request In, Response Out

Think of it like a phone tree:

```
Boss (Middleware A) gets the call
  → passes to Manager (Middleware B)
    → passes to Worker (App)
    → Worker responds to Manager
  → Manager responds to Boss
→ Boss responds to caller
```

Each layer can add something on the way in and on the way out.

**Example — logging middleware:**

```ruby
class Logger
  def initialize(app)
    @app = app
  end

  def call(env)
    puts "→ #{env['REQUEST_METHOD']} #{env['PATH_INFO']}"   # going IN

    status, headers, body = @app.call(env)

    puts "← #{status}"   # coming OUT

    [status, headers, body]
  end
end
```

For a stack A → B → App, the output looks like:

```
→ A: GET /users
  → B: GET /users
    App handles it
  ← B: 200
← A: 200
```

---

## How to Inspect the Chain

**Plain Rack app:**

```ruby
app = Rack::Builder.app do
  use MiddlewareA
  use MiddlewareB
  run MyApp.new
end

# Walk the chain
current = app
while current
  puts current.class
  current = current.instance_variable_get(:@app) rescue nil
end
```

**Rails app:**

```bash
rails middleware
```

Shows the full stack. Super useful when debugging "why isn't my middleware running?"

---

## Interview Questions — Middleware Chain

**Q: Which middleware sees the request first?**  
A: The one declared first (topmost `use` in config.ru).

**Q: Why does order matter?**  
A: Auth must run before the app. Compression must run after the body is built. Logging should wrap everything.

**Q: How do you see the middleware stack in Rails?**  
A: Run `rails middleware`.

---

# 7. Complete Request Lifecycle through Rack Middleware

Now let's follow **one request** from start to finish.

No skipping. Every step.

---

## Our Setup

```ruby
# config.ru

use RequestId       # adds unique ID to each request
use Authenticator   # checks login token
use Timer           # measures how long request takes
use Rack::ContentLength
use Rack::CommonLogger

run MyApp.new
```

```ruby
# app.rb
class MyApp
  def call(env)
    request = Rack::Request.new(env)
    user = env["current_user"]

    body = JSON.generate({
      message: "Hello, #{user['name']}",
      request_id: env["request_id"]
    })

    [200, { "Content-Type" => "application/json" }, [body]]
  end
end
```

---

## Step 1: Browser Sends Request

```http
GET /api/users HTTP/1.1
Host: localhost:9292
Authorization: Bearer token_abc123
```

---

## Step 2: Puma Receives It

Puma:

1. Accepts the TCP connection
2. Reads the HTTP request
3. Builds the `env` hash
4. Calls `app.call(env)`

```ruby
env = {
  "REQUEST_METHOD" => "GET",
  "PATH_INFO"      => "/api/users",
  "HTTP_AUTHORIZATION" => "Bearer token_abc123",
  "rack.input"     => # empty (GET has no body),
  "REMOTE_ADDR"    => "127.0.0.1",
  # ... more keys
}
```

Your app never sees raw HTTP. Puma converts it to `env`.

---

## Step 3: RequestId Middleware

```ruby
class RequestId
  def call(env)
    env["request_id"] = SecureRandom.uuid   # IN: set ID
    status, headers, body = @app.call(env)
    headers["X-Request-Id"] = env["request_id"]  # OUT: add header
    [status, headers, body]
  end
end
```

```
env["request_id"] = "abc-123-def"
→ passes to Authenticator
```

---

## Step 4: Authenticator Middleware

```ruby
class Authenticator
  def call(env)
    request = Rack::Request.new(env)
    token = request.headers["Authorization"]&.sub("Bearer ", "")

    user = USERS[token]

    unless user
      return [401, {}, ['{"error": "Unauthorized"}']]  # STOP HERE
    end

    env["current_user"] = user
    @app.call(env)
  end
end
```

Token is valid:

```
env["current_user"] = { "name" => "Alice" }
→ passes to Timer
```

If token was invalid, request **stops here**. Timer, MyApp — none of them run.

---

## Step 5: Timer Middleware

```ruby
class Timer
  def call(env)
    start = Time.now                          # IN: start clock
    status, headers, body = @app.call(env)
    headers["X-Runtime"] = (Time.now - start).to_s  # OUT: add timing
    [status, headers, body]
  end
end
```

→ passes to ContentLength → CommonLogger → MyApp

---

## Step 6: MyApp Runs

```ruby
MyApp#call(env)
```

Reads `env["current_user"]` and `env["request_id"]`.

Returns:

```ruby
[200, { "Content-Type" => "application/json" },
 ['{"message":"Hello, Alice","request_id":"abc-123-def"}']]
```

---

## Step 7: Response Flows Back Out

```
MyApp returns [200, headers, body]
  ↑
CommonLogger — writes log line
  ↑
ContentLength — sets Content-Length: 58
  ↑
Timer — adds X-Runtime: 0.003
  ↑
Authenticator — passes through
  ↑
RequestId — adds X-Request-Id: abc-123-def
  ↑
Puma sends HTTP response to browser
```

---

## Step 8: Browser Gets Response

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 58
X-Runtime: 0.003
X-Request-Id: abc-123-def

{"message":"Hello, Alice","request_id":"abc-123-def"}
```

Done.

---

## Full Diagram

```
Browser
  │
  │  GET /api/users
  ▼
Puma (builds env, calls app)
  │
  ▼
RequestId (sets request_id)
  │
  ▼
Authenticator (checks token, sets current_user)
  │
  ▼
Timer (starts clock)
  │
  ▼
ContentLength
  │
  ▼
CommonLogger
  │
  ▼
MyApp (returns JSON)
  │
  │  response goes back UP
  ▼
Puma (sends HTTP to browser)
  │
  ▼
Browser
```

---

## What Happens on Short-Circuit (401)?

Bad token:

```
Browser → Puma → RequestId (sets ID)
  → Authenticator (token invalid!)
    ← returns 401 immediately
  ← RequestId adds X-Request-Id
← Puma sends 401

Timer, ContentLength, CommonLogger, MyApp — NEVER RUN
```

Inner middleware is skipped. Outer middleware still runs its "after" code.

---

## What Happens on Error?

MyApp raises an exception:

**Without error middleware:**

```
MyApp raises
  → bubbles up through Timer (no rescue)
  → bubbles up through Authenticator (no rescue)
  → bubbles up through RequestId (no rescue)
  → Puma catches it → generic 500 page
```

**With error middleware (outermost):**

```ruby
use ErrorHandler   # first use = outermost
use RequestId
...
```

```
MyApp raises
  → bubbles up
  → ErrorHandler catches it
  ← returns clean 500 JSON response
  → Puma sends 500
```

---

## 7 Key Facts to Remember

1. **One env hash** — created by Puma, shared by all middleware and the app.
2. **Request goes IN** — outermost middleware first.
3. **Response goes OUT** — innermost app first, then back through middleware.
4. **Middleware can stop early** — short-circuit without calling the app.
5. **Middleware can change env** — add data for downstream layers.
6. **Errors bubble up** — unless middleware catches them.
7. **Rails works the same way** — `Rails.application` is a Rack app with 30+ middleware layers.

---

## Interview Questions — Complete Lifecycle

**Q: Describe a Rack request lifecycle.**  
A: Server builds env → outermost middleware runs → each layer passes to the next → app handles request → response flows back out through each middleware → server sends HTTP.

**Q: What is short-circuiting?**  
A: Middleware returns a response without calling `@app.call(env)`. Inner layers never run.

**Q: Is the same env used throughout?**  
A: Yes. One hash, passed and modified in place.

**Q: How does this relate to Rails?**  
A: Same pattern. Puma calls `Rails.application.call(env)`. Request passes through Rails' middleware stack before reaching the router and controller.

---

# Mental Model — Part 2

```
config.ru
  │
  ├── use MiddlewareA
  ├── use MiddlewareB
  └── run MyApp.new
        │
        ▼
  Request comes in
        │
        ├── Rack::Request  →  read the request easily
        ├── Your app logic
        └── Rack::Response →  build the response easily
        │
        ▼
  Response goes out
        │
        ▼
  Puma sends it to the browser
```

---

# Key Takeaways — Part 2

1. **Rack::Request** — easy way to read requests (params, headers, cookies, path)
2. **Rack::Response** — easy way to build responses (status, headers, cookies, redirects)
3. **Rack::Builder** — easy way to stack middleware (`use`, `run`, `map`)
4. **config.ru** — startup file that defines your middleware + app
5. **Middleware** — a Rack app that wraps another app (before/after/short-circuit)
6. **Middleware chain** — onion model, request goes in, response goes out
7. **Complete lifecycle** — Puma → env → middleware → app → middleware → HTTP response

---

# 🎉 Part 2 Complete

You now understand:

* ✅ Rack::Request — reading incoming requests
* ✅ Rack::Response — building outgoing responses
* ✅ Rack::Builder — composing middleware stacks
* ✅ config.ru / Rackup — starting your app
* ✅ Middleware fundamentals — the @app pattern
* ✅ Middleware chain — order and onion model
* ✅ Complete request lifecycle — end to end

**Next up in Part 3:** How Rails uses Rack, ActionDispatch, the Rails middleware stack, and writing custom Rails middleware.

---

## Progress Tracker — Rack Study Guide

| Part | Topics | Status |
|------|--------|--------|
| **Part 1** | Overview, History, Why Rack Exists, Problems Rack Solves, Rack Architecture, Rack Specification, Rack Application, Rack Environment, Hello World, Request Flow | 🟢 Complete |
| **Part 2** | Rack::Request, Rack::Response, Rack::Builder, config.ru (Rackup), Middleware Fundamentals, Middleware Chain, Complete Request Lifecycle | 🟢 Complete |
| **Part 3** | Rails + Rack, ActionDispatch, Rails Middleware Stack, Middleware Ordering, Custom Middleware, Production Examples | ⬜ Not Started |
| **Part 4** | Internal Implementation, Source Code Walkthrough, Thread Safety, Concurrency, Streaming, Hijacking, Async, HTTP/2 | ⬜ Not Started |
| **Part 5** | Performance, Security, Debugging, Logging, Common Mistakes, Best Practices, Anti-patterns | ⬜ Not Started |
| **Part 6** | Advanced Examples, Edge Cases, Comparison Tables, Interview Questions, Coding Exercises, Cheat Sheet, Summary, Resources | ⬜ Not Started |
