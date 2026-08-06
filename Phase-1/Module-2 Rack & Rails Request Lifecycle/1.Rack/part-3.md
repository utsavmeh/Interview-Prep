# Part 3 — Rails and Rack

Welcome back.

In Part 1, you learned the Rack specification: `call(env) → [status, headers, body]`.

In Part 2, you learned the helper tools (`Rack::Request`, `Rack::Response`, `Rack::Builder`) and middleware fundamentals (the onion model, short-circuiting, and the complete request lifecycle).

Now the big question:

> **How does Rails actually use Rack?**

Every time you run `rails server`, Rack is running. Every controller action, every `before_action`, every session, every cookie — all of it flows through Rack.

Part 3 connects everything you've learned to the framework you use every day.

---

# What You'll Learn in Part 3

1. **Rails + Rack** — how Rails is a Rack application
2. **ActionDispatch** — Rails' Rack layer
3. **Rails Middleware Stack** — all 20+ middleware, one by one
4. **Middleware Ordering** — why order matters and how Rails decides it
5. **Custom Middleware** — how to write and add your own in Rails
6. **Production Examples** — real middleware used at Shopify, GitLab, and similar companies

---

# 1. Rails + Rack

## Rails IS a Rack Application

This is the most important idea in Part 3:

> **Rails is not "built on top of" Rack. Rails IS a Rack application.**

What does that mean?

Remember the Rack contract from Part 1:

```ruby
call(env) → [status, headers, body]
```

Rails follows this exact contract.

---

## Proof

Open any Rails project. Look at `config.ru`:

```ruby
require_relative "config/environment"

run Rails.application
```

Two lines.

`Rails.application` is the Rack app.

Puma calls:

```ruby
Rails.application.call(env)
```

and gets back:

```ruby
[status, headers, body]
```

That's it. To Puma, Rails is just another Rack app. No different from Sinatra, Hanami, or the `HelloWorld` app we built in Part 1.

---

## What IS Rails.application?

```ruby
Rails.application
# => #<MyApp::Application ...>
```

It's an instance of your application class (defined in `config/application.rb`):

```ruby
module MyApp
  class Application < Rails::Application
    # configuration here
  end
end
```

`Rails::Application` inherits from `Rails::Engine`, which inherits from `Rails::Railtie`.

But the important thing: it responds to `call(env)`.

```ruby
Rails.application.respond_to?(:call)
# => true
```

So it's a valid Rack app.

---

## What Happens When Puma Calls Rails.application.call(env)?

Here's the short version:

```
Puma calls Rails.application.call(env)
  │
  ▼
Rails middleware stack runs (20+ middleware)
  │
  ▼
Router finds the right controller + action
  │
  ▼
Controller runs (before_action, action, after_action)
  │
  ▼
View renders (if needed)
  │
  ▼
Response goes back through middleware (in reverse)
  │
  ▼
Puma sends HTTP to browser
```

The middleware stack is the key part. Before your controller ever runs, the request passes through 20+ middleware layers.

---

## How to See This

Run this command in any Rails project:

```bash
bin/rails middleware
```

You'll see something like:

```
use ActionDispatch::HostAuthorization
use Rack::Sendfile
use ActionDispatch::Static
use ActionDispatch::Executor
use ActionDispatch::ServerTiming
use ActiveSupport::Cache::Strategy::LocalCache::Middleware
use Rack::Runtime
use Rack::MethodOverride
use ActionDispatch::RequestId
use ActionDispatch::RemoteIp
use Sprockets::Rails::QuietAssets
use Rails::Rack::Logger
use ActionDispatch::ShowExceptions
use ActionDispatch::DebugExceptions
use ActionDispatch::ActionableExceptions
use ActionDispatch::Reloader
use ActionDispatch::Callbacks
use ActiveRecord::Migration::CheckPending
use ActionDispatch::Cookies
use ActionDispatch::Session::CookieStore
use ActionDispatch::Flash
use ActionDispatch::ContentSecurityPolicy::Middleware
use ActionDispatch::PermissionsPolicy::Middleware
use Rack::Head
use Rack::ConditionalGet
use Rack::ETag
use Rack::TempfileReaper
run MyApp::Application.routes
```

That's the full stack.

Every single request goes through ALL of those middleware before reaching your controller.

---

## The Last Line is Important

```
run MyApp::Application.routes
```

At the very bottom is the **router**. The router is the actual "app" in Rack terms. It receives the request after all middleware have run, finds the right controller and action, and calls it.

---

## Mental Model

Think of Rails like this:

```
Rails.application = Middleware Stack + Router + Controllers
```

Or more precisely:

```
Rails.application.call(env)
    │
    ▼
┌──────────────────────────┐
│  Middleware 1             │
│  Middleware 2             │
│  Middleware 3             │
│  ...                      │
│  Middleware 20+           │
│                           │
│  ┌──────────────────────┐│
│  │  Router               ││
│  │  ┌──────────────────┐ ││
│  │  │  Controller       │ ││
│  │  └──────────────────┘ ││
│  └──────────────────────┘│
└──────────────────────────┘
```

The middleware stack wraps the router, which wraps the controller.

Same onion model from Part 2. Just with Rails-specific layers.

---

## Rails.application.call vs Rails.application.routes

A subtle but important difference:

- `Rails.application` — the full app **with** middleware
- `Rails.application.routes` — the router **without** middleware

```ruby
# Full middleware stack + router
Rails.application.call(env)

# Just routing, no middleware
Rails.application.routes.call(env)
```

Why does this matter?

When you test in RSpec with `get "/users"`, sometimes you're hitting the full stack, sometimes just the router. Understanding this helps you debug "why is my middleware not running in tests?"

---

## Interview Questions — Rails + Rack

**Q: Is Rails a Rack application?**
A: Yes. `Rails.application` responds to `call(env)` and returns `[status, headers, body]`. It follows the Rack specification exactly.

**Q: What does config.ru do in a Rails project?**
A: It requires the Rails environment and runs `Rails.application` as the Rack app. The server (Puma) reads this file to know what to run.

**Q: What happens before your controller runs?**
A: The request passes through 20+ middleware layers. These handle sessions, cookies, logging, security headers, static files, and more.

**Q: How do you see the middleware stack?**
A: Run `bin/rails middleware` from the project root.

---

# 2. ActionDispatch

## What is ActionDispatch?

ActionDispatch is the **Rails module that handles everything between Rack and your controllers**.

Think of it this way:

```
Rack gives you:     env hash, [status, headers, body]
Controllers need:   request object, response object, params, sessions, cookies

Who translates?     ActionDispatch
```

ActionDispatch is the translator.

---

## Where Does ActionDispatch Live?

ActionDispatch is part of the `actionpack` gem, which is one of the core Rails gems.

```
Rails
├── activerecord     (database)
├── actionpack       (controllers + dispatch)
│   ├── ActionController   (your controllers)
│   └── ActionDispatch     (Rack layer)
├── activesupport    (utilities)
├── actionview       (views)
└── ...
```

You never install ActionDispatch separately. It comes with Rails.

---

## What Does ActionDispatch Do?

ActionDispatch provides three things:

### 1. Middleware

Most of the middleware you saw in `bin/rails middleware` lives in ActionDispatch:

```ruby
ActionDispatch::Cookies
ActionDispatch::Session::CookieStore
ActionDispatch::Flash
ActionDispatch::RequestId
ActionDispatch::RemoteIp
ActionDispatch::ShowExceptions
# ... and more
```

These are the Rails-specific middleware that sit on top of Rack.

### 2. Request and Response Objects

Remember `Rack::Request` from Part 2? ActionDispatch builds on it:

```ruby
ActionDispatch::Request < Rack::Request
```

Yes, literally inherits from it.

So when you write `request.params` in a controller, you're using an `ActionDispatch::Request` that inherits from `Rack::Request` that wraps the raw `env` hash.

Same for response:

```ruby
ActionDispatch::Response
```

This builds on the Rack response concept but adds Rails features like:

- Content negotiation
- Cache headers
- Streaming

### 3. Routing

The router (`ActionDispatch::Routing`) matches URLs to controllers:

```ruby
# config/routes.rb
get "/users", to: "users#index"
```

The router is a Rack app itself. It receives `env`, finds the right controller, and calls it.

---

## The Chain

Let's trace exactly how a request goes from Rack to your controller:

```
Puma builds env
  │
  ▼
Rails.application.call(env)
  │
  ▼
Middleware stack (ActionDispatch middleware)
  │
  ▼
ActionDispatch::Routing::RouteSet#call(env)
  │
  ▼
Finds controller + action
  │
  ▼
Creates ActionDispatch::Request from env
Creates ActionDispatch::Response
  │
  ▼
Calls controller#action
  │
  ▼
Controller returns response
  │
  ▼
ActionDispatch::Response.finish → [status, headers, body]
  │
  ▼
Back through middleware
  │
  ▼
Puma sends HTTP
```

---

## ActionDispatch::Request — What It Adds Over Rack::Request

`Rack::Request` gives you basic stuff:

```ruby
request.params
request.path
request.get?
```

`ActionDispatch::Request` adds Rails-specific things:

```ruby
# Format detection
request.format        # => :json, :html, :xml
request.json?         # => true
request.html?         # => true

# UUID tracking
request.request_id    # => "abc-123-def" (set by RequestId middleware)

# IP detection (through proxies)
request.remote_ip     # => "203.0.113.1" (the real IP, not the proxy)

# Session and Flash
request.session       # => { user_id: 1 }
request.flash         # => { notice: "Saved!" }

# Content negotiation
request.accepts       # => [Mime[:html], Mime[:json]]

# Parameters (enhanced)
request.parameters    # => merged params (query + body + route)
request.path_parameters  # => { controller: "users", action: "index" }
request.query_parameters # => { page: "2" }
request.request_parameters # => { name: "Alice" } (from body)
```

### How Parameters Work in Rails vs Rack

In plain Rack:

```ruby
request = Rack::Request.new(env)
request.params
# => { "page" => "2", "name" => "Alice" }
# Only from query string and form body
```

In Rails:

```ruby
request = ActionDispatch::Request.new(env)
request.parameters
# => { "controller" => "users", "action" => "index", "page" => "2", "id" => "5" }
# Includes route params too!
```

Rails merges three sources:

```
Query params     →  ?page=2
Body params      →  name=Alice (from form or JSON)
Route params     →  /users/:id → { id: "5" }
```

into one `params` hash.

This is why `params[:id]` works in controllers even though the ID comes from the URL path, not query string.

---

## ActionDispatch::Response — What It Adds

Rails' response object adds:

```ruby
response.content_type = "application/json"
response.headers["X-Custom"] = "value"
response.status = 200
response.body = "Hello"

# Caching
response.cache_control
response.etag = "abc123"

# Cookies (through the response)
response.set_cookie("name", "value")
```

At the end, Rails calls something like:

```ruby
response.to_a
# => [200, { "Content-Type" => "application/json", ... }, ["Hello"]]
```

That's the standard Rack response array. It flows back through the middleware stack and to Puma.

---

## Interview Questions — ActionDispatch

**Q: What is ActionDispatch?**
A: The Rails module that sits between Rack and controllers. It provides middleware, request/response objects, and routing.

**Q: How does ActionDispatch::Request relate to Rack::Request?**
A: It inherits from Rack::Request. Everything Rack::Request can do, ActionDispatch::Request can do too, plus Rails extras like `format`, `remote_ip`, `session`, and merged parameters.

**Q: Where do Rails params come from?**
A: Three sources merged together: query string, request body, and route parameters.

**Q: What does ActionDispatch::Response return at the end?**
A: A standard Rack response: `[status, headers, body]`.

---

# 3. The Rails Middleware Stack — Every Middleware Explained

This is the longest section of Part 3. We'll go through every default middleware, one by one.

Understanding each one tells you **what happens to every request before your controller sees it**.

The order below is the default order for a new Rails 7+ app in development. Production may differ slightly (noted where relevant).

---

## How to Read This Section

For each middleware, I'll explain:

1. **What it does** — in plain English
2. **Why it exists** — the problem it solves
3. **When it runs** — before the app, after the app, or both
4. **Example** — what it actually does to the request or response

---

## 1. ActionDispatch::HostAuthorization

**What:** Checks that the request's `Host` header is allowed.

**Why:** Security. Without this, an attacker can send a request with a fake `Host` header like `evil.com`. If your app generates URLs using the host (like password reset links), it could create links pointing to `evil.com`.

**When:** Before the app.

**How it works:**

```ruby
# config/environments/development.rb
config.hosts << "myapp.com"
config.hosts << /.*\.myapp\.com/   # allow subdomains
```

If the request's host isn't in the allowed list:

```
Request with Host: evil.com
  → HostAuthorization checks
  → Not in allowed list
  → Returns 403 Forbidden
  → App never runs
```

**In development:** All hosts are allowed by default (so `localhost` works).

**In production:** You should configure allowed hosts.

```ruby
# config/environments/production.rb
config.hosts << "myapp.com"
config.hosts << ".myapp.com"  # subdomains
```

---

## 2. Rack::Sendfile

**What:** Lets the web server (Nginx/Apache) send files directly instead of Rails doing it.

**Why:** Performance. If Rails reads a file and sends it byte by byte, that's slow. Nginx is much faster at serving files.

**When:** After the app.

**How it works:**

When your controller does:

```ruby
send_file "/path/to/large_report.pdf"
```

Rails sets a special header:

```
X-Sendfile: /path/to/large_report.pdf
```

Rack::Sendfile sees this header and tells Nginx: "You serve this file directly." Rails doesn't waste time streaming the file.

```
Without Sendfile:
  Rails reads file → sends to Puma → sends to Nginx → sends to browser

With Sendfile:
  Rails sets header → Nginx reads file directly → sends to browser
```

Much faster for large files.

**Configuration:**

```ruby
# config/environments/production.rb
config.action_dispatch.x_sendfile_header = "X-Sendfile"         # Apache
config.action_dispatch.x_sendfile_header = "X-Accel-Redirect"   # Nginx
```

---

## 3. ActionDispatch::Static

**What:** Serves static files from the `public/` directory.

**Why:** In development, you don't have Nginx. Rails needs to serve CSS, JS, images, and other files itself.

**When:** Before the app.

**How it works:**

```
Request: GET /robots.txt
  → Static middleware checks: does public/robots.txt exist?
  → Yes → serves the file directly
  → App never runs

Request: GET /users
  → Static middleware checks: does public/users exist?
  → No → passes to the app
```

**In production:** Usually disabled because Nginx serves static files directly (much faster).

```ruby
# config/environments/production.rb
config.public_file_server.enabled = false  # Let Nginx handle it
```

---

## 4. ActionDispatch::Executor

**What:** Manages the execution environment for each request.

**Why:** Rails needs to do setup and cleanup for every request — things like autoloading code, managing database connections, and clearing caches.

**When:** Both before and after the app.

**How it works:**

```
Before:
  - Acquire database connection from pool
  - Set up autoloading context
  - Run any registered "to_run" callbacks

After:
  - Return database connection to pool
  - Run any registered "to_complete" callbacks
  - Clean up thread-local variables
```

You rarely interact with this directly. But it's critical for how Rails manages resources.

---

## 5. ActionDispatch::ServerTiming

**What:** Adds a `Server-Timing` header to the response with performance data.

**Why:** Browser DevTools can read this header and show you how long things took on the server side.

**When:** After the app.

**What it adds:**

```http
Server-Timing: miss, sql.active_record;dur=2.5, render.action_view;dur=15.3
```

Open Chrome DevTools → Network → click a request → Timing tab. You'll see these numbers.

This is disabled in production by default to avoid leaking performance data.

---

## 6. ActiveSupport::Cache::Strategy::LocalCache::Middleware

**What:** Creates a short-lived in-memory cache for each request.

**Why:** If your code reads the same cache key multiple times in one request, this avoids hitting Redis/Memcached repeatedly.

**When:** Both before and after.

**Example:**

```ruby
# In your controller
Rails.cache.read("user:1")   # hit 1: reads from Redis
Rails.cache.read("user:1")   # hit 2: reads from local memory (fast!)
Rails.cache.read("user:1")   # hit 3: local memory again
```

Without this middleware, all three reads would go to Redis.

With it, only the first read goes to Redis. The second and third read from a per-request in-memory hash.

After the request, the local cache is cleared. It only lasts for one request.

---

## 7. Rack::Runtime

**What:** Adds an `X-Runtime` header showing how long the request took.

**When:** Both (starts timer before, sets header after).

**Example:**

```http
X-Runtime: 0.054321
```

This means the request took about 54 milliseconds.

Useful for monitoring. You can log this, alert on slow requests, or display it in dashboards.

---

## 8. Rack::MethodOverride

**What:** Allows HTML forms to send PUT, PATCH, and DELETE requests.

**Why:** HTML forms only support GET and POST. There's no `<form method="PUT">`. But Rails routes use PUT, PATCH, and DELETE.

**When:** Before the app.

**How it works:**

Rails adds a hidden field to forms:

```html
<form method="POST" action="/users/1">
  <input type="hidden" name="_method" value="PATCH">
  ...
</form>
```

The browser sends a POST. But `Rack::MethodOverride` sees `_method=PATCH` and changes `env["REQUEST_METHOD"]` from `"POST"` to `"PATCH"`.

```
Browser sends: POST /users/1 with _method=PATCH
Rack::MethodOverride changes: REQUEST_METHOD = "PATCH"
Router sees: PATCH /users/1
Controller runs: users#update
```

Without this middleware, Rails' resourceful routing wouldn't work with HTML forms.

---

## 9. ActionDispatch::RequestId

**What:** Assigns a unique ID to every request.

**Why:** When something goes wrong in production, you need to trace a request across logs, services, and error trackers.

**When:** Before the app.

**How it works:**

```ruby
# Sets env["action_dispatch.request_id"]
# Also available as request.request_id in controllers

# If the client sends X-Request-Id header, Rails uses that
# Otherwise, Rails generates a new UUID
```

Response includes:

```http
X-Request-Id: 7c4b4c5a-9e3f-4b5a-8c5e-1a2b3c4d5e6f
```

**Why does it accept incoming X-Request-Id?**

In microservice architectures:

```
API Gateway → Service A → Service B → Service C
```

The gateway generates a request ID. Each service passes it along. Now you can search logs across all services with one ID.

```
[Service A] [req: 7c4b4c5a] Processing /users
[Service B] [req: 7c4b4c5a] Fetching user from DB
[Service C] [req: 7c4b4c5a] Sending email
```

One ID ties them all together.

---

## 10. ActionDispatch::RemoteIp

**What:** Figures out the real IP address of the client.

**Why:** In production, requests go through load balancers and proxies. The direct connection IP is the proxy's IP, not the user's.

**When:** Before the app.

**The problem:**

```
User (IP: 203.0.113.50)
    → CloudFlare (IP: 104.16.0.1)
    → Nginx (IP: 10.0.0.1)
    → Puma

REMOTE_ADDR = 10.0.0.1  (that's Nginx, not the user!)
```

**The solution:**

Proxies add headers:

```
X-Forwarded-For: 203.0.113.50, 104.16.0.1
```

`RemoteIp` reads this header and extracts the real client IP.

```ruby
request.remote_ip  # => "203.0.113.50"
```

**Security concern:** An attacker can fake `X-Forwarded-For`. RemoteIp is smart about this — it ignores known proxy IPs and picks the first untrusted IP.

```ruby
# Configure trusted proxies
config.action_dispatch.trusted_proxies = [
  IPAddr.new("10.0.0.0/8"),       # internal network
  IPAddr.new("172.16.0.0/12"),    # Docker
]
```

---

## 11. Rails::Rack::Logger

**What:** Logs the start and end of every request.

**When:** Both before and after.

**What you see:**

```
Started GET "/users" for 127.0.0.1 at 2024-01-15 10:30:45
Processing by UsersController#index as HTML
  User Load (1.2ms)  SELECT "users".* FROM "users"
  Rendering users/index.html.erb
  Rendered users/index.html.erb (Duration: 3.5ms)
Completed 200 OK in 15ms (Views: 5.2ms | ActiveRecord: 1.2ms)
```

This is probably the most familiar middleware output. You see it in your terminal every time you make a request in development.

It uses Rails' tagged logging to include the request ID:

```
[7c4b4c5a] Started GET "/users" ...
```

---

## 12. ActionDispatch::ShowExceptions

**What:** Catches exceptions and renders error pages.

**When:** Wraps the app (rescues errors).

**How it works:**

```ruby
def call(env)
  @app.call(env)
rescue Exception => exception
  render_exception(env, exception)
end
```

When your controller raises an error:

```
ActiveRecord::RecordNotFound → 404 page
ActionController::RoutingError → 404 page
StandardError → 500 page
```

**In development:** Shows the detailed error page with stack trace (the one with the red header).

**In production:** Shows `public/404.html` or `public/500.html`.

You can customize this with `config.exceptions_app`:

```ruby
# config/application.rb
config.exceptions_app = routes  # Use your own controller for error pages
```

---

## 13. ActionDispatch::DebugExceptions

**What:** Shows the detailed debugging page in development.

**Why:** The colorful error page with the full stack trace, source code, and request details.

**When:** Wraps the app (rescues errors).

**Difference from ShowExceptions:**

```
ShowExceptions:  "Show the user a clean error page"
DebugExceptions: "Show the developer debugging info"
```

In production, `DebugExceptions` is basically a no-op. It only shows detailed errors in development.

---

## 14. ActionDispatch::ActionableExceptions

**What:** Adds buttons to the development error page that can fix the problem.

**Example:** When you have a pending migration, the error page shows a "Run pending migrations" button. Click it, and Rails runs `rails db:migrate` for you.

**When:** Development only.

---

## 15. ActionDispatch::Reloader

**What:** Reloads your code when files change.

**Why:** In development, you change a controller and refresh the browser. You don't want to restart the server. Reloader detects file changes and reloads the modified code.

**When:** Before the app.

**How it works:**

```
Request comes in
  → Reloader checks: any files changed?
  → Yes → unload old code, load new code
  → No → do nothing
  → Pass to app
```

**In production:** Disabled. Code doesn't change while the server is running.

---

## 16. ActionDispatch::Callbacks

**What:** Runs registered `before` and `after` callbacks around each request.

**When:** Both before and after.

**How to use:**

```ruby
ActionDispatch::Callbacks.before do
  # runs before every request
end

ActionDispatch::Callbacks.after do
  # runs after every request
end
```

This is rarely used directly. It's more of a framework hook.

---

## 17. ActiveRecord::Migration::CheckPending

**What:** Checks if you have pending database migrations.

**Why:** If you pull new code with new migrations and forget to run `rails db:migrate`, your app will crash with confusing errors. This middleware catches that early and shows a clear error message.

**When:** Before the app.

```
Request comes in
  → CheckPending: any unmigrated migrations?
  → Yes → raise ActiveRecord::PendingMigrationError
  → No → pass to app
```

**In production:** Usually not included (you should run migrations during deployment, not at runtime).

---

## 18. ActionDispatch::Cookies

**What:** Reads cookies from the request and writes cookies to the response.

**When:** Both before and after.

**Before:**

```ruby
# Reads Cookie header, parses it
# Makes cookies available as request.cookies
```

**After:**

```ruby
# Takes any cookies you set in the controller
# Writes Set-Cookie headers to the response
```

**In your controller:**

```ruby
cookies[:user_name] = "Alice"
cookies.encrypted[:secret] = "hidden_value"
cookies.permanent[:remember_me] = "yes"
```

All of this is handled by the Cookies middleware. Without it, `cookies` in your controller wouldn't work.

---

## 19. ActionDispatch::Session::CookieStore

**What:** Stores the session data in an encrypted cookie.

**When:** Both before and after.

**Before:**

```ruby
# Reads the session cookie
# Decrypts it
# Makes session data available as request.session
```

**After:**

```ruby
# Takes session changes from the controller
# Encrypts them
# Writes the session cookie to the response
```

**In your controller:**

```ruby
session[:user_id] = 1
current_user_id = session[:user_id]
```

**Alternative session stores:**

```ruby
# config/initializers/session_store.rb
Rails.application.config.session_store :cookie_store     # default
Rails.application.config.session_store :cache_store       # Redis/Memcached
Rails.application.config.session_store :active_record_store  # database
```

Each store uses different middleware, but the controller API stays the same.

---

## 20. ActionDispatch::Flash

**What:** Manages flash messages (like "User saved successfully!").

**When:** Both before and after.

**How it works:**

Flash is stored in the session. But it's special — flash messages only last for one request.

```ruby
# Request 1: set flash
flash[:notice] = "Saved!"
redirect_to users_path

# Request 2: flash is available
flash[:notice]  # => "Saved!"

# Request 3: flash is gone
flash[:notice]  # => nil
```

The Flash middleware handles this lifecycle — loading flash from session before the request, and cleaning up old flash after the request.

---

## 21. ActionDispatch::ContentSecurityPolicy::Middleware

**What:** Adds `Content-Security-Policy` header to responses.

**Why:** Security. CSP tells the browser which sources of JavaScript, CSS, images, etc. are allowed. This prevents XSS (cross-site scripting) attacks.

**When:** After the app.

**Example header:**

```http
Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.example.com
```

This tells the browser: "Only run JavaScript from our domain and cdn.example.com. Block everything else."

**Configuration:**

```ruby
# config/initializers/content_security_policy.rb
Rails.application.configure_content_security_policy do |policy|
  policy.default_src :self
  policy.script_src  :self, "https://cdn.example.com"
  policy.style_src   :self, :unsafe_inline
end
```

---

## 22. ActionDispatch::PermissionsPolicy::Middleware

**What:** Adds `Permissions-Policy` header (formerly `Feature-Policy`).

**Why:** Controls which browser features your site can use — camera, microphone, geolocation, etc.

**When:** After the app.

**Example:**

```ruby
# config/initializers/permissions_policy.rb
Rails.application.configure_permissions_policy do |policy|
  policy.camera      :none
  policy.microphone  :none
  policy.geolocation :self
end
```

Response header:

```http
Permissions-Policy: camera=(), microphone=(), geolocation=(self)
```

---

## 23. Rack::Head

**What:** Handles HTTP HEAD requests.

**Why:** A HEAD request is like GET but the server should return only headers, no body. Instead of writing special logic for HEAD everywhere, this middleware handles it.

**When:** Before and after.

**How it works:**

```
HEAD /users comes in
  → Rack::Head changes REQUEST_METHOD to GET
  → App processes it as a normal GET
  → App returns [200, headers, body]
  → Rack::Head removes the body
  → Returns [200, headers, []]
```

The app doesn't even know it was a HEAD request. Clean.

---

## 24. Rack::ConditionalGet

**What:** Handles caching with `ETag` and `Last-Modified` headers.

**Why:** If the browser already has the latest version of a page, the server can return `304 Not Modified` instead of the full response. This saves bandwidth and makes the app feel faster.

**When:** After the app.

**How it works:**

```
Browser sends: If-None-Match: "abc123"
App returns:   ETag: "abc123"

ConditionalGet sees: they match!
  → Changes status to 304
  → Removes the body
  → Browser uses its cached version
```

---

## 25. Rack::ETag

**What:** Automatically generates an `ETag` header based on the response body.

**Why:** Works with `Rack::ConditionalGet` above. You don't have to calculate ETags yourself.

**When:** After the app.

**How it works:**

```ruby
# App returns body: "Hello World"
# Rack::ETag calculates MD5 of "Hello World"
# Adds header: ETag: "b10a8db164e0754105b7a99be72e3fe5"
```

---

## 26. Rack::TempfileReaper

**What:** Cleans up temporary files created during the request.

**Why:** File uploads create temp files. If they're not cleaned up, they'll fill up your disk.

**When:** After the app.

---

## Quick Reference — All Middleware

| # | Middleware | Purpose | When |
|---|-----------|---------|------|
| 1 | HostAuthorization | Block fake Host headers | Before |
| 2 | Rack::Sendfile | Let Nginx serve files | After |
| 3 | ActionDispatch::Static | Serve public/ files | Before |
| 4 | ActionDispatch::Executor | Manage execution context | Both |
| 5 | ActionDispatch::ServerTiming | Add timing headers | After |
| 6 | LocalCache::Middleware | Per-request cache | Both |
| 7 | Rack::Runtime | X-Runtime header | Both |
| 8 | Rack::MethodOverride | Support PUT/PATCH/DELETE in forms | Before |
| 9 | ActionDispatch::RequestId | Unique request IDs | Before |
| 10 | ActionDispatch::RemoteIp | Find real client IP | Before |
| 11 | Rails::Rack::Logger | Log requests | Both |
| 12 | ShowExceptions | Render error pages | Rescue |
| 13 | DebugExceptions | Dev error pages | Rescue |
| 14 | ActionableExceptions | Fix-it buttons in dev | Rescue |
| 15 | ActionDispatch::Reloader | Reload changed code | Before |
| 16 | ActionDispatch::Callbacks | Before/after hooks | Both |
| 17 | CheckPending | Check pending migrations | Before |
| 18 | ActionDispatch::Cookies | Read/write cookies | Both |
| 19 | Session::CookieStore | Read/write sessions | Both |
| 20 | ActionDispatch::Flash | Flash messages | Both |
| 21 | CSP::Middleware | Content Security Policy | After |
| 22 | PermissionsPolicy | Browser permissions | After |
| 23 | Rack::Head | Handle HEAD requests | Both |
| 24 | Rack::ConditionalGet | 304 Not Modified | After |
| 25 | Rack::ETag | Auto-generate ETags | After |
| 26 | Rack::TempfileReaper | Clean up temp files | After |

---

## Interview Questions — Rails Middleware Stack

**Q: How many middleware does a default Rails app have?**
A: Around 20-26, depending on the Rails version and configuration.

**Q: Name 5 middleware and what they do.**
A: RequestId (unique request tracking), Cookies (cookie management), RemoteIp (real client IP), MethodOverride (PUT/PATCH/DELETE from forms), ShowExceptions (error pages).

**Q: Which middleware handles sessions?**
A: `ActionDispatch::Session::CookieStore` (or whichever session store you configure).

**Q: What middleware makes `flash[:notice]` work?**
A: `ActionDispatch::Flash`.

**Q: Why does `Rack::MethodOverride` exist?**
A: HTML forms only support GET and POST. This middleware reads the `_method` parameter and overrides `REQUEST_METHOD` so Rails routing works with PUT, PATCH, and DELETE.

---

# 4. Middleware Ordering

## Why Order Matters

From Part 2, you know middleware runs in order:

```
First middleware  → sees request FIRST
Last middleware   → sees request LAST (right before the app)
```

And on the way back:

```
Last middleware   → sees response FIRST
First middleware  → sees response LAST (right before Puma sends it)
```

Getting the order wrong can cause bugs, security holes, or performance problems.

---

## How Rails Decides the Order

Rails doesn't randomly pick an order. There are specific reasons for every placement.

### Rule 1: Security First

Security middleware runs early to reject bad requests before wasting resources.

```
HostAuthorization  →  first (block bad hosts immediately)
RemoteIp           →  early (figure out who's requesting)
```

### Rule 2: Infrastructure Before Logic

Things like logging, request IDs, and timing wrap everything so they measure the full request.

```
RequestId  →  early (so logs can include the ID)
Logger     →  early (so it logs the entire request lifecycle)
Runtime    →  early (so it times everything)
```

### Rule 3: Request Transformation Before Routing

Things that change the request (like MethodOverride) must run before the router sees it.

```
MethodOverride  →  before routing
```

Why? If MethodOverride ran after routing, the router would see `POST /users/1` instead of `PATCH /users/1` and route to the wrong action.

### Rule 4: Sessions and Cookies Before Controllers

Your controller needs `session` and `cookies` to be available. So these middleware must run before the request reaches the router.

```
Cookies       →  before controller
Session       →  after Cookies (session is stored IN a cookie)
Flash         →  after Session (flash is stored IN the session)
```

### Rule 5: Response Headers After the App

Middleware that adds response headers (CSP, ETag, Server-Timing) runs after the app because the response doesn't exist until the app creates it.

```
ETag               →  after app (needs the body to calculate hash)
ConditionalGet     →  after app (needs ETag to compare)
ContentSecurityPolicy →  after app (adds header to response)
```

---

## The Full Order Logic

```
1. Security checks (reject bad requests)
2. Static file serving (avoid loading Rails for static files)
3. Execution setup (connection pool, autoloading)
4. Timing and logging (wrap everything)
5. Request transformation (MethodOverride)
6. Request identification (RequestId, RemoteIp)
7. Error handling (ShowExceptions, DebugExceptions)
8. Code reloading (Reloader — development only)
9. Data middleware (Cookies, Session, Flash)
10. Response headers (CSP, ETag, caching)
11. Cleanup (TempfileReaper)
```

---

## Manipulating the Order

Rails gives you methods to control where your middleware goes:

### Add to the End

```ruby
# config/application.rb
config.middleware.use MyMiddleware
```

Adds `MyMiddleware` at the bottom of the stack (right before the router).

### Add to the Beginning

```ruby
config.middleware.unshift MyMiddleware
```

Adds `MyMiddleware` at the top (first to see the request).

### Add Before a Specific Middleware

```ruby
config.middleware.insert_before ActionDispatch::Cookies, MyMiddleware
```

`MyMiddleware` runs just before Cookies.

### Add After a Specific Middleware

```ruby
config.middleware.insert_after ActionDispatch::RequestId, MyMiddleware
```

`MyMiddleware` runs right after RequestId.

### Remove a Middleware

```ruby
config.middleware.delete ActionDispatch::Flash
```

Removes Flash entirely. Do this if you're building an API and don't need flash messages.

### Swap a Middleware

```ruby
config.middleware.swap ActionDispatch::ShowExceptions, MyCustomExceptionHandler
```

Replaces ShowExceptions with your own.

---

## API-Only Apps

If you run `rails new myapp --api`, Rails removes middleware that HTML apps need but APIs don't:

Removed middleware in API mode:

```
ActionDispatch::Cookies          (APIs use tokens, not cookies)
ActionDispatch::Session          (APIs are stateless)
ActionDispatch::Flash            (no flash in APIs)
Rack::MethodOverride             (APIs send real PUT/PATCH/DELETE)
ActionDispatch::ContentSecurityPolicy   (no browser rendering)
ActionDispatch::PermissionsPolicy       (no browser features)
```

The API middleware stack is much shorter. Fewer layers = faster requests.

```bash
# Compare:
rails new full_app
cd full_app && bin/rails middleware | wc -l     # ~26 lines

rails new api_app --api
cd api_app && bin/rails middleware | wc -l      # ~16 lines
```

---

## Common Ordering Mistakes

### Mistake 1: Putting Auth After the Router

```ruby
# WRONG — auth runs after router already matched a route
config.middleware.use AuthMiddleware   # this goes at the bottom!
```

The router has already found the controller. If auth rejects the request, we've wasted work.

```ruby
# RIGHT — auth runs early
config.middleware.insert_before Rails::Rack::Logger, AuthMiddleware
```

### Mistake 2: Putting Rate Limiting Too Late

```ruby
# WRONG — rate limiter runs after sessions, cookies, etc.
config.middleware.use RateLimiter
```

If a request is going to be rate-limited, why load its session and cookies first?

```ruby
# RIGHT — rate limit early
config.middleware.insert_after ActionDispatch::RemoteIp, RateLimiter
```

We put it after RemoteIp because we need the real IP to rate-limit by IP.

### Mistake 3: Compression in the Wrong Place

```ruby
# WRONG — compresses before ETag
config.middleware.insert_before Rack::ETag, Rack::Deflater
```

ETag calculates a hash of the body. If the body is compressed, the ETag changes every time (depending on compression params).

```ruby
# RIGHT — compress after ETag
config.middleware.insert_after Rack::ETag, Rack::Deflater
```

---

## Interview Questions — Middleware Ordering

**Q: Why does MethodOverride run before the router?**
A: It needs to change `REQUEST_METHOD` from POST to PATCH/PUT/DELETE before the router matches routes.

**Q: Why does Cookies run before Session?**
A: The session is stored in a cookie. Session middleware needs Cookies middleware to have already parsed the cookies.

**Q: How do you add middleware before a specific middleware in Rails?**
A: `config.middleware.insert_before TargetMiddleware, YourMiddleware`

**Q: What middleware is removed in API-only mode?**
A: Cookies, Session, Flash, MethodOverride, CSP, and PermissionsPolicy — things only HTML browser apps need.

---

# 5. Custom Middleware in Rails

## When to Write Custom Middleware

Write middleware when you need to:

- Add something to **every** request or response
- Run code **before or after** every controller action
- Short-circuit requests that shouldn't reach the app
- Add headers, log data, or transform requests/responses globally

**Don't** write middleware for things that only apply to some controllers. Use `before_action` for that.

**Rule of thumb:**

```
Affects ALL requests?     → Middleware
Affects SOME controllers? → before_action / around_action
```

---

## Step 1: Create the Middleware Class

```ruby
# app/middleware/request_timer.rb

class RequestTimer
  def initialize(app)
    @app = app
  end

  def call(env)
    start_time = Process.clock_gettime(Process::CLOCK_MONOTONIC)

    status, headers, body = @app.call(env)

    duration = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start_time
    headers["X-Request-Duration"] = "#{(duration * 1000).round(2)}ms"

    [status, headers, body]
  end
end
```

This is exactly the same pattern from Part 2. Nothing changes in Rails.

1. `initialize(app)` — stores the next layer
2. `call(env)` — does work before/after calling `@app.call(env)`

---

## Step 2: Register the Middleware

```ruby
# config/application.rb

module MyApp
  class Application < Rails::Application
    config.middleware.use RequestTimer
  end
end
```

Or if you need it at a specific position:

```ruby
config.middleware.insert_after ActionDispatch::RequestId, RequestTimer
```

---

## Step 3: Verify

```bash
bin/rails middleware
```

You should see `RequestTimer` in the list.

Test it:

```bash
curl -I http://localhost:3000/users
# Look for: X-Request-Duration: 12.34ms
```

---

## Passing Options to Middleware

Your middleware can accept configuration:

```ruby
class RequestTimer
  def initialize(app, header_name: "X-Request-Duration", precision: 2)
    @app = app
    @header_name = header_name
    @precision = precision
  end

  def call(env)
    start_time = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    status, headers, body = @app.call(env)
    duration = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start_time
    headers[@header_name] = "#{(duration * 1000).round(@precision)}ms"
    [status, headers, body]
  end
end
```

Register with options:

```ruby
config.middleware.use RequestTimer, header_name: "X-Duration", precision: 4
```

---

## Environment-Specific Middleware

Sometimes you only want middleware in certain environments:

```ruby
# config/environments/development.rb
config.middleware.use DebugRequestLogger

# config/environments/production.rb
config.middleware.use RateLimiter
config.middleware.use RequestThrottler
```

Or conditionally:

```ruby
# config/application.rb
config.middleware.use RateLimiter if Rails.env.production?
```

---

## Where to Put Middleware Files

Convention (not enforced, but widely used):

```
app/
  middleware/
    request_timer.rb
    rate_limiter.rb
    json_parser.rb
```

Rails autoloads files under `app/`, so you don't need `require` statements.

Some teams put middleware in `lib/`:

```
lib/
  middleware/
    request_timer.rb
```

If you use `lib/`, you need to add it to the autoload path:

```ruby
# config/application.rb
config.autoload_paths << Rails.root.join("lib")
```

---

## Testing Custom Middleware

### Unit Test (isolated)

```ruby
# test/middleware/request_timer_test.rb

require "test_helper"

class RequestTimerTest < ActiveSupport::TestCase
  test "adds X-Request-Duration header" do
    # Create a simple inner app
    inner_app = ->(env) { [200, {}, ["OK"]] }

    # Wrap it with our middleware
    app = RequestTimer.new(inner_app)

    # Call it
    status, headers, body = app.call({})

    assert_equal 200, status
    assert headers.key?("X-Request-Duration")
    assert_match(/\d+\.\d+ms/, headers["X-Request-Duration"])
  end
end
```

### Integration Test (with Rails)

```ruby
# test/integration/request_timer_integration_test.rb

require "test_helper"

class RequestTimerIntegrationTest < ActionDispatch::IntegrationTest
  test "response includes duration header" do
    get "/users"
    assert_response :success
    assert response.headers["X-Request-Duration"].present?
  end
end
```

---

## Interview Questions — Custom Middleware

**Q: How do you add custom middleware to Rails?**
A: Create a class with `initialize(app)` and `call(env)`. Register it with `config.middleware.use` in `config/application.rb`.

**Q: Where do you put middleware files?**
A: `app/middleware/` (auto-loaded) or `lib/middleware/` (needs autoload path config).

**Q: How do you test middleware?**
A: Unit test: create a simple inner app (lambda), wrap it with your middleware, call it, check the results. Integration test: use Rails integration tests to make requests and check headers.

**Q: How do you pass options to middleware?**
A: Add keyword arguments to `initialize` after `app`, then pass them when registering: `config.middleware.use MyMiddleware, option: value`.

---

# 6. Production Examples

These are real-world middleware patterns used at companies like Shopify, GitLab, Stripe, Basecamp, and similar. Every example is something you might discuss in a senior backend interview.

---

## Example 1: Rate Limiting with Rack::Attack

**Problem:** Your API gets too many requests from one client. Need to limit requests per IP or per API key.

**Solution:** `Rack::Attack` is the most popular rate-limiting middleware for Rails.

```ruby
# Gemfile
gem "rack-attack"
```

```ruby
# config/initializers/rack_attack.rb

class Rack::Attack
  # Limit all requests to 100 per minute per IP
  throttle("requests/ip", limit: 100, period: 60.seconds) do |request|
    request.ip
  end

  # Limit login attempts to 5 per 20 seconds per email
  throttle("logins/email", limit: 5, period: 20.seconds) do |request|
    if request.path == "/login" && request.post?
      request.params["email"].to_s.downcase
    end
  end

  # Block known bad IPs
  blocklist("block bad ips") do |request|
    BAD_IPS.include?(request.ip)
  end

  # Allow all requests from the office
  safelist("allow office") do |request|
    OFFICE_IPS.include?(request.ip)
  end
end
```

```ruby
# config/application.rb
config.middleware.use Rack::Attack
```

**Where in the stack?**

```ruby
# After RemoteIp (need real IP), before everything else
config.middleware.insert_after ActionDispatch::RemoteIp, Rack::Attack
```

**What happens when rate-limited?**

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30

{"error": "Rate limit exceeded"}
```

**Why is this middleware and not a controller concern?**

Because rate limiting should happen as early as possible. If a request is going to be rejected, don't waste time loading sessions, cookies, parsing params, routing, etc.

---

## Example 2: Request Logging for Monitoring

**Problem:** You need structured JSON logs for your monitoring system (Datadog, Splunk, ELK Stack).

```ruby
# app/middleware/structured_logger.rb

class StructuredLogger
  def initialize(app)
    @app = app
  end

  def call(env)
    request = Rack::Request.new(env)
    start_time = Process.clock_gettime(Process::CLOCK_MONOTONIC)

    status, headers, body = @app.call(env)

    duration = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start_time

    log_data = {
      timestamp: Time.now.utc.iso8601,
      method: request.request_method,
      path: request.path,
      status: status,
      duration_ms: (duration * 1000).round(2),
      ip: env["action_dispatch.remote_ip"]&.to_s,
      request_id: env["action_dispatch.request_id"],
      user_agent: request.user_agent,
      content_length: headers["Content-Length"]
    }

    Rails.logger.info(log_data.to_json)

    [status, headers, body]
  end
end
```

```ruby
# config/application.rb
config.middleware.insert_after Rails::Rack::Logger, StructuredLogger
```

**Output:**

```json
{
  "timestamp": "2024-01-15T10:30:45Z",
  "method": "GET",
  "path": "/api/users",
  "status": 200,
  "duration_ms": 45.23,
  "ip": "203.0.113.50",
  "request_id": "7c4b4c5a-9e3f-4b5a",
  "user_agent": "Mozilla/5.0...",
  "content_length": "1234"
}
```

This format is easy for Datadog/Splunk to parse and index.

---

## Example 3: CORS (Cross-Origin Resource Sharing)

**Problem:** Your frontend runs on `app.example.com` but your API is on `api.example.com`. Browsers block cross-origin requests by default.

**Solution:** `rack-cors` gem.

```ruby
# Gemfile
gem "rack-cors"
```

```ruby
# config/initializers/cors.rb

Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins "https://app.example.com", "https://admin.example.com"

    resource "/api/*",
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options],
      max_age: 3600
  end
end
```

**Why `insert_before 0`?**

CORS preflight requests (OPTIONS) should be handled as fast as possible. Put the middleware first so it responds immediately without running the entire middleware stack.

**What it does:**

```
Browser sends OPTIONS /api/users (preflight)
  → CORS middleware responds with allowed origins and methods
  → 204 No Content
  → App never runs

Browser sends GET /api/users (actual request)
  → CORS middleware adds Access-Control headers
  → Request continues to app
  → Response includes: Access-Control-Allow-Origin: https://app.example.com
```

---

## Example 4: Maintenance Mode

**Problem:** You need to take the app down for maintenance. All requests should get a maintenance page.

```ruby
# app/middleware/maintenance_mode.rb

class MaintenanceMode
  MAINTENANCE_FILE = Rails.root.join("tmp", "maintenance.txt")

  def initialize(app)
    @app = app
  end

  def call(env)
    if File.exist?(MAINTENANCE_FILE)
      message = File.read(MAINTENANCE_FILE)
      body = {
        error: "Service Unavailable",
        message: message.strip
      }.to_json

      [
        503,
        {
          "Content-Type" => "application/json",
          "Retry-After" => "300"
        },
        [body]
      ]
    else
      @app.call(env)
    end
  end
end
```

```ruby
# config/application.rb
config.middleware.insert_before ActionDispatch::Static, MaintenanceMode
```

**Usage:**

```bash
# Enable maintenance mode
echo "Deploying database migration. Back in 5 minutes." > tmp/maintenance.txt

# Disable maintenance mode
rm tmp/maintenance.txt
```

**Why middleware?**

Because maintenance mode should work even if Rails can't boot (broken database, missing gems, etc.). Since this middleware runs before most of the stack, it doesn't need Rails to be fully functional.

---

## Example 5: API Authentication with JWT

**Problem:** Your API uses JWT tokens. Every request should be authenticated before reaching any controller.

```ruby
# app/middleware/jwt_authenticator.rb

class JwtAuthenticator
  SKIP_PATHS = ["/health", "/api/login", "/api/signup"].freeze

  def initialize(app)
    @app = app
  end

  def call(env)
    request = Rack::Request.new(env)

    # Skip auth for public paths
    return @app.call(env) if SKIP_PATHS.include?(request.path)

    token = extract_token(request)

    unless token
      return [
        401,
        { "Content-Type" => "application/json" },
        ['{"error": "Missing authorization token"}']
      ]
    end

    begin
      payload = JWT.decode(token, Rails.application.credentials.secret_key_base, true, algorithm: "HS256").first
      env["jwt.payload"] = payload
      env["current_user_id"] = payload["user_id"]
    rescue JWT::DecodeError => e
      return [
        401,
        { "Content-Type" => "application/json" },
        [{ error: "Invalid token", detail: e.message }.to_json]
      ]
    end

    @app.call(env)
  end

  private

  def extract_token(request)
    header = request.env["HTTP_AUTHORIZATION"]
    return nil unless header

    scheme, token = header.split(" ", 2)
    return nil unless scheme&.downcase == "bearer"

    token
  end
end
```

```ruby
# config/application.rb
config.middleware.insert_after ActionDispatch::RemoteIp, JwtAuthenticator
```

**In your controller:**

```ruby
class ApplicationController < ActionController::API
  private

  def current_user
    @current_user ||= User.find(request.env["current_user_id"])
  end
end
```

**Why middleware instead of `before_action`?**

Two reasons:

1. **Performance** — Rejected requests are stopped early, before session/cookie/routing work.
2. **Consistency** — Every request is authenticated the same way, even if a developer forgets to add `before_action` in a new controller.

**When to use `before_action` instead:**

When different controllers need different auth logic (some need admin, some need user, some are public). Middleware is all-or-nothing. Controller filters give you fine-grained control.

---

## Example 6: Request ID Propagation for Microservices

**Problem:** In a microservice architecture, you need to trace a request across services.

```ruby
# app/middleware/correlation_id.rb

class CorrelationId
  HEADER = "X-Correlation-Id".freeze

  def initialize(app)
    @app = app
  end

  def call(env)
    # Use incoming correlation ID or generate a new one
    correlation_id = env["HTTP_X_CORRELATION_ID"] || SecureRandom.uuid

    # Store it in env for the app to use
    env["correlation_id"] = correlation_id

    # Add to thread-local for logging
    Thread.current[:correlation_id] = correlation_id

    status, headers, body = @app.call(env)

    # Add to response so the caller can see it
    headers[HEADER] = correlation_id

    # Clean up
    Thread.current[:correlation_id] = nil

    [status, headers, body]
  end
end
```

Now when your service calls another service:

```ruby
class UserService
  def self.fetch_user(id)
    HTTParty.get(
      "https://user-service.internal/users/#{id}",
      headers: {
        "X-Correlation-Id" => Thread.current[:correlation_id]
      }
    )
  end
end
```

The same correlation ID flows through all services:

```
API Gateway (id: abc-123)
  → Auth Service (id: abc-123)
  → User Service (id: abc-123)
  → Notification Service (id: abc-123)
```

Search your logs with one ID, find all related log entries across all services.

---

## Example 7: Health Check Endpoint

**Problem:** Load balancers and Kubernetes need a `/health` endpoint to check if your app is alive. This should be fast and bypass most middleware.

```ruby
# app/middleware/health_check.rb

class HealthCheck
  def initialize(app)
    @app = app
  end

  def call(env)
    if env["PATH_INFO"] == "/health"
      [
        200,
        { "Content-Type" => "application/json" },
        ['{"status": "ok"}']
      ]
    else
      @app.call(env)
    end
  end
end
```

```ruby
# config/application.rb
# Put it first — health checks should be FAST
config.middleware.unshift HealthCheck
```

**Why middleware?**

1. **Speed** — No routing, no controller, no database. Just return 200.
2. **Reliability** — Works even if the router or database is broken.
3. **No logging noise** — If you put it before the logger, health checks don't fill up your logs.

**Advanced version with dependency checks:**

```ruby
class HealthCheck
  def initialize(app)
    @app = app
  end

  def call(env)
    if env["PATH_INFO"] == "/health"
      health = check_health
      status = health[:healthy] ? 200 : 503

      [status, { "Content-Type" => "application/json" }, [health.to_json]]
    else
      @app.call(env)
    end
  end

  private

  def check_health
    checks = {}

    # Check database
    begin
      ActiveRecord::Base.connection.execute("SELECT 1")
      checks[:database] = "ok"
    rescue => e
      checks[:database] = "error: #{e.message}"
    end

    # Check Redis
    begin
      Redis.current.ping
      checks[:redis] = "ok"
    rescue => e
      checks[:redis] = "error: #{e.message}"
    end

    {
      healthy: checks.values.all? { |v| v == "ok" },
      checks: checks,
      timestamp: Time.now.utc.iso8601
    }
  end
end
```

Response:

```json
{
  "healthy": true,
  "checks": {
    "database": "ok",
    "redis": "ok"
  },
  "timestamp": "2024-01-15T10:30:45Z"
}
```

---

## Example 8: Response Compression

**Problem:** Large API responses are slow to download. Need to compress them.

```ruby
# config/application.rb
config.middleware.insert_after Rack::ETag, Rack::Deflater
```

That's it. `Rack::Deflater` comes with Rack.

**What it does:**

```
Client sends: Accept-Encoding: gzip
App returns: 50KB JSON body
Rack::Deflater compresses: 50KB → 8KB
Client receives: Content-Encoding: gzip, 8KB body
Client decompresses automatically
```

**Why after ETag?**

ETag should be calculated on the uncompressed body. If you compress first, the ETag changes even when the content hasn't changed.

---

## Example 9: Multi-Tenant Isolation

**Problem:** Your SaaS app serves multiple companies. Each request needs to know which tenant it belongs to.

```ruby
# app/middleware/tenant_identifier.rb

class TenantIdentifier
  def initialize(app)
    @app = app
  end

  def call(env)
    request = Rack::Request.new(env)

    # Determine tenant from subdomain
    # shop1.myapp.com → tenant = "shop1"
    subdomain = request.host.split(".").first
    tenant = Tenant.find_by(subdomain: subdomain)

    unless tenant
      return [
        404,
        { "Content-Type" => "application/json" },
        ['{"error": "Tenant not found"}']
      ]
    end

    # Store tenant in env
    env["current_tenant"] = tenant

    # Switch database connection to tenant's schema
    Apartment::Tenant.switch!(tenant.schema_name)

    status, headers, body = @app.call(env)

    # Reset to default schema
    Apartment::Tenant.reset

    [status, headers, body]
  end
end
```

```ruby
config.middleware.insert_after ActionDispatch::RemoteIp, TenantIdentifier
```

**Why middleware?**

Every single request needs tenant identification. It must happen before any database query. Middleware is the right place.

---

## When NOT to Use Middleware

Not everything belongs in middleware. Here are cases where middleware is overkill:

| Situation | Use Instead |
|-----------|-------------|
| Auth for specific controllers only | `before_action` |
| Logging for specific actions | Controller callbacks |
| Transforming params for one endpoint | Controller code |
| Adding headers to specific responses | `after_action` |
| Business logic | Service objects |

**Rule of thumb:**

```
Middleware = cross-cutting concerns (every request)
Controller = specific concerns (some requests)
```

---

## Interview Questions — Production Examples

**Q: How would you implement rate limiting in Rails?**
A: Use `Rack::Attack` middleware. Configure throttle rules based on IP or API key. Insert it after `RemoteIp` middleware so it has the real client IP. Rate-limited requests get a 429 response without hitting the app.

**Q: Where should a health check endpoint live?**
A: As middleware, inserted first in the stack with `config.middleware.unshift`. This makes it fast and reliable — it works even if the database or router is broken.

**Q: How would you handle CORS in a Rails API?**
A: Use the `rack-cors` gem. Insert it at position 0 (first middleware) so preflight OPTIONS requests are handled immediately without running the full stack.

**Q: How do you trace requests across microservices?**
A: Use a correlation ID middleware. Accept an incoming `X-Correlation-Id` header (or generate one). Store it in `env` and `Thread.current`. Pass it along when calling other services. Include it in logs.

**Q: When should you use middleware vs controller callbacks?**
A: Middleware for cross-cutting concerns that affect every request (auth, logging, rate limiting). Controller callbacks (`before_action`, etc.) for logic that applies to specific controllers or actions.

**Q: How would you add compression to a Rails API?**
A: Add `Rack::Deflater` to the middleware stack after `Rack::ETag`. It automatically gzips responses when the client sends `Accept-Encoding: gzip`.

---

# Mental Model — Part 3

```
Rails.application.call(env)
  │
  ▼
┌────────────────────────────────────────┐
│  Security (HostAuthorization)          │
│  Performance (Sendfile, Static)        │
│  Infrastructure (Executor, Runtime)    │
│  Identification (RequestId, RemoteIp)  │
│  Logging (Logger)                      │
│  Error Handling (ShowExceptions)       │
│  Data (Cookies, Session, Flash)        │
│  Headers (CSP, ETag, ConditionalGet)   │
│  Cleanup (TempfileReaper)              │
│                                        │
│  Your Custom Middleware                │
│                                        │
│  ┌──────────────────────────────────┐  │
│  │  ActionDispatch::Routing         │  │
│  │  (Router → Controller → View)    │  │
│  └──────────────────────────────────┘  │
└────────────────────────────────────────┘
  │
  ▼
[status, headers, body] → Puma → Browser
```

---

# Key Takeaways — Part 3

1. **Rails IS a Rack app** — `Rails.application.call(env)` follows the exact Rack contract.
2. **ActionDispatch** — translates between Rack's raw `env` and Rails' rich request/response objects. It provides middleware, routing, and request/response classes.
3. **The middleware stack** — 20+ layers that handle security, sessions, cookies, logging, error handling, and more — all before your controller runs.
4. **Order matters** — security first, infrastructure next, data loading before controllers, response headers after the app.
5. **Custom middleware** — same pattern as Part 2 (`initialize(app)` + `call(env)`), registered with `config.middleware.use`.
6. **Production patterns** — rate limiting, CORS, health checks, JWT auth, correlation IDs, compression, and multi-tenancy are all solved with middleware.

---

# 🎉 Part 3 Complete

You now understand:

* ✅ Rails + Rack — how Rails is a Rack application
* ✅ ActionDispatch — Rails' Rack layer (middleware + request/response + routing)
* ✅ Rails Middleware Stack — all 26 default middleware explained
* ✅ Middleware Ordering — why order matters and how to control it
* ✅ Custom Middleware — how to write, register, configure, and test
* ✅ Production Examples — rate limiting, CORS, health checks, JWT auth, structured logging, maintenance mode, correlation IDs, compression, multi-tenancy

**Next up in Part 4:** Internal Implementation, Rack Source Code Walkthrough, Thread Safety, Concurrency, Streaming, Hijacking, Async, and HTTP/2 Considerations.

---

## Progress Tracker — Rack Study Guide

| Part | Topics | Status |
|------|--------|--------|
| **Part 1** | Overview, History, Why Rack Exists, Problems Rack Solves, Rack Architecture, Rack Specification, Rack Application, Rack Environment, Hello World, Request Flow | 🟢 Complete |
| **Part 2** | Rack::Request, Rack::Response, Rack::Builder, config.ru (Rackup), Middleware Fundamentals, Middleware Chain, Complete Request Lifecycle | 🟢 Complete |
| **Part 3** | Rails + Rack, ActionDispatch, Rails Middleware Stack, Middleware Ordering, Custom Middleware, Production Examples | 🟢 Complete |
| **Part 4** | Internal Implementation, Source Code Walkthrough, Thread Safety, Concurrency, Streaming, Hijacking, Async, HTTP/2 | ⬜ Not Started |
| **Part 5** | Performance, Security, Debugging, Logging, Common Mistakes, Best Practices, Anti-patterns | ⬜ Not Started |
| **Part 6** | Advanced Examples, Edge Cases, Comparison Tables, Interview Questions, Coding Exercises, Cheat Sheet, Summary, Resources | ⬜ Not Started |
