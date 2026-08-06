# Part 6 — Advanced Examples, Edge Cases, and Interview Prep

Welcome to the final part.

You’ve made it. You now understand Rack from its simple interface (`call(env)`) all the way to its deepest internals, concurrency models, and production optimizations.

Part 6 is designed to be your **ultimate interview prep and reference guide**. It pulls everything together. We'll look at some advanced real-world examples, cover tricky edge cases, provide quick-reference comparison tables, and test your knowledge with senior-level interview questions and coding exercises.

Let's finish strong.

---

# What You'll Learn in Part 6

1. **Advanced Examples** — building complex middleware (A/B testing, API Gateway)
2. **Edge Cases** — weird Rack behaviors and how to handle them
3. **Comparison Tables** — quick references for servers, frameworks, and middleware
4. **Interview Questions** — the ultimate list of senior backend questions on Rack
5. **Coding Exercises** — practical tasks to test your skills
6. **Cheat Sheet** — a condensed summary of everything you've learned
7. **Summary** — the final wrap-up
8. **Resources** — where to go next

---

# 1. Advanced Examples

In previous parts, we looked at standard middleware (auth, logging, rate limiting). Let's look at two advanced patterns you might build in a large enterprise system like Shopify or GitLab.

## Advanced Example 1: A/B Testing Routing Middleware

**Scenario:** You want to test a new checkout system. You want 10% of users to go to the new system (`/checkout_v2`), and 90% to go to the old system (`/checkout`). The routing should happen transparently—the URL in the browser should stay `/checkout` for everyone.

**Why Middleware?** Doing this in a controller is messy and requires changing your core business logic. Middleware can intercept the request, rewrite the path for 10% of users based on a cookie, and let Rails route it to the new controller automatically.

```ruby
class AbTestingRouter
  def initialize(app)
    @app = app
  end

  def call(env)
    request = Rack::Request.new(env)

    # Only apply to the checkout path
    if request.path_info == "/checkout"
      # Determine which cohort the user is in
      cohort = determine_cohort(request)

      if cohort == :treatment
        # Rewrite the path internally. The browser doesn't know.
        # Rails router will now see this as "/checkout_v2"
        env["PATH_INFO"] = "/checkout_v2"
        
        # Save original path for logging or downstream needs
        env["ab_test.original_path"] = "/checkout"
        env["ab_test.cohort"] = "treatment"
      else
        env["ab_test.cohort"] = "control"
      end
    end

    status, headers, body = @app.call(env)

    # Make sure the user stays in the same cohort on future requests
    response = Rack::Response.new(body, status, headers)
    if request.cookies["ab_cohort"].nil?
      # Set cookie for 30 days
      response.set_cookie("ab_cohort", {
        value: env["ab_test.cohort"].to_s,
        expires: Time.now + (30 * 24 * 60 * 60),
        path: "/",
        httponly: true
      })
    end

    response.finish
  end

  private

  def determine_cohort(request)
    existing = request.cookies["ab_cohort"]
    return existing.to_sym if existing == "control" || existing == "treatment"

    # Random assignment: 10% get treatment, 90% get control
    rand < 0.10 ? :treatment : :control
  end
end
```

**Key Takeaways:**
- We manipulated `env["PATH_INFO"]` *before* it hit the Rails router.
- We used `Rack::Response` to inject a cookie on the way out.
- The downstream controllers don't need to know about the A/B test.

---

## Advanced Example 2: API Gateway / Request Proxy

**Scenario:** You are breaking up a monolith. You want to route all `/api/billing` requests to a new Go microservice, but keep everything else in the Rails app.

**Why Middleware?** You *could* configure Nginx to do this. But doing it in Rack allows you to share authentication context (e.g., passing the decoded JWT to the Go service).

```ruby
require 'net/http'

class BillingServiceProxy
  BILLING_PATH = "/api/billing"
  BILLING_SERVICE_URL = "http://internal-billing-service:8080"

  def initialize(app)
    @app = app
  end

  def call(env)
    request = Rack::Request.new(env)

    # If it's not a billing request, pass it to Rails normally
    unless request.path_info.start_with?(BILLING_PATH)
      return @app.call(env)
    end

    # It IS a billing request. Proxy it to the Go service.
    proxy_request_to_billing_service(request, env)
  end

  private

  def proxy_request_to_billing_service(request, env)
    # Construct the target URL
    uri = URI("#{BILLING_SERVICE_URL}#{request.path_info}")
    uri.query = request.query_string unless request.query_string.empty?

    # Create the HTTP request to the microservice
    outbound_request = Net::HTTP.const_get(request.request_method.capitalize).new(uri)
    
    # Forward necessary headers (e.g., Authorization)
    outbound_request['Authorization'] = request.env['HTTP_AUTHORIZATION']
    
    # Forward the body if it's a POST/PUT/PATCH
    if request.post? || request.put? || request.patch?
      outbound_request.body = request.body.read
      request.body.rewind
      outbound_request['Content-Type'] = request.content_type
    end

    # Send the request
    response = Net::HTTP.start(uri.hostname, uri.port) do |http|
      http.request(outbound_request)
    end

    # Build the Rack response from the microservice response
    rack_status = response.code.to_i
    
    rack_headers = {}
    response.each_header do |key, value|
      # Don't forward chunked encoding headers, Rack handles it
      next if key.downcase == 'transfer-encoding'
      rack_headers[key] = value
    end

    rack_body = [response.body]

    [rack_status, rack_headers, rack_body]
  rescue => e
    Rails.logger.error "Billing Proxy Error: #{e.message}"
    [502, { "Content-Type" => "application/json" }, ['{"error":"Bad Gateway"}']]
  end
end
```

**Key Takeaways:**
- This is a complete proxy implementation inside Rack.
- It bypasses Rails entirely for billing routes.
- It demonstrates short-circuiting to communicate with an external service.

---

# 2. Edge Cases

Rack is simple, but HTTP is complicated. Here are some edge cases you should be aware of.

## Edge Case 1: Empty Header Values

What happens if a client sends a header with no value?

```http
X-Custom-Header: 
```

In `env`, this becomes:
```ruby
env["HTTP_X_CUSTOM_HEADER"] = "" # Empty string, NOT nil
```

If you do `env["HTTP_X_CUSTOM_HEADER"] ? "yes" : "no"`, it will evaluate to `"yes"` because empty strings are truthy in Ruby. Always use `.present?` (in Rails) or `.empty?` when checking header values.

## Edge Case 2: Rack Input Freezing (Rack 3)

In Rack 2, you could sometimes get away with modifying `env["rack.input"]` directly.
In Rack 3, the `env` hash and many of its values are strictly managed. The `rack.input` stream might be frozen or wrapped in ways that prevent tampering.

Always use `request.body.read` and `request.body.rewind`, and never try to assign a new IO object to `env["rack.input"]` unless you absolutely have to (and know exactly what you are doing).

## Edge Case 3: Multiple Headers with the Same Name

HTTP allows multiple headers with the same name, particularly `Set-Cookie`.

```http
Set-Cookie: session_id=123; Path=/
Set-Cookie: theme=dark; Path=/
```

**In Rack 2:** Headers were a hash with string values. To send multiple headers, you had to join them with a newline `\n`.
```ruby
headers["Set-Cookie"] = "session_id=123; Path=/\ntheme=dark; Path=/"
```

**In Rack 3:** Headers can be Arrays! (This was a huge and welcome change).
```ruby
headers["Set-Cookie"] = ["session_id=123; Path=/", "theme=dark; Path=/"]
```

If you are writing middleware that manipulates headers, you **must** check if the header value is an Array or a String to be compatible with Rack 3.

```ruby
# Safe way to append a header in Rack 3
if headers["Set-Cookie"].is_a?(Array)
  headers["Set-Cookie"] << "new_cookie=val"
else
  # Fallback for Rack 2
  existing = headers["Set-Cookie"] || ""
  headers["Set-Cookie"] = existing.empty? ? "new_cookie=val" : "#{existing}\nnew_cookie=val"
end
```

## Edge Case 4: The PATH_INFO vs SCRIPT_NAME Split

When you mount a Rack app inside another (e.g., mounting Sidekiq Web at `/sidekiq`), Rack splits the URL.

Request: `GET /sidekiq/queues`

- `env["SCRIPT_NAME"]` = `"/sidekiq"` (The path to the mount point)
- `env["PATH_INFO"]` = `"/queues"` (The path *inside* the mounted app)

If your middleware needs to log the *full* path, `PATH_INFO` is not enough! You must use `Rack::Request#path` which combines them, or combine them yourself: `env["SCRIPT_NAME"] + env["PATH_INFO"]`.

---

# 3. Comparison Tables

Quick reference guides for your interview.

## Servers

| Server | Concurrency Model | Best For | Notes |
|--------|-------------------|----------|-------|
| **Puma** | Multi-process + Multi-thread | Standard Rails apps | The default. Excellent balance of throughput and memory. |
| **Unicorn** | Multi-process (Workers) | Fast, short requests | Old reliable. Kills slow workers. High memory footprint. |
| **Falcon** | Fibers (Async) | WebSockets, high I/O wait | Native HTTP/2. Handles 10k+ connections easily. |
| **Passenger** | Processes/Threads + Web Server | Easy deployment | Integrates directly into Nginx/Apache. |

## Frameworks

| Framework | Relationship to Rack | Best For |
|-----------|----------------------|----------|
| **Rails** | Massive Rack app (`Rails.application`) | Full-stack, convention over configuration. |
| **Sinatra** | Thin wrapper over Rack | Microservices, tiny APIs. |
| **Grape** | Rack app optimized for APIs | RESTful APIs, versioning, parameter validation. |
| **Hanami** | Modular Rack app | Clean architecture, isolation, fast boot. |

## Request Properties (Plain Rack vs ActionDispatch)

| Property | Plain `Rack::Request` | `ActionDispatch::Request` (Rails) |
|----------|-----------------------|-----------------------------------|
| `params` | Query + Form Body | Query + Form Body + **Route params** + JSON parsing |
| `ip` | `REMOTE_ADDR` (Often proxy IP) | Parses `X-Forwarded-For` to find true IP |
| `session`| Requires `Rack::Session` middleware | Built-in via `CookieStore` |
| `format` | N/A | Detects `:html`, `:json`, `:xml` via `Accept` header |

---

# 4. Interview Questions

If you understand the answers to these questions, you are ready for any senior backend interview involving Rack.

### Core Architecture

**1. What is Rack, and why does the Ruby ecosystem use it?**
*Answer:* Rack is a standard interface connecting Ruby web servers (Puma) to Ruby frameworks (Rails). It exists so that servers don't need to write framework-specific code, and frameworks don't need to write server-specific code. It’s an adapter pattern.

**2. Describe the Rack specification contract.**
*Answer:* A Rack application is an object that responds to `call`. It takes one argument, `env` (a hash of the HTTP request). It returns exactly a three-element array: `[status (Integer), headers (Hash), body (Enumerable responding to each)]`.

**3. Why is the Rack body an enumerable object instead of a String?**
*Answer:* To support streaming. If the body was a string, a 5GB file download would require 5GB of RAM. By requiring `each`, the server can read and transmit the response in small chunks, keeping memory usage low.

### Middleware

**4. What is the difference between `Rack::Builder`'s `use` and `run`?**
*Answer:* `use` adds a middleware layer to the stack. `run` defines the terminal application at the center of the stack. You can have many `use` declarations, but only one `run`.

**5. How does middleware short-circuiting work, and when would you use it?**
*Answer:* Short-circuiting happens when a middleware returns a response array `[status, headers, body]` *without* calling `@app.call(env)`. This halts the request from going deeper into the stack. It's used for authentication (returning 401 early), rate limiting (429 early), or caching (304 Not Modified).

**6. Walk me through the exact order of execution in a middleware stack.**
*Answer:* It follows the Onion Model. The request travels IN through the middleware top-to-bottom. It hits the core app. The response travels OUT through the middleware bottom-to-top.
`Request -> Middleware A (before) -> Middleware B (before) -> App -> Middleware B (after) -> Middleware A (after) -> Response`.

### Rails Specifics

**7. Where does ActionDispatch fit into this picture?**
*Answer:* ActionDispatch is the layer in Rails that bridges Rack and your controllers. It provides the Rails middleware stack, the router (`ActionDispatch::Routing`), and enhances the request/response objects (`ActionDispatch::Request`).

**8. Why does `Rack::MethodOverride` exist in Rails?**
*Answer:* HTML forms natively only support GET and POST. Rails encourages RESTful routes (PUT/PATCH/DELETE). `MethodOverride` looks for a hidden `_method` parameter in a POST request and rewrites `env["REQUEST_METHOD"]` so the Rails router sees the intended HTTP verb.

**9. How do you view the middleware stack in a Rails app, and how do you insert custom middleware?**
*Answer:* View it by running `bin/rails middleware`. Insert custom middleware in `config/application.rb` using `config.middleware.use MyClass`, `insert_before`, or `insert_after`.

### Concurrency and Thread Safety

**10. What is a race condition in Rack middleware? How do you prevent it?**
*Answer:* Because servers like Puma share a single middleware instance across multiple threads, storing per-request state in an instance variable (e.g., `@current_user = ...`) causes a race condition where threads overwrite each other's data. Prevent it by using local variables in the `call` method, storing data in the `env` hash, or using a Mutex for genuinely shared state.

**11. Explain how Puma handles concurrency given Ruby's Global VM Lock (GVL).**
*Answer:* Puma uses a multi-process, multi-threaded model. While the GVL prevents Ruby code from executing in parallel within a single process, the GVL is released when a thread waits for I/O (like a DB query or HTTP request). This allows other threads in the same process to execute Ruby code while the first thread waits.

### Advanced

**12. What is Rack Hijacking?**
*Answer:* Hijacking allows an application to bypass the standard Rack request/response cycle and take raw ownership of the underlying TCP socket. It's used for protocols that require persistent, bi-directional communication, most notably WebSockets (which Action Cable uses internally).

**13. What happens if you read `request.body.read` in a middleware but forget to rewind it?**
*Answer:* The IO pointer remains at the end of the stream. When the request reaches the controller, `params` will be empty (if it was a POST/JSON body) because ActionDispatch will try to read an already-exhausted stream. Always call `request.body.rewind`.

---

# 5. Coding Exercises

Try writing these on your own without looking at the solutions.

### Exercise 1: The "Down for Maintenance" Middleware
**Task:** Write a middleware that checks if a file named `tmp/stop.txt` exists. If it does, return a 503 status with the body "Maintenance". If it doesn't, process the request normally.

<details>
<summary>Click to view solution</summary>

```ruby
class MaintenanceMiddleware
  def initialize(app)
    @app = app
  end

  def call(env)
    if File.exist?("tmp/stop.txt")
      [503, { "Content-Type" => "text/plain" }, ["Maintenance"]]
    else
      @app.call(env)
    end
  end
end
```
</details>

### Exercise 2: The "JSON Enforcer" Middleware
**Task:** Write a middleware for an API. If a POST or PUT request comes in, ensure the `Content-Type` header is exactly `application/json`. If it's not, return a 415 Unsupported Media Type.

<details>
<summary>Click to view solution</summary>

```ruby
class JsonEnforcer
  def initialize(app)
    @app = app
  end

  def call(env)
    request = Rack::Request.new(env)

    if (request.post? || request.put?) && request.content_type != "application/json"
      return [415, { "Content-Type" => "application/json" }, ['{"error":"Must send JSON"}']]
    end

    @app.call(env)
  end
end
```
</details>

### Exercise 3: The "Response Appender" Middleware
**Task:** Write a middleware that intercepts the response from the app and appends the string `" - Appended by Middleware!"` to the end of the body. (Assume the body is small and it's safe to load into memory).

<details>
<summary>Click to view solution</summary>

```ruby
class ResponseAppender
  def initialize(app)
    @app = app
  end

  def call(env)
    status, headers, body = @app.call(env)

    # We must iterate over the existing body
    full_body = ""
    body.each { |chunk| full_body << chunk }
    
    # Close original body if necessary
    body.close if body.respond_to?(:close)

    new_body = full_body + " - Appended by Middleware!"
    
    # Update Content-Length!
    headers["Content-Length"] = new_body.bytesize.to_s

    [status, headers, [new_body]]
  end
end
```
</details>

---

# 6. Cheat Sheet

Copy this and review it the morning of your interview.

### The Contract
- Responds to `call(env)`.
- Returns `[status (Integer), headers (Hash), body (responds to each)]`.

### The Layers
1. **Server** (Puma/Unicorn): Speaks HTTP to network, speaks Rack to app.
2. **Rack Middleware**: Intercepts `env` in, intercepts response out.
3. **ActionDispatch**: Rails' Rack layer (Routing, Cookies, Sessions).
4. **App** (Rails Controller): Business logic.

### `Rack::Builder` (`config.ru`)
- `use Middleware`: Adds layer.
- `run App`: Terminates stack.
- `map '/path'`: Branches to a different app.

### The `env` Hash
- `REQUEST_METHOD` (GET, POST)
- `PATH_INFO` (/users)
- `QUERY_STRING` (page=1)
- `HTTP_*` (Headers, e.g. `HTTP_USER_AGENT`)
- `rack.input` (The request body IO)

### Thread Safety Golden Rule
**Never store per-request state in an instance variable (`@var`) inside a middleware.** It will be shared across threads and cause race conditions. Use local variables or store data in `env["my_namespace.var"]`.

### Important Rails Middleware
- `HostAuthorization`: DNS rebinding protection.
- `RemoteIp`: IP spoofing protection.
- `RequestId`: Traceability (`X-Request-Id`).
- `MethodOverride`: Lets HTML forms use PUT/PATCH/DELETE.
- `CookieStore`: Encrypted session storage.

---

# 7. Summary

You did it. 

You started by asking "What is Rack?", and you ended up learning how to proxy traffic to microservices, how the Global VM Lock interacts with Puma threads, and how to write thread-safe, high-performance middleware.

Understanding Rack is the dividing line between a developer who *uses* Rails, and an engineer who *understands* Rails. 

When you sit in your senior backend interview, and they ask you how an HTTP request actually turns into a controller action, you won't just say "routing." You will talk about the web server building the environment hash, the Onion model of the middleware stack, how ActionDispatch enhances the request, and how the array response bubbles back up through Puma to the client.

You are ready.

---

# 8. Resources

If you want to keep exploring, here are the best places to look:

1. **The Source Code:** The best way to learn is to read. Open your terminal and run `bundle open rack`. Look at `lib/rack/builder.rb` and `lib/rack/request.rb`.
2. **The Rack Spec:** Read the official specification. It's surprisingly short and readable. [github.com/rack/rack/blob/main/SPEC.rdoc](https://github.com/rack/rack/blob/main/SPEC.rdoc)
3. **Rails Guides - Rails on Rack:** The official documentation on how Rails uses Rack. [guides.rubyonrails.org/rails_on_rack.html](https://guides.rubyonrails.org/rails_on_rack.html)
4. **Puma Documentation:** Understand the architecture of Puma to better understand concurrency. [github.com/puma/puma/tree/master/docs](https://github.com/puma/puma/tree/master/docs)
5. **Falcon & Async:** Explore the future of Ruby concurrency. [socketry.github.io/falcon](https://socketry.github.io/falcon/)

Good luck with your interviews!

---
*End of Module 2 - Rack Master Study Guide*
