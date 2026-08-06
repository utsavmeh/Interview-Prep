# Part 5 — Performance, Security, and Best Practices

Welcome back.

In Part 1, you learned the Rack specification.

In Part 2, you learned middleware and the request lifecycle.

In Part 3, you learned how Rails uses Rack — ActionDispatch, the middleware stack, and production patterns.

In Part 4, you learned Rack internals — source code, thread safety, concurrency, streaming, hijacking, and async.

Now Part 5. This is the part that makes you interview-ready.

Senior engineers don't just know how Rack works. They know how to make it **fast**, keep it **secure**, **debug** it when things go wrong, and avoid **common mistakes** that trip up less experienced developers.

Every section in Part 5 is something you might be asked about in a senior backend interview.

---

# What You'll Learn in Part 5

1. **Performance** — making Rack and middleware fast
2. **Security** — protecting your app at the Rack level
3. **Debugging** — finding and fixing Rack-level problems
4. **Logging** — structured, useful logging through middleware
5. **Common Mistakes** — bugs that even experienced developers make
6. **Best Practices** — rules to follow when writing middleware
7. **Anti-patterns** — things to avoid

---

# 1. Performance

## Why Performance at the Rack Level Matters

Every single request passes through the middleware stack. If one middleware is slow, **every request** is slow.

```
20 middleware × 1ms each = 20ms overhead before your controller runs
20 middleware × 5ms each = 100ms overhead (that's noticeable!)
```

Middleware overhead is invisible to most developers. They look at controller and database times, but forget that 20+ layers run before and after their code.

---

## Measuring Middleware Performance

### The X-Runtime Header

Rails adds this by default (via `Rack::Runtime`):

```http
X-Runtime: 0.054321
```

This measures the total time inside the Rack stack. But it doesn't tell you which middleware is slow.

### Profiling Individual Middleware

Add a simple profiler middleware at the top of the stack:

```ruby
# app/middleware/middleware_profiler.rb

class MiddlewareProfiler
  def initialize(app)
    @app = app
  end

  def call(env)
    env["middleware_timings"] = []

    start = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    status, headers, body = @app.call(env)
    total = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start

    timings = env["middleware_timings"]
    Rails.logger.info "Middleware timings: #{timings.inspect}"
    Rails.logger.info "Total middleware time: #{(total * 1000).round(2)}ms"

    [status, headers, body]
  end
end
```

Or use `rack-mini-profiler` for a visual breakdown:

```ruby
# Gemfile
gem "rack-mini-profiler"
```

This shows a badge on every page with detailed timing, including middleware.

### Rails Server Timing

In Part 3 we mentioned `ActionDispatch::ServerTiming`. It adds timing data to response headers:

```http
Server-Timing: sql.active_record;dur=2.5, render.action_view;dur=15.3
```

Chrome DevTools shows this in the Network → Timing tab.

---

## Performance Optimization Techniques

### 1. Short-Circuit Early

If a request should be rejected, do it as early as possible:

```ruby
# SLOW — loads sessions, cookies, does routing, THEN rejects
class SlowAuth
  def initialize(app)
    @app = app
  end

  def call(env)
    status, headers, body = @app.call(env)

    # Too late! All the work is already done
    if env["current_user"].nil?
      return [401, {}, ["Unauthorized"]]
    end

    [status, headers, body]
  end
end
```

```ruby
# FAST — rejects immediately, nothing else runs
class FastAuth
  def initialize(app)
    @app = app
  end

  def call(env)
    unless valid_token?(env)
      return [401, { "Content-Type" => "application/json" },
              ['{"error":"Unauthorized"}']]
    end

    @app.call(env)
  end

  private

  def valid_token?(env)
    token = env["HTTP_AUTHORIZATION"]&.sub("Bearer ", "")
    token && TokenStore.valid?(token)
  end
end
```

**Savings:** All downstream middleware and the entire app are skipped. Could save 50-200ms per rejected request.

---

### 2. Skip Middleware for Certain Paths

Not every middleware needs to run for every request. Health checks don't need sessions. Static files don't need CSRF protection.

```ruby
class ConditionalMiddleware
  SKIP_PATHS = ["/health", "/ping", "/assets"].freeze

  def initialize(app)
    @app = app
  end

  def call(env)
    # Skip this middleware for certain paths
    if SKIP_PATHS.any? { |path| env["PATH_INFO"].start_with?(path) }
      return @app.call(env)
    end

    # Normal middleware logic for everything else
    do_expensive_work(env)
    @app.call(env)
  end
end
```

This is how Shopify handles their massive middleware stack. Not every middleware runs for every request.

---

### 3. Cache Expensive Computations

If middleware does something expensive (like parsing a JWT or looking up a user), cache the result:

```ruby
class CachedAuth
  def initialize(app)
    @app = app
    @cache = {}                    # careful: need thread safety!
    @mutex = Mutex.new
  end

  def call(env)
    token = env["HTTP_AUTHORIZATION"]&.sub("Bearer ", "")
    return @app.call(env) unless token

    user = lookup_user(token)
    env["current_user"] = user

    @app.call(env)
  end

  private

  def lookup_user(token)
    @mutex.synchronize do
      @cache[token] ||= begin
        # Expensive operation: decode JWT, hit database
        payload = JWT.decode(token, secret).first
        User.find(payload["user_id"])
      end
    end
  end
end
```

**Better approach:** Use Rails cache instead of a manual hash:

```ruby
def lookup_user(token)
  Rails.cache.fetch("auth:#{token}", expires_in: 5.minutes) do
    payload = JWT.decode(token, secret).first
    User.find(payload["user_id"])
  end
end
```

Rails cache is thread-safe and handles expiration for you.

---

### 4. Avoid Unnecessary Object Creation

Every object Ruby creates must be garbage collected. In middleware that runs on every request, this adds up.

```ruby
# SLOW — creates new Rack::Request on every call
def call(env)
  request = Rack::Request.new(env)   # new object
  path = request.path                 # could just read env directly
  @app.call(env)
end
```

```ruby
# FASTER — read env directly when you only need one thing
def call(env)
  path = env["PATH_INFO"]    # no object creation
  @app.call(env)
end
```

If you only need the path, don't create a whole `Rack::Request` object. Use `env` directly.

If you need many things from the request, then `Rack::Request` is worth it because it's more readable.

---

### 5. Freeze Strings

In Ruby, every string literal creates a new object:

```ruby
# Creates a new string object every call
def call(env)
  headers["X-Custom"] = "my-value"   # new string each time
  @app.call(env)
end
```

```ruby
# Reuses the same frozen string
HEADER_VALUE = "my-value".freeze

def call(env)
  headers["X-Custom"] = HEADER_VALUE   # same object, no allocation
  @app.call(env)
end
```

Or add this to the top of your file:

```ruby
# frozen_string_literal: true
```

This freezes all string literals in the file automatically.

At Shopify's scale, this kind of optimization saves real memory.

---

### 6. Remove Unused Middleware

Every middleware adds overhead, even if it does almost nothing. The `call` method, object allocation, and method dispatch all cost time.

Check if you need everything:

```bash
bin/rails middleware
```

**API-only apps** should remove HTML-related middleware:

```ruby
# config/application.rb (API app)
config.middleware.delete ActionDispatch::Cookies
config.middleware.delete ActionDispatch::Session::CookieStore
config.middleware.delete ActionDispatch::Flash
config.middleware.delete Rack::MethodOverride
```

**Apps that don't use flash:**

```ruby
config.middleware.delete ActionDispatch::Flash
```

**Apps behind a CDN that handles static files:**

```ruby
config.middleware.delete ActionDispatch::Static
```

Each removed middleware saves a tiny bit of time, but it adds up across millions of requests.

---

### 7. Use Rack::Deflater for Compression

Adding response compression is one of the easiest performance wins:

```ruby
# config/application.rb
config.middleware.insert_after Rack::ETag, Rack::Deflater
```

Typical savings:

```
JSON response: 50KB → 8KB (84% smaller)
HTML response: 100KB → 15KB (85% smaller)
```

Less data to send = faster response for the user.

**Caveat:** Don't compress already-compressed content (images, videos, zipped files). Rack::Deflater is smart enough to skip these based on Content-Type.

---

### 8. Connection Pool Sizing

From Part 4, you know threads need database connections. Undersized pools cause waiting:

```ruby
# config/database.yml
production:
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
```

**Rule:**

```
pool >= threads per Puma worker
```

If pool is too small, threads queue up waiting for a connection. Your middleware stack still runs, but the app hangs at the database call.

Monitor this with:

```ruby
ActiveRecord::Base.connection_pool.stat
# => { size: 5, connections: 5, busy: 3, dead: 0, idle: 2, waiting: 0, checkout_timeout: 5.0 }
```

If `waiting > 0` frequently, increase your pool.

---

### Performance Summary Table

| Technique | Savings | Effort |
|-----------|---------|--------|
| Short-circuit early | 50-200ms per rejected request | Low |
| Skip middleware for paths | 5-20ms for health checks | Low |
| Cache expensive ops | 10-100ms per cached hit | Medium |
| Avoid object creation | Microseconds per request | Low |
| Freeze strings | Memory savings | Low |
| Remove unused middleware | 1-5ms total | Low |
| Rack::Deflater | 80%+ bandwidth reduction | Very low |
| Connection pool sizing | Prevents timeouts | Low |

---

## Interview Questions — Performance

**Q: How do you measure middleware performance?**
A: Use `Rack::Runtime` for total time, `rack-mini-profiler` for per-middleware breakdown, or custom profiling middleware that logs individual timings. Browser DevTools shows Server-Timing headers.

**Q: What's the most impactful middleware performance optimization?**
A: Short-circuiting early. If a request should be rejected (bad auth, rate limited, maintenance mode), return immediately without running downstream middleware and the app.

**Q: How do you reduce middleware overhead in an API-only Rails app?**
A: Remove middleware you don't need: Cookies, Session, Flash, MethodOverride. Use `rails new --api` or manually delete them with `config.middleware.delete`.

**Q: Should you use Rack::Deflater?**
A: Yes, for most apps. It compresses responses by 80%+. Insert it after `Rack::ETag`. Don't use it if your reverse proxy (Nginx) already handles compression — doing it twice wastes CPU.

---

# 2. Security

## Why Security at the Rack Level?

The Rack middleware stack is the **first code that touches every request**. It's the perfect place to enforce security because:

1. Security checks run before any application logic
2. You can't accidentally bypass them (unlike controller-level checks)
3. One middleware protects the entire app

Many attacks target the HTTP layer — and that's exactly where Rack lives.

---

## Security Threats and Their Middleware Solutions

### Threat 1: Host Header Injection

**Attack:** Attacker sends a request with a fake `Host` header:

```http
GET /password-reset HTTP/1.1
Host: evil.com
```

If your app generates a password reset link using `request.host`:

```ruby
reset_link = "https://#{request.host}/reset?token=abc123"
# => "https://evil.com/reset?token=abc123"
```

The user receives an email with a link to `evil.com`. They click it, attacker steals the token.

**Solution:** `ActionDispatch::HostAuthorization` (covered in Part 3)

```ruby
# config/environments/production.rb
config.hosts << "myapp.com"
config.hosts << ".myapp.com"  # subdomains
```

Requests with unrecognized hosts get a `403 Forbidden`.

---

### Threat 2: IP Spoofing

**Attack:** Attacker sends a fake `X-Forwarded-For` header to bypass IP-based restrictions:

```http
GET /admin HTTP/1.1
X-Forwarded-For: 10.0.0.1
```

If your app trusts this header blindly, the attacker appears to come from an internal IP.

**Solution:** `ActionDispatch::RemoteIp` (covered in Part 3)

```ruby
# config/application.rb
config.action_dispatch.trusted_proxies = [
  IPAddr.new("10.0.0.0/8"),
  IPAddr.new("172.16.0.0/12"),
]
```

RemoteIp is smart — it strips known proxy IPs and uses the first untrusted IP in the chain.

---

### Threat 3: HTTP Method Tampering

**Attack:** Attacker crafts a request with `_method=DELETE` to delete resources they shouldn't:

```http
POST /admin/users/1
Content-Type: application/x-www-form-urlencoded

_method=DELETE
```

`Rack::MethodOverride` converts this to `DELETE /admin/users/1`.

**Solution:** This isn't really a Rack vulnerability — it's an authorization issue. Your app should check permissions regardless of how the HTTP method was set. But be aware that `MethodOverride` only works with POST requests (by design). An attacker can't override a GET request.

For API-only apps, remove `MethodOverride` entirely:

```ruby
config.middleware.delete Rack::MethodOverride
```

APIs send real PUT/PATCH/DELETE requests. They don't need the form workaround.

---

### Threat 4: Denial of Service (DoS)

**Attack:** Attacker floods your server with thousands of requests per second.

**Solution:** Rate limiting with `Rack::Attack` (covered in Part 3):

```ruby
class Rack::Attack
  throttle("req/ip", limit: 300, period: 5.minutes) do |req|
    req.ip
  end
end
```

Also useful: blocking known bad actors:

```ruby
Rack::Attack.blocklist("block bad IPs") do |req|
  BAD_IPS.include?(req.ip)
end
```

**Place in the stack:** After `RemoteIp` (need real IP), before everything else (reject early).

---

### Threat 5: Large Request Bodies

**Attack:** Attacker sends an enormous request body to exhaust server memory:

```http
POST /upload
Content-Length: 999999999999
```

**Solution:** Limit request body size:

```ruby
# Custom middleware
class RequestSizeLimiter
  MAX_BODY_SIZE = 10 * 1024 * 1024  # 10MB

  def initialize(app)
    @app = app
  end

  def call(env)
    content_length = env["CONTENT_LENGTH"].to_i

    if content_length > MAX_BODY_SIZE
      return [
        413,
        { "Content-Type" => "application/json" },
        ['{"error": "Request body too large"}']
      ]
    end

    @app.call(env)
  end
end
```

Nginx also has this:

```nginx
client_max_body_size 10m;
```

Best to limit at both layers — Nginx catches it first, middleware catches anything that slips through.

---

### Threat 6: Slow Client Attacks (Slowloris)

**Attack:** Attacker opens many connections but sends data very slowly. Each connection ties up a server thread. Eventually, all threads are busy with slow attackers and real users can't connect.

**Solution:** This is handled at the server level, not middleware:

```ruby
# config/puma.rb
first_data_timeout 30  # close connections that don't send data within 30s
persistent_timeout 20  # close idle keep-alive connections after 20s
```

Nginx is even better at this:

```nginx
client_body_timeout 10s;
client_header_timeout 10s;
send_timeout 10s;
```

---

### Threat 7: Missing Security Headers

**Attack:** Without proper security headers, browsers allow XSS, clickjacking, and other attacks.

**Solution:** Use middleware to add security headers:

```ruby
class SecurityHeaders
  def initialize(app)
    @app = app
  end

  def call(env)
    status, headers, body = @app.call(env)

    # Prevent XSS
    headers["X-Content-Type-Options"] = "nosniff"

    # Prevent clickjacking
    headers["X-Frame-Options"] = "SAMEORIGIN"

    # Enable browser XSS filter
    headers["X-XSS-Protection"] = "1; mode=block"

    # Control referrer information
    headers["Referrer-Policy"] = "strict-origin-when-cross-origin"

    # Force HTTPS
    headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"

    [status, headers, body]
  end
end
```

Rails sets some of these by default. The `secure_headers` gem adds more:

```ruby
# Gemfile
gem "secure_headers"
```

---

### Threat 8: Session Fixation

**Attack:** Attacker sets a session ID, tricks the user into logging in with that session, then uses the same session ID to access the user's account.

**Solution:** Rails' session middleware regenerates the session ID on login:

```ruby
# In your login controller
reset_session  # destroys old session, creates new one
session[:user_id] = user.id
```

The `ActionDispatch::Session::CookieStore` middleware handles the cookie management. The encrypted cookie prevents attackers from reading or tampering with session data.

---

### Threat 9: Cookie Tampering

**Attack:** Attacker modifies cookie values to change their session data, pretend to be another user, or escalate privileges.

**Solution:** Rails' cookie middleware uses encryption and signing:

```ruby
cookies.signed[:user_id] = 1     # signed — tamper-proof
cookies.encrypted[:secret] = "x"  # encrypted — can't even read it
```

The `ActionDispatch::Cookies` middleware handles this. The secret key in `credentials.yml.enc` is used for signing/encryption.

**Important:** Never store sensitive data in unsigned cookies. Always use `signed` or `encrypted`.

---

### Threat 10: Information Leakage in Error Pages

**Attack:** Detailed error pages in production reveal stack traces, file paths, gem versions, and database schema. Attackers use this information to find vulnerabilities.

**Solution:** `ActionDispatch::ShowExceptions` serves clean error pages in production:

```ruby
# config/environments/production.rb
config.consider_all_requests_local = false  # don't show debug info
```

`DebugExceptions` only shows detailed errors in development.

Also remove `Rack::ShowExceptions` in production — it's a development tool.

---

## Security Checklist

| Threat | Middleware/Solution | Default in Rails? |
|--------|-------------------|-------------------|
| Host injection | `HostAuthorization` | ✅ Yes |
| IP spoofing | `RemoteIp` with trusted proxies | ✅ Yes |
| DoS/DDoS | `Rack::Attack` | ❌ Add yourself |
| Large bodies | Size limiter / Nginx | ❌ Add yourself |
| Slow clients | Puma/Nginx timeouts | Partial |
| Missing headers | `SecurityHeaders` / `secure_headers` | Partial |
| XSS | `ContentSecurityPolicy` | ✅ Yes (Rails 6+) |
| Clickjacking | `X-Frame-Options` | ✅ Yes |
| Session fixation | `reset_session` + encrypted cookies | ✅ Yes |
| Cookie tampering | Signed/encrypted cookies | ✅ Yes |
| Info leakage | `ShowExceptions` in production mode | ✅ Yes |
| CORS misconfig | `rack-cors` with strict origins | ❌ Add yourself |

---

## Interview Questions — Security

**Q: How does Rails protect against host header injection?**
A: `ActionDispatch::HostAuthorization` middleware checks the `Host` header against a whitelist. Unrecognized hosts get a `403`. Configure with `config.hosts`.

**Q: How does RemoteIp prevent IP spoofing?**
A: It doesn't blindly trust `X-Forwarded-For`. It strips known proxy IPs and picks the first untrusted IP. You configure trusted proxies explicitly.

**Q: How would you protect a Rails API against DoS?**
A: Use `Rack::Attack` middleware with throttle rules. Place it early in the stack (after RemoteIp). Combine with Nginx rate limiting for defense in depth.

**Q: Why should you add security headers at the middleware level?**
A: Because middleware runs on every response. You can't forget to add them. A missing security header on even one endpoint is a vulnerability.

**Q: What's the difference between signed and encrypted cookies?**
A: Signed cookies prevent tampering — the value is readable but can't be changed without invalidating the signature. Encrypted cookies prevent both reading and tampering — the value is unreadable without the secret key.

---

# 3. Debugging

## Why Debugging at the Rack Level is Hard

Most Rails developers debug at the controller level. They use `byebug`, `puts`, or `Rails.logger`.

But some bugs live in the middleware stack:

- "Why is my session empty?"
- "Why is the request ID missing?"
- "Why is my IP address wrong?"
- "Why is my middleware not running?"
- "Why does this request get a 500 before reaching my controller?"

These bugs are invisible if you only look at controllers.

---

## Debugging Tools

### Tool 1: `bin/rails middleware`

Shows the full middleware stack in order:

```bash
bin/rails middleware
```

Output:

```
use ActionDispatch::HostAuthorization
use Rack::Sendfile
use ActionDispatch::Static
...
use ActionDispatch::Cookies
use ActionDispatch::Session::CookieStore
...
run MyApp::Application.routes
```

**Use when:** You want to know what middleware is in the stack and in what order.

---

### Tool 2: Debugging Middleware (Interceptor)

Drop this in your stack to see exactly what's happening:

```ruby
# app/middleware/debug_middleware.rb

class DebugMiddleware
  def initialize(app, label: "DEBUG")
    @app = app
    @label = label
  end

  def call(env)
    request = Rack::Request.new(env)

    Rails.logger.debug "[#{@label}] BEFORE: #{request.request_method} #{request.path}"
    Rails.logger.debug "[#{@label}] Headers: #{env.select { |k,v| k.start_with?('HTTP_') }}"
    Rails.logger.debug "[#{@label}] Env keys added: #{env.keys.select { |k| k.include?('.') }}"

    status, headers, body = @app.call(env)

    Rails.logger.debug "[#{@label}] AFTER: status=#{status}"
    Rails.logger.debug "[#{@label}] Response headers: #{headers}"

    [status, headers, body]
  end
end
```

Insert it around the suspicious middleware:

```ruby
# config/application.rb

# Debug what happens before and after Cookies middleware
config.middleware.insert_before ActionDispatch::Cookies, DebugMiddleware, label: "BEFORE_COOKIES"
config.middleware.insert_after ActionDispatch::Cookies, DebugMiddleware, label: "AFTER_COOKIES"
```

Now you can see exactly what the Cookies middleware adds to `env` and headers.

---

### Tool 3: Rack::Lint

From Part 4 — validates your middleware follows the spec:

```ruby
# Add temporarily for debugging
config.middleware.insert_before 0, Rack::Lint
config.middleware.use Rack::Lint  # also at the end
```

If any middleware breaks the spec (wrong status type, missing headers, body doesn't respond to `each`), Lint catches it.

---

### Tool 4: Reading env at Any Point

Print the entire env hash to see what's been set:

```ruby
class EnvDumper
  def initialize(app)
    @app = app
  end

  def call(env)
    # Print all env keys and values
    env.sort.each do |key, value|
      puts "#{key}: #{value.inspect}" unless value.is_a?(IO)
    end

    @app.call(env)
  end
end
```

This shows you every key in the env hash — CGI variables, Rack variables, HTTP headers, and anything middleware has added.

---

### Tool 5: curl for Isolating Issues

Use curl to send precise requests and see raw responses:

```bash
# Basic request
curl -v http://localhost:3000/users

# See response headers
curl -I http://localhost:3000/users

# Send specific headers
curl -H "Authorization: Bearer abc123" http://localhost:3000/api/users

# Send POST with body
curl -X POST -H "Content-Type: application/json" \
     -d '{"name":"Alice"}' http://localhost:3000/users

# Follow redirects
curl -L http://localhost:3000/old-path
```

The `-v` flag shows the full HTTP conversation — request headers, response headers, status code, body. This helps you see exactly what Rack is sending back.

---

### Tool 6: Rack::Utils for Manual Testing

In Rails console, you can manually test Rack utilities:

```ruby
# Parse a query string
Rack::Utils.parse_query("page=2&sort=name")
# => {"page" => "2", "sort" => "name"}

# Build a mock env
env = Rack::MockRequest.env_for("/users?page=2", method: "GET")

# Call your app directly
status, headers, body = Rails.application.call(env)
puts status    # => 200
puts headers   # => {"Content-Type" => "text/html", ...}
body.each { |chunk| puts chunk }
```

This lets you test the full middleware stack from the console, without running a server.

---

## Common Debugging Scenarios

### Scenario 1: "My middleware isn't running"

**Symptoms:** You added middleware but it has no effect.

**Debugging steps:**

```bash
# 1. Check if it's in the stack
bin/rails middleware | grep MyMiddleware
```

If it's not listed, you didn't register it properly.

```ruby
# 2. Check your registration
# config/application.rb
config.middleware.use MyMiddleware   # Did you put it here?
```

```ruby
# 3. Add a puts to confirm
class MyMiddleware
  def call(env)
    puts "=== MyMiddleware is running ==="
    @app.call(env)
  end
end
```

If it prints but has no effect, your logic has a bug.

If it doesn't print, it's not in the stack or it's being skipped.

---

### Scenario 2: "Request gets a 500 before reaching my controller"

**Symptoms:** Server returns 500, but your controller code never runs. No useful error in logs.

**Debugging steps:**

```ruby
# 1. Add debug middleware early in the stack
config.middleware.insert_before 0, DebugMiddleware, label: "VERY_FIRST"
```

If the debug middleware prints "BEFORE" but not "AFTER", something between it and the app raised an error.

```ruby
# 2. Add a catch-all error middleware
class ErrorCatcher
  def initialize(app)
    @app = app
  end

  def call(env)
    @app.call(env)
  rescue => e
    Rails.logger.error "CAUGHT IN MIDDLEWARE: #{e.class}: #{e.message}"
    Rails.logger.error e.backtrace.first(10).join("\n")
    raise  # re-raise so normal error handling continues
  end
end

config.middleware.insert_before 0, ErrorCatcher
```

This catches errors anywhere in the stack and logs them before they're swallowed.

---

### Scenario 3: "Session data is empty"

**Symptoms:** `session[:user_id]` returns nil even though you set it.

**Debugging steps:**

```ruby
# 1. Check middleware order
bin/rails middleware
```

Is `ActionDispatch::Session::CookieStore` in the stack? Is `ActionDispatch::Cookies` before it?

```ruby
# 2. Check the session cookie in the browser
# Open DevTools → Application → Cookies
# Look for _myapp_session
```

If the cookie is missing, cookies aren't being set. Check if your response has `Set-Cookie` header.

```ruby
# 3. Debug in middleware
config.middleware.insert_after ActionDispatch::Session::CookieStore,
  DebugMiddleware, label: "AFTER_SESSION"
```

Check if `env["rack.session"]` has data after the session middleware runs.

---

### Scenario 4: "Wrong IP address in logs"

**Symptoms:** All requests show `127.0.0.1` or the load balancer's IP.

**Debugging steps:**

```ruby
# 1. Check what RemoteIp sees
class IpDebugger
  def initialize(app)
    @app = app
  end

  def call(env)
    puts "REMOTE_ADDR: #{env['REMOTE_ADDR']}"
    puts "HTTP_X_FORWARDED_FOR: #{env['HTTP_X_FORWARDED_FOR']}"
    puts "HTTP_X_REAL_IP: #{env['HTTP_X_REAL_IP']}"

    status, headers, body = @app.call(env)

    puts "remote_ip: #{env['action_dispatch.remote_ip']}"

    [status, headers, body]
  end
end

config.middleware.insert_after ActionDispatch::RemoteIp, IpDebugger
```

Usually the issue is that Nginx isn't setting `X-Forwarded-For`, or trusted proxies aren't configured.

Fix in Nginx:

```nginx
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Real-IP $remote_addr;
```

Fix in Rails:

```ruby
config.action_dispatch.trusted_proxies = [
  IPAddr.new("10.0.0.0/8"),
  IPAddr.new("172.16.0.0/12"),
  IPAddr.new("192.168.0.0/16"),
]
```

---

### Scenario 5: "Middleware works in development but not production"

**Symptoms:** Your middleware works locally but behaves differently in production.

**Common causes:**

1. **Different middleware stack.** Development has `ShowExceptions`, `Reloader`, `CheckPending`. Production doesn't.

2. **Environment-specific config.** Middleware registered in `config/environments/development.rb` doesn't exist in production.

3. **Reverse proxy interferes.** Nginx might strip headers, modify the request, or handle things your middleware expects.

4. **Different concurrency.** Development runs single-threaded. Production runs multi-threaded. Race conditions only appear in production.

**Fix:** Compare stacks:

```bash
# Development
RAILS_ENV=development bin/rails middleware > /tmp/dev_stack.txt

# Production
RAILS_ENV=production bin/rails middleware > /tmp/prod_stack.txt

diff /tmp/dev_stack.txt /tmp/prod_stack.txt
```

---

## Interview Questions — Debugging

**Q: How do you debug middleware issues in Rails?**
A: Start with `bin/rails middleware` to see the stack. Add a debug middleware with logging at specific points. Use curl to send precise requests. Check env keys and response headers.

**Q: A request gets a 500 but the controller never runs. How do you find the problem?**
A: Add error-catching middleware at the top of the stack that logs exceptions with stack traces. The error is in a middleware before the router.

**Q: How do you test the full middleware stack in Rails console?**
A: Build a mock env with `Rack::MockRequest.env_for`, then call `Rails.application.call(env)`. This runs the full stack without a server.

**Q: Session data is empty even though you set it. What do you check?**
A: Verify session middleware is in the stack and ordered after Cookies middleware. Check the browser for the session cookie. Debug with middleware to inspect `env["rack.session"]`.

---

# 4. Logging

## Why Logging at the Rack Level?

Rack middleware is the ideal place for request logging because:

1. It sees **every** request — even ones that never reach a controller (404s, middleware rejections)
2. It can measure **total** request time (including middleware overhead)
3. It can capture **both** request and response data in one place
4. It runs in a known order, so you can control what gets logged when

---

## Rails' Default Logging

Rails' `Rails::Rack::Logger` middleware produces the logs you see in development:

```
Started GET "/users" for 127.0.0.1 at 2024-01-15 10:30:45
Processing by UsersController#index as HTML
  User Load (1.2ms)  SELECT "users".* FROM "users"
  Rendering users/index.html.erb
  Rendered users/index.html.erb (Duration: 3.5ms)
Completed 200 OK in 15ms (Views: 5.2ms | ActiveRecord: 1.2ms)
```

This is great for development. But production needs different logging.

---

## Production Logging Requirements

Production logs need to be:

| Requirement | Why |
|-------------|-----|
| **Structured (JSON)** | Monitoring tools (Datadog, Splunk, ELK) can parse JSON. Human-readable text is hard to parse. |
| **One line per request** | Easy to grep. No multi-line log entries that get split across log files. |
| **Include request ID** | Trace requests across services |
| **Include timing** | Find slow requests |
| **Include metadata** | IP, user agent, user ID, response status |
| **Not too verbose** | High-traffic apps generate millions of log lines. Keep them concise. |

---

## Building a Production Logger Middleware

```ruby
# app/middleware/json_request_logger.rb

class JsonRequestLogger
  def initialize(app)
    @app = app
  end

  def call(env)
    start = Process.clock_gettime(Process::CLOCK_MONOTONIC)

    # Let the request proceed
    status, headers, body = @app.call(env)

    duration = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start

    # Build the log entry
    log_entry = {
      method: env["REQUEST_METHOD"],
      path: env["PATH_INFO"],
      query: env["QUERY_STRING"].presence,
      status: status,
      duration_ms: (duration * 1000).round(2),
      ip: env["action_dispatch.remote_ip"]&.to_s || env["REMOTE_ADDR"],
      request_id: env["action_dispatch.request_id"],
      user_agent: env["HTTP_USER_AGENT"],
      content_length: headers["Content-Length"],
      timestamp: Time.now.utc.iso8601(3)
    }.compact  # remove nil values

    # Write as single JSON line
    Rails.logger.info(log_entry.to_json)

    [status, headers, body]
  rescue => e
    # Log errors too
    duration = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start

    error_entry = {
      method: env["REQUEST_METHOD"],
      path: env["PATH_INFO"],
      status: 500,
      duration_ms: (duration * 1000).round(2),
      error: e.class.name,
      error_message: e.message,
      timestamp: Time.now.utc.iso8601(3)
    }

    Rails.logger.error(error_entry.to_json)
    raise  # re-raise the error
  end
end
```

Output (one line per request):

```json
{"method":"GET","path":"/api/users","status":200,"duration_ms":45.23,"ip":"203.0.113.50","request_id":"7c4b4c5a","user_agent":"Mozilla/5.0...","content_length":"1234","timestamp":"2024-01-15T10:30:45.123Z"}
```

---

## Tagged Logging

Rails supports tagged logging — adding tags to every log line:

```ruby
# config/application.rb
config.log_tags = [:request_id]
```

Now every log line includes the request ID:

```
[7c4b4c5a-9e3f-4b5a] Started GET "/users" ...
[7c4b4c5a-9e3f-4b5a] User Load (1.2ms) ...
[7c4b4c5a-9e3f-4b5a] Completed 200 OK ...
```

You can add custom tags:

```ruby
config.log_tags = [
  :request_id,
  -> (request) { "IP:#{request.remote_ip}" },
  -> (request) { "User:#{request.cookies['user_id']}" }
]
```

Output:

```
[7c4b4c5a] [IP:203.0.113.50] [User:42] Started GET "/users" ...
```

---

## Silencing Noisy Requests

Health checks and asset requests generate a lot of log noise:

```
Started GET "/health" for 10.0.0.1 at ...
Completed 200 OK in 1ms
Started GET "/health" for 10.0.0.1 at ...
Completed 200 OK in 1ms
Started GET "/health" for 10.0.0.1 at ...
Completed 200 OK in 1ms
(every 10 seconds...)
```

Silence them:

```ruby
# app/middleware/silence_request.rb

class SilenceRequest
  SILENT_PATHS = ["/health", "/ping"].freeze

  def initialize(app)
    @app = app
  end

  def call(env)
    if SILENT_PATHS.include?(env["PATH_INFO"])
      Rails.logger.silence do
        @app.call(env)
      end
    else
      @app.call(env)
    end
  end
end
```

```ruby
# config/application.rb
config.middleware.insert_before Rails::Rack::Logger, SilenceRequest
```

Now health checks don't clutter your logs.

---

## Log Levels in Middleware

Different environments need different log levels:

```ruby
# config/environments/development.rb
config.log_level = :debug    # show everything

# config/environments/production.rb
config.log_level = :info     # skip debug noise
```

In your middleware:

```ruby
def call(env)
  Rails.logger.debug "Detailed info..."   # only in development
  Rails.logger.info  "Important info..."   # in production too
  Rails.logger.warn  "Something wrong..."  # something unexpected
  Rails.logger.error "Error occurred..."   # errors
  @app.call(env)
end
```

---

## Logging Request Bodies (Carefully)

Sometimes you need to log the request body for debugging API issues:

```ruby
class RequestBodyLogger
  def initialize(app)
    @app = app
  end

  def call(env)
    request = Rack::Request.new(env)

    if request.post? || request.put? || request.patch?
      body = request.body.read
      request.body.rewind   # IMPORTANT: rewind so the app can read it too

      # Don't log sensitive data!
      safe_body = filter_sensitive(body)
      Rails.logger.info "Request body: #{safe_body}"
    end

    @app.call(env)
  end

  private

  def filter_sensitive(body)
    # Replace passwords, tokens, credit cards
    body.gsub(/"password"\s*:\s*"[^"]*"/, '"password":"[FILTERED]"')
        .gsub(/"token"\s*:\s*"[^"]*"/, '"token":"[FILTERED]"')
        .gsub(/\b\d{13,16}\b/, "[CARD_FILTERED]")
  end
end
```

**Important rules for body logging:**

1. Always `rewind` the body after reading (Part 2 explained why)
2. Never log passwords, tokens, or credit card numbers
3. Only enable in development or with careful filtering
4. Be aware of GDPR and privacy regulations

---

## Interview Questions — Logging

**Q: Why is middleware a good place for logging?**
A: It sees every request including 404s and middleware rejections. It can measure total request time. It runs in a predictable order.

**Q: How do you implement structured logging in Rails?**
A: Create a middleware that logs a JSON object with method, path, status, duration, IP, request ID, and timestamp. One JSON line per request.

**Q: How do you silence health check logs?**
A: Create middleware that checks the path and wraps `@app.call(env)` in `Rails.logger.silence` for health check paths. Place it before the Logger middleware.

**Q: What are log tags and why are they useful?**
A: Tags are prefixes added to every log line. The request ID tag lets you trace all log lines for one request. Configure with `config.log_tags`.

**Q: What's the danger of logging request bodies?**
A: You might log passwords, tokens, or personal data. Always filter sensitive fields. Always rewind the body after reading so downstream code can read it too.

---

# 5. Common Mistakes

These are mistakes that even experienced Ruby developers make when working with Rack and middleware.

---

## Mistake 1: Forgetting to Return the Response

```ruby
# BROKEN
class BadMiddleware
  def initialize(app)
    @app = app
  end

  def call(env)
    @app.call(env)
    # forgot to return the response!
    # Ruby returns nil (the return value of the last expression)
  end
end
```

Wait — doesn't `@app.call(env)` return the response? Yes, but if you have code after it, Ruby returns the last expression:

```ruby
def call(env)
  status, headers, body = @app.call(env)
  headers["X-Custom"] = "value"
  # Ruby returns "value" (result of the assignment), not [status, headers, body]!
end
```

**Fix:** Always explicitly return the response array:

```ruby
def call(env)
  status, headers, body = @app.call(env)
  headers["X-Custom"] = "value"
  [status, headers, body]   # explicit return
end
```

---

## Mistake 2: Reading the Body Without Rewinding

From Part 2, the request body is an IO-like object. Reading it moves the pointer:

```ruby
# BROKEN
def call(env)
  body = env["rack.input"].read   # reads the body
  Rails.logger.info "Body: #{body}"
  # rack.input pointer is now at the end
  # downstream code gets EMPTY body!
  @app.call(env)
end
```

**Fix:** Always rewind:

```ruby
def call(env)
  body = env["rack.input"].read
  env["rack.input"].rewind   # reset pointer
  Rails.logger.info "Body: #{body}"
  @app.call(env)
end
```

---

## Mistake 3: Storing Per-Request State in Instance Variables

From Part 4 — middleware instances are shared across threads:

```ruby
# BROKEN — race condition in threaded Puma
class BadMiddleware
  def initialize(app)
    @app = app
  end

  def call(env)
    @current_path = env["PATH_INFO"]   # Thread A writes "/users"
                                        # Thread B writes "/orders"
                                        # Thread A now sees "/orders"!
    do_something_with(@current_path)
    @app.call(env)
  end
end
```

**Fix:** Use local variables:

```ruby
def call(env)
  current_path = env["PATH_INFO"]   # local — thread-safe
  do_something_with(current_path)
  @app.call(env)
end
```

---

## Mistake 4: Modifying the Body Incorrectly

```ruby
# BROKEN — modifies a streaming body
def call(env)
  status, headers, body = @app.call(env)

  # This breaks streaming!
  modified_body = ""
  body.each { |chunk| modified_body << chunk }
  modified_body.upcase!

  [status, headers, [modified_body]]
end
```

This loads the entire body into memory. If the body is a streaming Enumerator, you've defeated the purpose.

**Fix:** Only modify the body if you need to, and be aware of the memory implications:

```ruby
def call(env)
  status, headers, body = @app.call(env)

  # Only modify small responses
  if headers["Content-Length"].to_i < 1_000_000  # less than 1MB
    chunks = []
    body.each { |chunk| chunks << chunk }
    body.close if body.respond_to?(:close)  # clean up original body

    modified = chunks.join.upcase
    headers["Content-Length"] = modified.bytesize.to_s

    [status, headers, [modified]]
  else
    [status, headers, body]  # pass through large responses unchanged
  end
end
```

---

## Mistake 5: Not Closing the Body

The Rack spec says:

> The body must respond to `close` if it has state to clean up.

```ruby
# BROKEN — body is never closed, resources leak
def call(env)
  status, headers, body = @app.call(env)

  new_body = transform(body)
  # original body is abandoned without closing!

  [status, headers, [new_body]]
end
```

**Fix:**

```ruby
def call(env)
  status, headers, body = @app.call(env)

  new_body = transform(body)
  body.close if body.respond_to?(:close)   # clean up

  [status, headers, [new_body]]
end
```

This matters when the body is a file handle, database cursor, or any resource that needs cleanup.

---

## Mistake 6: Wrong Middleware Order

```ruby
# BROKEN — rate limiter runs after sessions and cookies
config.middleware.use RateLimiter
```

By default, `use` appends to the end. Your rate limiter runs after all the heavy middleware, defeating its purpose.

**Fix:** Place it early:

```ruby
config.middleware.insert_after ActionDispatch::RemoteIp, RateLimiter
```

---

## Mistake 7: Raising Exceptions in After-Processing

```ruby
# BROKEN
def call(env)
  status, headers, body = @app.call(env)

  # This can raise!
  process_response_data(status, headers, body)

  [status, headers, body]
rescue SomeError
  # You caught the error, but the original response is lost!
  [500, {}, ["Error in middleware"]]
end
```

If `process_response_data` raises, you replace a perfectly good 200 response with a 500.

**Fix:** Don't let after-processing errors affect the response:

```ruby
def call(env)
  status, headers, body = @app.call(env)

  begin
    process_response_data(status, headers, body)
  rescue => e
    Rails.logger.error "Middleware post-processing failed: #{e.message}"
    # Don't change the response!
  end

  [status, headers, body]
end
```

---

## Mistake 8: Assuming env Keys Exist

```ruby
# BROKEN — crashes if Authorization header is missing
def call(env)
  token = env["HTTP_AUTHORIZATION"].split(" ").last
  # NoMethodError: undefined method 'split' for nil:NilClass
  @app.call(env)
end
```

**Fix:** Always handle missing keys:

```ruby
def call(env)
  auth_header = env["HTTP_AUTHORIZATION"]
  return @app.call(env) unless auth_header

  token = auth_header.split(" ", 2).last
  @app.call(env)
end
```

Or use the safe navigation operator:

```ruby
token = env["HTTP_AUTHORIZATION"]&.split(" ", 2)&.last
```

---

## Mistake 9: Middleware That Works in Tests but Fails in Production

Tests often use `Rack::MockRequest`, which creates a minimal env. Production servers add many more keys.

```ruby
# Works in tests (mock env), breaks in production
def call(env)
  if env["rack.url_scheme"] == "https"
    # Works fine
  end

  # But this might not work — env["HTTP_X_FORWARDED_PROTO"] exists
  # in production but not in tests
end
```

**Fix:** Test with realistic env hashes or use integration tests that go through the full stack.

---

## Mistake 10: Not Handling HEAD Requests

`Rack::Head` strips the body for HEAD requests. But if your middleware checks `Content-Length` or reads the body, it might get confused:

```ruby
# BROKEN — HEAD request has empty body, Content-Length mismatch
def call(env)
  status, headers, body = @app.call(env)

  full_body = ""
  body.each { |chunk| full_body << chunk }

  if full_body.bytesize != headers["Content-Length"].to_i
    Rails.logger.warn "Content-Length mismatch!"   # false alarm for HEAD!
  end

  [status, headers, [full_body]]
end
```

**Fix:** Check the request method:

```ruby
def call(env)
  status, headers, body = @app.call(env)

  unless env["REQUEST_METHOD"] == "HEAD"
    # only process body for non-HEAD requests
    validate_body(body)
  end

  [status, headers, body]
end
```

---

## Interview Questions — Common Mistakes

**Q: What happens if you forget to return the response array from middleware?**
A: The server gets `nil` instead of `[status, headers, body]`. You get a cryptic `NoMethodError` or `TypeError`. Always explicitly return the three-element array.

**Q: Why must you rewind the request body after reading?**
A: The body is an IO object with a read pointer. After reading, the pointer is at the end. Without rewinding, downstream code gets an empty body. Call `body.rewind` after `body.read`.

**Q: What's the most common thread-safety mistake in middleware?**
A: Storing per-request state in instance variables. Instance variables are shared across threads. Use local variables or `env` instead.

---

# 6. Best Practices

## The Rules of Good Middleware

### Rule 1: Single Responsibility

Each middleware should do **one thing**:

```ruby
# GOOD — does one thing
class RequestTimer
  def call(env)
    start = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    status, headers, body = @app.call(env)
    headers["X-Runtime"] = (Process.clock_gettime(Process::CLOCK_MONOTONIC) - start).to_s
    [status, headers, body]
  end
end
```

```ruby
# BAD — does authentication, logging, and rate limiting
class KitchenSinkMiddleware
  def call(env)
    log_request(env)
    check_auth(env)
    check_rate_limit(env)
    # ... too many responsibilities
  end
end
```

Split it into three separate middleware. Each can be reordered, removed, or replaced independently.

---

### Rule 2: Always Call @app.call(env) Unless Short-Circuiting

```ruby
# GOOD — explicitly short-circuits with a reason
def call(env)
  unless authorized?(env)
    return [401, { "Content-Type" => "application/json" }, ['{"error":"Unauthorized"}']]
  end

  @app.call(env)
end
```

```ruby
# BAD — sometimes forgets to call @app
def call(env)
  if env["PATH_INFO"] == "/special"
    do_something
    # forgot @app.call(env)! returns nil
  else
    @app.call(env)
  end
end
```

Every code path must either call `@app.call(env)` or return a valid response.

---

### Rule 3: Namespace Your env Keys

```ruby
# BAD — might clash with other middleware
env["user"] = current_user
env["start_time"] = Time.now

# GOOD — namespaced
env["my_app.user"] = current_user
env["my_app.start_time"] = Time.now
```

Rack uses `rack.` prefix. Rails uses `action_dispatch.`. Your middleware should use your own prefix.

---

### Rule 4: Handle Errors Gracefully

```ruby
# GOOD
def call(env)
  begin
    do_pre_processing(env)
  rescue => e
    Rails.logger.error "Middleware pre-processing failed: #{e.message}"
    # Continue with the request even if pre-processing fails
  end

  @app.call(env)
end
```

```ruby
# BAD — unhandled error crashes the entire request
def call(env)
  do_pre_processing(env)   # if this raises, request dies
  @app.call(env)
end
```

Unless your middleware is meant to halt the request on error (like auth), don't let errors break the request.

---

### Rule 5: Close the Body When You Replace It

```ruby
# GOOD
def call(env)
  status, headers, body = @app.call(env)

  new_body = transform_body(body)
  body.close if body.respond_to?(:close)  # release resources

  [status, headers, new_body]
end
```

---

### Rule 6: Be Idempotent

Middleware should produce the same result regardless of how many times it runs on the same request:

```ruby
# GOOD — setting a header is idempotent
def call(env)
  status, headers, body = @app.call(env)
  headers["X-Custom"] = "value"
  [status, headers, body]
end
```

```ruby
# BAD — each call adds another value
def call(env)
  status, headers, body = @app.call(env)
  existing = headers["X-Custom"] || ""
  headers["X-Custom"] = existing + ",value"  # grows with each call!
  [status, headers, body]
end
```

---

### Rule 7: Make Middleware Configurable

```ruby
# GOOD — configurable
class RequestTimer
  def initialize(app, header_name: "X-Runtime", precision: 2)
    @app = app
    @header_name = header_name
    @precision = precision
  end

  def call(env)
    start = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    status, headers, body = @app.call(env)
    duration = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start
    headers[@header_name] = (duration * 1000).round(@precision).to_s
    [status, headers, body]
  end
end

# Usage
config.middleware.use RequestTimer, header_name: "X-Duration", precision: 4
```

This makes middleware reusable across different apps with different needs.

---

### Rule 8: Use Process.clock_gettime for Timing

```ruby
# BAD — Time.now is affected by system clock changes
start = Time.now
# ... if NTP adjusts the clock, your duration is wrong
duration = Time.now - start

# GOOD — monotonic clock always moves forward
start = Process.clock_gettime(Process::CLOCK_MONOTONIC)
# ... clock adjustments don't affect this
duration = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start
```

`Time.now` can go backward (NTP adjustments, daylight saving). The monotonic clock never goes backward. Always use it for measuring durations.

---

### Rule 9: Test Middleware in Isolation

```ruby
# Test setup
inner_app = ->(env) { [200, { "Content-Type" => "text/plain" }, ["OK"]] }
middleware = MyMiddleware.new(inner_app)

# Test
status, headers, body = middleware.call(Rack::MockRequest.env_for("/test"))
assert_equal 200, status
assert headers.key?("X-Custom")
```

Don't test middleware through the full Rails stack unless you need to test integration. Unit tests are faster and more focused.

---

### Rule 10: Document Your Middleware

```ruby
# Adds a unique request ID to every request.
#
# Sets env["my_app.request_id"] for use by downstream middleware and the app.
# Adds X-Request-Id header to the response.
#
# If the client sends X-Request-Id, that value is used (for tracing across services).
# Otherwise, a new UUID is generated.
#
# Usage:
#   config.middleware.insert_after ActionDispatch::RemoteIp, RequestId
#
# Options:
#   header_name: Custom header name (default: "X-Request-Id")
#
class RequestId
  def initialize(app, header_name: "X-Request-Id")
    @app = app
    @header_name = header_name
  end

  # ...
end
```

When another developer sees this middleware in the stack, they should understand what it does without reading the code.

---

## Interview Questions — Best Practices

**Q: What's the most important rule when writing middleware?**
A: Single responsibility. Each middleware should do one thing well. This makes the stack composable — you can add, remove, or reorder middleware without breaking things.

**Q: How do you avoid env key collisions?**
A: Namespace your keys with a prefix like `my_app.something`. Rack uses `rack.`, Rails uses `action_dispatch.`.

**Q: Why should you use the monotonic clock for timing?**
A: `Time.now` can be affected by system clock changes (NTP, daylight saving). `Process.clock_gettime(Process::CLOCK_MONOTONIC)` always moves forward, giving accurate duration measurements.

**Q: How do you test middleware?**
A: Create a simple lambda as the inner app, wrap it with your middleware, call it with a mock env (`Rack::MockRequest.env_for`), and assert on the result.

---

# 7. Anti-Patterns

Anti-patterns are things that **look** reasonable but cause problems. Learn to recognize them.

---

## Anti-Pattern 1: The God Middleware

```ruby
# ANTI-PATTERN
class ApplicationMiddleware
  def call(env)
    # Authentication
    check_auth(env)

    # Rate limiting
    check_rate_limit(env)

    # Logging
    log_request(env)

    # CORS
    add_cors_headers(env)

    # Request ID
    set_request_id(env)

    status, headers, body = @app.call(env)

    # Response logging
    log_response(status)

    # Compression
    compress(body)

    # Timing header
    add_timing(headers)

    [status, headers, body]
  end
end
```

**Problem:** Can't reorder, remove, or test individual features. One bug in CORS affects auth. One change in logging breaks rate limiting.

**Fix:** Split into separate middleware, one per concern.

---

## Anti-Pattern 2: Database Calls in Middleware

```ruby
# ANTI-PATTERN
class TenantMiddleware
  def call(env)
    # Hits the database on EVERY request, including health checks!
    tenant = Tenant.find_by!(subdomain: extract_subdomain(env))
    env["current_tenant"] = tenant
    @app.call(env)
  end
end
```

**Problem:** Every request hits the database — even health checks, asset requests, and OPTIONS requests. If the database is down, health checks fail too.

**Fix:** Cache the result and skip for non-application requests:

```ruby
class TenantMiddleware
  SKIP_PATHS = ["/health", "/assets"].freeze

  def initialize(app)
    @app = app
  end

  def call(env)
    return @app.call(env) if skip_path?(env)

    subdomain = extract_subdomain(env)

    tenant = Rails.cache.fetch("tenant:#{subdomain}", expires_in: 5.minutes) do
      Tenant.find_by(subdomain: subdomain)
    end

    unless tenant
      return [404, { "Content-Type" => "application/json" }, ['{"error":"Tenant not found"}']]
    end

    env["current_tenant"] = tenant
    @app.call(env)
  end

  private

  def skip_path?(env)
    SKIP_PATHS.any? { |path| env["PATH_INFO"].start_with?(path) }
  end
end
```

---

## Anti-Pattern 3: Middleware That Swallows Errors

```ruby
# ANTI-PATTERN
class SilentMiddleware
  def call(env)
    @app.call(env)
  rescue => e
    # Silently swallows ALL errors
    [200, {}, ["OK"]]   # pretends everything is fine!
  end
end
```

**Problem:** Real errors are hidden. The user gets a 200 but the data is wrong. Bugs go undetected for weeks.

**Fix:** If you catch errors, log them and return an appropriate error status:

```ruby
class ErrorHandler
  def call(env)
    @app.call(env)
  rescue => e
    Rails.logger.error "#{e.class}: #{e.message}"
    Rails.logger.error e.backtrace.first(10).join("\n")

    # Report to error tracker
    Sentry.capture_exception(e) if defined?(Sentry)

    [500, { "Content-Type" => "application/json" },
     ['{"error":"Internal Server Error"}']]
  end
end
```

---

## Anti-Pattern 4: Middleware That Modifies env Destructively

```ruby
# ANTI-PATTERN
class PathRewriter
  def call(env)
    # Overwrites the original path with no way to recover it
    env["PATH_INFO"] = env["PATH_INFO"].gsub("/api/v1", "/api/v2")
    @app.call(env)
  end
end
```

**Problem:** The original path is lost. If downstream middleware or the app needs the original path for logging or routing, it gets the modified version.

**Fix:** Store the original value:

```ruby
class PathRewriter
  def call(env)
    env["my_app.original_path"] = env["PATH_INFO"]
    env["PATH_INFO"] = env["PATH_INFO"].gsub("/api/v1", "/api/v2")
    @app.call(env)
  end
end
```

---

## Anti-Pattern 5: Time-Dependent Middleware Without Timeouts

```ruby
# ANTI-PATTERN
class ExternalServiceMiddleware
  def call(env)
    # If the external service is slow, EVERY request is slow
    result = HTTP.get("https://slow-service.com/check")
    env["service_check"] = result
    @app.call(env)
  end
end
```

**Problem:** If the external service takes 30 seconds, every request takes 30+ seconds. Your app becomes unusable.

**Fix:** Add timeouts and fallbacks:

```ruby
class ExternalServiceMiddleware
  def call(env)
    begin
      result = HTTP.timeout(connect: 1, read: 2)
                   .get("https://slow-service.com/check")
      env["service_check"] = result
    rescue HTTP::TimeoutError
      Rails.logger.warn "External service timeout, using fallback"
      env["service_check"] = default_result
    end

    @app.call(env)
  end
end
```

---

## Anti-Pattern 6: Middleware That Changes Behavior Based on Global State

```ruby
# ANTI-PATTERN
class FeatureMiddleware
  @@enabled = true  # class variable — shared across ALL instances and threads

  def self.disable!
    @@enabled = false
  end

  def call(env)
    if @@enabled
      do_feature_stuff(env)
    end
    @app.call(env)
  end
end
```

**Problem:** Class variables are shared across threads. Calling `disable!` affects all requests instantly, including ones in progress. No way to enable for some requests and disable for others.

**Fix:** Use per-request configuration:

```ruby
class FeatureMiddleware
  def initialize(app, enabled_check: -> (env) { true })
    @app = app
    @enabled_check = enabled_check
  end

  def call(env)
    if @enabled_check.call(env)
      do_feature_stuff(env)
    end
    @app.call(env)
  end
end

# Usage
config.middleware.use FeatureMiddleware,
  enabled_check: -> (env) { ENV["FEATURE_X_ENABLED"] == "true" }
```

Or use a feature flag system that checks per-request.

---

## Anti-Pattern 7: Duplicate Middleware

```ruby
# ANTI-PATTERN — same middleware added twice
config.middleware.use RequestTimer
# ... later in another initializer ...
config.middleware.use RequestTimer   # oops, added again
```

**Problem:** The middleware runs twice. Timing is wrong, headers are set twice, or side effects happen twice.

**Fix:** Check before adding:

```ruby
unless config.middleware.include?(RequestTimer)
  config.middleware.use RequestTimer
end
```

Or use `insert_before`/`insert_after` which makes the position explicit and less likely to be duplicated.

---

## Anti-Pattern Summary Table

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| God middleware | Can't test, reorder, or debug | Split into single-responsibility middleware |
| DB calls in middleware | Hits DB on every request | Cache results, skip non-app paths |
| Swallowing errors | Bugs go undetected | Log errors, return proper status codes |
| Destructive env changes | Original data lost | Preserve originals in separate keys |
| No timeouts | One slow service blocks everything | Add timeouts and fallbacks |
| Global state | Race conditions, no per-request control | Use per-request checks or feature flags |
| Duplicate middleware | Double processing | Check before adding, use explicit positioning |

---

## Interview Questions — Anti-Patterns

**Q: What's wrong with a middleware that does auth, logging, AND rate limiting?**
A: Too many responsibilities. Can't test, reorder, or remove individual features. If one breaks, all break. Split into separate middleware.

**Q: Why shouldn't middleware make database calls without caching?**
A: Because it runs on every request — including health checks, assets, and OPTIONS requests. Without caching, it adds unnecessary database load and fails if the database is down.

**Q: What's the danger of catching errors in middleware without logging?**
A: Real bugs get hidden. Users get 200 OK but wrong data. Issues go undetected until they become critical. Always log caught errors and report to error tracking.

**Q: How do you prevent a slow external service from blocking all requests?**
A: Add timeouts (connect and read) to external HTTP calls in middleware. Provide a fallback value when the service is unavailable.

---

# Mental Model — Part 5

```
Performance
├── Short-circuit early (reject fast)
├── Skip middleware for certain paths
├── Cache expensive operations
├── Remove unused middleware
├── Compress with Rack::Deflater
└── Size connection pools correctly

Security
├── HostAuthorization (host injection)
├── RemoteIp (IP spoofing)
├── Rack::Attack (DoS/rate limiting)
├── Request size limits (memory attacks)
├── Security headers (XSS, clickjacking)
├── Encrypted/signed cookies (tampering)
└── Clean error pages in production (info leakage)

Debugging
├── bin/rails middleware (see the stack)
├── Debug middleware (log before/after)
├── Rack::Lint (spec validation)
├── curl -v (raw HTTP inspection)
├── Rack::MockRequest (console testing)
└── Compare dev vs prod stacks

Logging
├── Structured JSON (one line per request)
├── Tagged logging (request ID prefix)
├── Silence noisy paths (health checks)
├── Filter sensitive data (passwords, tokens)
└── Log levels (debug in dev, info in prod)

Common Mistakes
├── Forgetting to return response
├── Not rewinding body after read
├── Instance variables in call (thread unsafe)
├── Not closing replaced bodies
├── Wrong middleware order
└── Not handling missing env keys

Best Practices
├── Single responsibility
├── Always call @app or return response
├── Namespace env keys
├── Handle errors gracefully
├── Use monotonic clock for timing
├── Make middleware configurable
├── Test in isolation
└── Document your middleware

Anti-Patterns
├── God middleware (too many responsibilities)
├── DB calls without caching
├── Swallowing errors silently
├── Destructive env changes
├── No timeouts on external calls
├── Global mutable state
└── Duplicate middleware
```

---

# Key Takeaways — Part 5

1. **Performance** — Short-circuit early, skip paths, cache, remove unused middleware, compress responses. Every millisecond in middleware multiplies across all requests.
2. **Security** — Rack middleware is the first defense. Host authorization, rate limiting, request size limits, security headers, and encrypted cookies all live here.
3. **Debugging** — Use `bin/rails middleware`, debug middleware, `Rack::Lint`, curl, and mock requests. Compare dev and prod stacks when behavior differs.
4. **Logging** — Structured JSON, one line per request, tagged with request ID, filtered for sensitive data. Silence health checks.
5. **Common Mistakes** — Return the response array, rewind the body, don't use instance variables in `call`, close replaced bodies, check for nil env keys.
6. **Best Practices** — Single responsibility, namespace env keys, handle errors, use monotonic clock, configure via options, test in isolation, document.
7. **Anti-Patterns** — No god middleware, no uncached DB calls, no swallowed errors, no destructive env changes, no missing timeouts, no global mutable state.

---

# 🎉 Part 5 Complete

You now understand:

* ✅ Performance — optimizing middleware for speed and memory
* ✅ Security — protecting your app at the Rack level
* ✅ Debugging — finding and fixing middleware bugs
* ✅ Logging — structured, production-ready request logging
* ✅ Common Mistakes — the 10 most common middleware bugs
* ✅ Best Practices — the 10 rules of good middleware
* ✅ Anti-Patterns — the 7 things to never do in middleware

**Next up in Part 6:** Advanced Examples, Edge Cases, Comparison Tables, Interview Questions, Coding Exercises, Cheat Sheet, Summary, and Resources — the final part!

---

## Progress Tracker — Rack Study Guide

| Part | Topics | Status |
|------|--------|--------|
| **Part 1** | Overview, History, Why Rack Exists, Problems Rack Solves, Rack Architecture, Rack Specification, Rack Application, Rack Environment, Hello World, Request Flow | 🟢 Complete |
| **Part 2** | Rack::Request, Rack::Response, Rack::Builder, config.ru (Rackup), Middleware Fundamentals, Middleware Chain, Complete Request Lifecycle | 🟢 Complete |
| **Part 3** | Rails + Rack, ActionDispatch, Rails Middleware Stack, Middleware Ordering, Custom Middleware, Production Examples | 🟢 Complete |
| **Part 4** | Internal Implementation, Source Code Walkthrough, Thread Safety, Concurrency, Streaming, Hijacking, Async, HTTP/2 | 🟢 Complete |
| **Part 5** | Performance, Security, Debugging, Logging, Common Mistakes, Best Practices, Anti-patterns | 🟢 Complete |
| **Part 6** | Advanced Examples, Edge Cases, Comparison Tables, Interview Questions, Coding Exercises, Cheat Sheet, Summary, Resources | ⬜ Not Started |
