# Rack Application

> **Interview target:** Senior Backend Engineer / Staff Engineer
> **Prerequisites:** Ruby, Rails basics, HTTP basics, Rack Environment
> **Curriculum position:** Rack → Rack Environment → **Rack Application** → Rack Middleware → Request Lifecycle

---

# 1. Overview

## 1.1 What is a Rack Application?

A **Rack Application** is any Ruby object that conforms to the Rack interface.

At its simplest, a Rack application is an object that responds to:

```ruby
def call(env)
  # ...
end
```

and returns:

```ruby
[
  status,
  headers,
  body
]
```

Conceptually:

```text
HTTP Request
     │
     ▼
Web Server
     │
     │ creates Rack environment
     ▼
Rack Application
     │
     │ call(env)
     ▼
[status, headers, body]
     │
     ▼
Web Server
     │
     ▼
HTTP Response
```

This is one of the most important abstractions underneath Rails.

Rails itself is a Rack application.

The Rails Guides explicitly identify `Rails.application` as the primary Rack application object of a Rails application, and Rack-compliant web servers can use that object to serve Rails.

---

## 1.2 The Core Rack Contract

The fundamental contract is:

```ruby
response = app.call(env)
```

where:

```ruby
response = [
  status,
  headers,
  body
]
```

For example:

```ruby
class HelloApp
  def call(env)
    [
      200,
      { "Content-Type" => "text/plain" },
      ["Hello World"]
    ]
  end
end
```

The important thing isn't the class.

The important thing is:

```ruby
call(env)
```

and the response triple.

---

## 1.3 Why Does Rack Have This Abstraction?

Before Rack became common, Ruby web frameworks and web servers had more framework-specific interfaces.

Rack created a common interface between:

```text
Web Server
     │
     ▼
Rack
     │
     ▼
Web Application
```

That means the server doesn't need to know whether the application is:

* Rails
* Sinatra
* Hanami
* a custom Ruby application
* a tiny Ruby script

It only needs to know:

> "I have a Rack application. I give it an environment and receive a response."

This decoupling is the central architectural value of Rack.

---

# 2. The Most Important Mental Model

You should be able to explain Rack Application in one sentence during an interview:

> **A Rack application is a Ruby callable object that accepts a Rack environment and returns an HTTP response represented as `[status, headers, body]`.**

Everything else builds on this.

---

# 3. The Three Parts of a Rack Application

A Rack application has three important pieces:

```text
             Rack Application
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
     Input       Processing     Output
       │                         │
     env                       response
       │                         │
       └────── call(env) ────────┘
```

## 3.1 Input

The input is:

```ruby
env
```

The Rack environment contains request information.

For example:

```ruby
env["REQUEST_METHOD"]
env["PATH_INFO"]
env["QUERY_STRING"]
env["rack.input"]
```

---

## 3.2 Processing

The application examines the environment and decides what to do.

Example:

```ruby
def call(env)
  if env["PATH_INFO"] == "/hello"
    ...
  else
    ...
  end
end
```

---

## 3.3 Output

The application returns:

```ruby
[
  status,
  headers,
  body
]
```

Example:

```ruby
[
  200,
  { "Content-Type" => "text/plain" },
  ["Hello"]
]
```

---

# 4. What Makes Something a Rack Application?

A very common interview misconception is:

> "A Rack application has to inherit from a Rack class."

No.

There is no requirement for inheritance.

This works:

```ruby
class MyApp
  def call(env)
    [200, {}, ["Hello"]]
  end
end
```

So does this:

```ruby
app = lambda do |env|
  [200, {}, ["Hello"]]
end
```

And conceptually even:

```ruby
app = Object.new

def app.call(env)
  [200, {}, ["Hello"]]
end
```

The important property is the callable interface.

---

# 5. Rack Application vs Rack Server

This distinction is extremely important.

## Rack Application

Responsible for:

* interpreting the request
* executing application logic
* generating the response

Example:

```ruby
Rails.application
```

## Rack Server

Responsible for:

* listening for network connections
* accepting HTTP requests
* creating the Rack environment
* invoking the Rack application
* translating the Rack response back into HTTP

Examples historically/currently include servers such as:

* Puma
* Unicorn
* Passenger
* other Rack-compatible servers

Conceptually:

```text
                    Web Server
                       │
                 HTTP request
                       │
                       ▼
                Build env Hash
                       │
                       ▼
                 app.call(env)
                       │
                       ▼
              [status, headers, body]
                       │
                       ▼
                HTTP response
```

The server and application are deliberately decoupled.

---

# 6. Rack Application vs Rails Application

These are related but not identical concepts.

A Rails application is a sophisticated Rack application.

```text
Rails Application
       │
       └── implements Rack application interface
```

Therefore:

```ruby
Rails.application.call(env)
```

is conceptually a valid Rack invocation.

Rails does not throw away Rack.

Instead, Rails builds a large application pipeline around the Rack contract.

The Rails Guides describe `Rails.application` as the primary Rack application object.

---

# 7. The Simplest Rack Application

The smallest useful example:

```ruby
class App
  def call(env)
    [
      200,
      { "Content-Type" => "text/plain" },
      ["Hello Rack"]
    ]
  end
end
```

Then:

```ruby
app = App.new
```

Now:

```ruby
app.call(env)
```

returns:

```ruby
[
  200,
  { "Content-Type" => "text/plain" },
  ["Hello Rack"]
]
```

---

# 8. Why Is the Body an Enumerable?

This is a subtle but important Rack concept.

You might initially expect:

```ruby
"Hello Rack"
```

But Rack applications conventionally return a body that can be enumerated.

For example:

```ruby
["Hello Rack"]
```

The server can iterate over it.

Conceptually:

```ruby
body.each do |chunk|
  send_to_client(chunk)
end
```

This allows responses to be produced incrementally.

For example:

```ruby
class StreamingApp
  def call(env)
    body = Enumerator.new do |yielder|
      10.times do |i|
        yielder << "chunk #{i}\n"
      end
    end

    [
      200,
      { "Content-Type" => "text/plain" },
      body
    ]
  end
end
```

This becomes important when studying:

* streaming
* large responses
* memory usage
* Server-Sent Events
* HTTP chunking
* Rack hijacking

---

# 9. Status

The first element is the HTTP status.

Example:

```ruby
200
```

Other examples:

```ruby
201
204
301
400
401
403
404
500
```

Example:

```ruby
[
  404,
  { "Content-Type" => "text/plain" },
  ["Not Found"]
]
```

Rack deals with the response at the HTTP abstraction level.

---

# 10. Headers

The second element is a headers collection.

Example:

```ruby
{
  "Content-Type" => "application/json",
  "X-Request-ID" => "abc123"
}
```

For example:

```ruby
class ApiApp
  def call(env)
    [
      200,
      {
        "Content-Type" => "application/json",
        "Cache-Control" => "no-store"
      },
      ['{"ok":true}']
    ]
  end
end
```

The web server eventually turns these into HTTP response headers.

---

# 11. Body

The body must be something the server can consume according to the Rack response contract.

Typical example:

```ruby
["Hello"]
```

Another:

```ruby
["Hello ", "Rack"]
```

This distinction matters:

```ruby
["Hello ", "Rack"]
```

is conceptually two body chunks.

It doesn't necessarily mean two HTTP responses.

---

# 12. Rack Application Lifecycle

A simplified lifecycle:

```text
1. Client sends HTTP request
             │
             ▼
2. Web server accepts request
             │
             ▼
3. Server constructs Rack env
             │
             ▼
4. Server invokes app.call(env)
             │
             ▼
5. Application examines env
             │
             ▼
6. Application performs business logic
             │
             ▼
7. Application returns
   [status, headers, body]
             │
             ▼
8. Server consumes response
             │
             ▼
9. Server sends HTTP response
```

The key boundary is:

```ruby
app.call(env)
```

---

# 13. Request Example

Suppose the client sends:

```http
GET /users/42 HTTP/1.1
Host: example.com
Accept: application/json
```

The server constructs an environment approximately containing:

```ruby
{
  "REQUEST_METHOD" => "GET",
  "PATH_INFO" => "/users/42",
  "QUERY_STRING" => "",
  "HTTP_HOST" => "example.com",
  "HTTP_ACCEPT" => "application/json",
  ...
}
```

Then:

```ruby
app.call(env)
```

The application might return:

```ruby
[
  200,
  {
    "Content-Type" => "application/json"
  },
  ['{"id":42}']
]
```

The server converts that into:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"id":42}
```

---

# 14. A More Realistic Rack Application

```ruby
class Router
  def call(env)
    case env["PATH_INFO"]
    when "/"
      [
        200,
        { "Content-Type" => "text/plain" },
        ["Home"]
      ]

    when "/health"
      [
        200,
        { "Content-Type" => "text/plain" },
        ["OK"]
      ]

    else
      [
        404,
        { "Content-Type" => "text/plain" },
        ["Not Found"]
      ]
    end
  end
end
```

This is already a tiny web framework.

It has:

* request inspection
* routing
* response generation
* HTTP status handling

Frameworks add many layers on top.

---

# 15. Rack Application as a Function

The Rack contract is especially elegant because it resembles:

```text
Request
   │
   ▼
Function
   │
   ▼
Response
```

In Ruby:

```ruby
app.call(env)
```

This makes Rack applications composable.

For example:

```ruby
def app(env)
  ...
end
```

and:

```ruby
lambda { |env| ... }
```

can both act as applications if they satisfy the callable contract.

---

# 16. Procs and Lambdas as Rack Applications

Example:

```ruby
app = lambda do |env|
  [
    200,
    { "Content-Type" => "text/plain" },
    ["Hello"]
  ]
end
```

Then:

```ruby
app.call(env)
```

works.

This is useful because it demonstrates that Rack doesn't care about object-oriented structure.

It cares about behavior.

This is classic Ruby duck typing.

---

# 17. `config.ru`

A major Rack concept is `config.ru`.

A Rack application can be described through a Rack configuration file.

Example:

```ruby
run MyApp.new
```

For Rails, the application can be exposed as:

```ruby
require_relative "config/environment"

run Rails.application
```

The Rails documentation explicitly shows this pattern for running Rails through Rack.

---

# 18. What Does `run` Mean?

Conceptually:

```ruby
run MyApp.new
```

means:

> "This is the Rack application that should receive requests."

It identifies the final application in the Rack configuration.

This becomes especially interesting once middleware is introduced:

```ruby
use LoggingMiddleware
use AuthMiddleware

run MyApp.new
```

Now the architecture becomes:

```text
Request
   │
   ▼
LoggingMiddleware
   │
   ▼
AuthMiddleware
   │
   ▼
MyApp
```

We will study this deeply in **Rack Middleware**.

---

# 19. `Rack::Builder`

Rack provides a way to construct application pipelines.

Conceptually:

```ruby
Rack::Builder.new do
  use SomeMiddleware
  use AnotherMiddleware

  run MyApp.new
end
```

This creates a composite application.

That is an extremely important insight:

> A Rack application doesn't have to be the final business application.

A Rack application can itself be a composition of other Rack applications.

---

# 20. Composition

Suppose:

```ruby
app = MyApp.new
```

and:

```ruby
middleware = LoggingMiddleware.new(app)
```

Now:

```ruby
middleware.call(env)
```

eventually calls:

```ruby
app.call(env)
```

This creates a chain:

```text
A
│
▼
B
│
▼
C
│
▼
Application
```

Every layer obeys the same general contract.

This is why Rack is so composable.

---

# 21. Rack Application as a Pipeline

Imagine:

```text
                  Rack Application
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
      Auth             Logging          Metrics
        │                │                │
        └────────────────┼────────────────┘
                         ▼
                      Rails
```

The exact implementation is a nested call chain rather than a parallel graph:

```text
Auth
 └── Logging
      └── Metrics
           └── Rails
```

For example:

```ruby
Auth.new(
  Logging.new(
    Metrics.new(
      Rails.application
    )
  )
)
```

This is one of the most important mental models for understanding Rails middleware.

---

# 22. Rails.application

In Rails:

```ruby
Rails.application
```

is an instance of the application's class, generally:

```ruby
MyApp::Application
```

which inherits through Rails' application hierarchy.

It provides the Rack-compatible entry point.

Therefore, conceptually:

```ruby
Rails.application.call(env)
```

crosses the boundary from:

```text
Rack world
```

into:

```text
Rails world
```

The Rails Guides explicitly state that `Rails.application` is the primary Rack application object.

---

# 23. What Happens Inside Rails?

This is where the topic becomes senior-level.

The simplistic model is:

```text
env
 ↓
Rails.application.call
 ↓
controller
```

That is wrong.

A better model is:

```text
Web Server
    │
    ▼
Rack environment
    │
    ▼
Rails middleware stack
    │
    ▼
Rails application
    │
    ▼
Router
    │
    ▼
Controller
    │
    ▼
Action
    │
    ▼
Response
```

And the middleware stack can contain many components before the request reaches routing.

Rails uses `ActionDispatch::MiddlewareStack` to assemble its Rack middleware pipeline.

---

# 24. Rails Middleware Stack

You can inspect the stack with:

```bash
bin/rails middleware
```

The exact stack varies by Rails version and application configuration.

For example, Rails documentation shows middleware such as:

```text
ActionDispatch::HostAuthorization
Rack::Sendfile
ActionDispatch::Static
ActionDispatch::Executor
Rack::Runtime
Rack::MethodOverride
ActionDispatch::RequestId
ActionDispatch::RemoteIp
Rails::Rack::Logger
ActionDispatch::ShowExceptions
ActionDispatch::DebugExceptions
ActionDispatch::Reloader
ActionDispatch::Callbacks
ActionDispatch::Cookies
ActionDispatch::Session::CookieStore
ActionDispatch::Flash
Rack::Head
Rack::ConditionalGet
Rack::ETag
Rack::TempfileReaper
run MyApp::Application.routes
```

The exact ordering is version/configuration dependent, so never memorize one generated stack as universally correct.

---

# 25. The Important Architectural Boundary

For interview purposes, remember:

```text
HTTP
 │
 ▼
Web Server
 │
 ▼
Rack
 │
 ▼
Rack Application
 │
 ▼
Rails Middleware
 │
 ▼
Rails Routing
 │
 ▼
Controller
 │
 ▼
Application Code
```

Rack is therefore **below Rails MVC**.

It is part of the infrastructure that allows Rails to receive web requests.

---

# 26. Rack Application and Rails Routing

A subtle distinction:

```text
Rack Application
```

is not synonymous with:

```text
Rails Router
```

The Rack application is the larger callable system.

The router is one component inside that system.

Conceptually:

```text
Rails.application
      │
      ▼
middleware
      │
      ▼
routes
      │
      ▼
controller
```

The router determines which Rails endpoint should handle the request.

---

# 27. Directly Calling a Rack Application

For experimentation:

```ruby
env = {
  "REQUEST_METHOD" => "GET",
  "PATH_INFO" => "/",
  "QUERY_STRING" => "",
  "rack.input" => StringIO.new
}

app.call(env)
```

This is useful for understanding Rack without involving an actual TCP connection.

You are manually performing the server → application boundary.

---

# 28. A Minimal Rack Environment

A realistic Rack environment has many keys.

A toy environment might look like:

```ruby
{
  "REQUEST_METHOD" => "GET",
  "SCRIPT_NAME" => "",
  "PATH_INFO" => "/hello",
  "QUERY_STRING" => "",
  "SERVER_NAME" => "localhost",
  "SERVER_PORT" => "3000",
  "SERVER_PROTOCOL" => "HTTP/1.1",
  "rack.url_scheme" => "http",
  "rack.input" => StringIO.new,
  "rack.errors" => $stderr
}
```

The exact required environment depends on the Rack specification/version and server.

This is why you should not casually construct production Rack environments by hand.

---

# 29. Why `env` Is a Hash

The Rack environment is intentionally generic.

The web server doesn't need to know your framework.

The framework doesn't need to know the server's internal implementation.

They communicate through agreed keys.

This creates:

```text
Server implementation
       │
       │ standardized env
       ▼
Rack application
```

rather than:

```text
Puma-specific API
Unicorn-specific API
Passenger-specific API
...
```

---

# 30. Rack Extensions

Rack also uses namespaced keys.

Examples:

```ruby
env["rack.input"]
env["rack.errors"]
env["rack.url_scheme"]
```

HTTP headers are commonly exposed as:

```ruby
env["HTTP_ACCEPT"]
env["HTTP_HOST"]
env["HTTP_AUTHORIZATION"]
```

This provides a common vocabulary between servers and applications.

---

# 31. HTTP Headers vs Rack Environment

An important interview question:

> "Does Rack env contain HTTP headers directly?"

Not exactly.

Rack maps HTTP request information into environment keys.

For example:

```http
Host: example.com
```

becomes conceptually:

```ruby
env["HTTP_HOST"]
```

Similarly:

```http
Authorization: Bearer ...
```

becomes:

```ruby
env["HTTP_AUTHORIZATION"]
```

There are also special Rack variables that are not ordinary HTTP headers:

```ruby
env["rack.input"]
env["rack.errors"]
env["rack.url_scheme"]
```

---

# 32. `rack.input`

Request bodies are especially important.

For:

```http
POST /users
Content-Type: application/json

{"name":"Alice"}
```

the request body is made available through:

```ruby
env["rack.input"]
```

The application can read it:

```ruby
body = env["rack.input"].read
```

For example:

```ruby
class JsonApp
  def call(env)
    body = env["rack.input"].read

    [
      200,
      { "Content-Type" => "text/plain" },
      ["Received: #{body}"]
    ]
  end
end
```

---

# 33. Important `rack.input` Mistake

This is dangerous:

```ruby
env["rack.input"].read
```

followed later by:

```ruby
env["rack.input"].read
```

The second read may return nothing because the stream has already been consumed.

That becomes particularly important in middleware.

For example:

```text
Middleware
   │
   ├── reads request body
   │
   ▼
Rails
   │
   └── expects request body
```

If the middleware consumes the stream incorrectly, downstream code may see an empty body.

Senior engineers should recognize this immediately during debugging.

---

# 34. Rack Application and Request Body Rewinding

Depending on the server/Rack implementation and stream involved, middleware may need to preserve/rewind request input when appropriate.

Conceptually:

```ruby
input = env["rack.input"]

body = input.read

input.rewind
```

But blindly rewinding every input is not a universal solution.

You need to understand:

* whether the stream supports rewind
* whether the body is large
* whether buffering creates memory pressure
* whether downstream code expects a particular stream state

---

# 35. Response Body Lifecycle

Consider:

```ruby
body = ["hello", "world"]
```

The server can conceptually do:

```ruby
body.each do |chunk|
  write(chunk)
end
```

Therefore the body is not simply:

```text
"response string"
```

It is a response-producing object.

This is fundamental to streaming.

---

# 36. Close Semantics

Some response bodies may expose lifecycle behavior such as:

```ruby
body.close
```

A server/framework can use this to clean up resources after iteration.

This matters for:

* streaming
* files
* custom enumerators
* resource management

A senior engineer should understand that the response body has a lifecycle, not merely a string value.

---

# 37. Lazy Response Bodies

You can generate data lazily:

```ruby
body = Enumerator.new do |y|
  100.times do |i|
    y << "line #{i}\n"
  end
end
```

This avoids constructing one giant string in memory.

But lazy generation does not magically make an endpoint cheap.

You still need to consider:

* CPU
* network throughput
* database access
* connection lifetime
* client disconnects
* server worker/thread occupancy

---

# 38. Rack Application and Concurrency

A Rack application can be called concurrently.

For example, a threaded server may have:

```text
Thread 1 → app.call(env1)
Thread 2 → app.call(env2)
Thread 3 → app.call(env3)
```

Therefore application objects should generally be designed with concurrency in mind.

This is especially important because Rack application objects can be long-lived.

Bad:

```ruby
class App
  def initialize
    @current_user = nil
  end

  def call(env)
    @current_user = ...
  end
end
```

If the same application object serves concurrent requests, mutable instance state can cause race conditions.

Prefer request-local state:

```ruby
def call(env)
  current_user = ...
end
```

---

# 39. Rack Application Objects Are Usually Long-Lived

A common misconception is:

> "Rails creates a new application object for every request."

Usually, no.

The application object is initialized during application boot and then used to process requests.

Conceptually:

```text
Process
 │
 ├── Rails.application
 │
 ├── Request 1 → call(env1)
 │
 ├── Request 2 → call(env2)
 │
 ├── Request 3 → call(env3)
 │
 └── ...
```

This is one reason global mutable state in Rack/Rails applications is dangerous.

---

# 40. Rack Application and Thread Safety

Suppose:

```ruby
class App
  def initialize
    @counter = 0
  end

  def call(env)
    @counter += 1

    [
      200,
      {},
      ["#{@counter}"]
    ]
  end
end
```

This looks harmless.

Under concurrent execution, it introduces synchronization concerns.

The problem isn't Rack itself.

The problem is shared mutable state.

A senior engineer should ask:

> Is this state request-local, thread-local, process-local, or shared across workers?

---

# 41. Rack Application and Processes

A production Rails deployment might look like:

```text
Load Balancer
      │
 ┌────┼────┐
 ▼    ▼    ▼
Puma Puma Puma
 │    │    │
 ▼    ▼    ▼
Rails Rails Rails
```

Each process has its own application object and memory.

Therefore:

```ruby
@cache = {}
```

does not automatically mean:

```text
one global cache across all Rails processes
```

Instead, each process may have its own copy.

This is a critical production distinction.

---

# 42. Application State vs External State

If state must be shared across processes, use an external system such as:

```text
Redis
PostgreSQL
external cache
message broker
```

rather than assuming Rack/Rails application instance variables are shared.

This connects Rack Application to later curriculum topics:

* concurrency
* Redis
* caching
* background jobs
* deployment
* load balancing

---

# 43. Error Handling

What happens if:

```ruby
app.call(env)
```

raises?

Example:

```ruby
class App
  def call(env)
    raise "boom"
  end
end
```

The exception propagates upward unless something catches it.

In a Rails application, middleware participates in exception handling.

For example, Rails includes middleware for exception handling in its stack.

Conceptually:

```text
Exception
   │
   ▼
Application
   │
   ▲
Exception middleware
   │
   ▼
HTTP 500 response
```

This is another reason middleware ordering matters.

---

# 44. Rack Application and Observability

Because every request passes through:

```ruby
call(env)
```

Rack is a powerful observability boundary.

A middleware can record:

```ruby
start = Process.clock_gettime(Process::CLOCK_MONOTONIC)

status, headers, body = @app.call(env)

duration = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start
```

Then emit:

```text
request duration
status
path
request ID
```

This pattern underlies many real production observability mechanisms.

---

# 45. A Production-Style Timing Application

```ruby
class TimingMiddleware
  def initialize(app)
    @app = app
  end

  def call(env)
    start = Process.clock_gettime(Process::CLOCK_MONOTONIC)

    status, headers, body = @app.call(env)

    duration =
      Process.clock_gettime(Process::CLOCK_MONOTONIC) - start

    puts "#{env['REQUEST_METHOD']} #{env['PATH_INFO']} #{duration}s"

    [status, headers, body]
  end
end
```

This is technically middleware, but it demonstrates a key property:

> A middleware is itself a Rack application.

That relationship becomes central in the next topic.

---

# 46. Rack Application and Short-Circuiting

A Rack-compatible layer does not necessarily have to call the downstream application.

For example:

```ruby
class HealthCheck
  def initialize(app)
    @app = app
  end

  def call(env)
    if env["PATH_INFO"] == "/health"
      [
        200,
        { "Content-Type" => "text/plain" },
        ["OK"]
      ]
    else
      @app.call(env)
    end
  end
end
```

This creates:

```text
/health
   │
   └── HealthCheck → response

/anything-else
   │
   ▼
HealthCheck
   │
   ▼
Rails
```

This pattern is extremely common.

---

# 47. Why Short-Circuiting Matters

It allows requests to be handled before reaching the expensive application stack.

Examples:

* health checks
* maintenance mode
* IP filtering
* authentication
* rate limiting
* static files
* redirects
* feature gates

For example:

```text
Request
  │
  ▼
Rate limiter
  │
  ├── rejected → 429
  │
  ▼
Authentication
  │
  ├── rejected → 401
  │
  ▼
Rails
```

---

# 48. Performance Implications

Rack sits on the request path.

Therefore every middleware/application layer can affect latency.

If you have:

```text
A
 ↓
B
 ↓
C
 ↓
D
 ↓
Rails
```

every request crosses all these layers unless something short-circuits.

If each layer adds:

```text
0.2 ms
```

and there are 20 layers:

```text
~4 ms
```

of additional overhead in the simplified example.

Real overhead is workload-dependent, but the principle is important.

---

# 49. Middleware Ordering and Performance

Consider:

```text
ExpensiveAuthentication
      ↓
HealthCheck
```

versus:

```text
HealthCheck
      ↓
ExpensiveAuthentication
```

For health checks, the second can avoid unnecessary work.

Similarly:

```text
Rate Limit
   ↓
Authentication
   ↓
Rails
```

may prevent abusive requests from reaching expensive downstream processing.

But ordering must preserve correctness.

---

# 50. Rack Application and Security

Rack is close to the HTTP boundary, so it is a useful security enforcement point.

Potential concerns include:

* Host header attacks
* authentication
* request size limits
* rate limiting
* malicious headers
* malformed requests
* request body handling
* IP handling
* response security headers

Rails includes middleware such as `ActionDispatch::HostAuthorization` to protect against Host-header-related attacks and DNS rebinding scenarios.

---

# 51. Never Trust `env` Automatically

The Rack environment represents client/server request information.

That doesn't mean every value is trustworthy.

For example:

```ruby
env["HTTP_X_FORWARDED_FOR"]
```

should not automatically be treated as the real client IP.

In production you need to understand:

```text
Client
  ↓
Proxy
  ↓
Load Balancer
  ↓
Web Server
  ↓
Rack
```

and which proxies are trusted.

This connects directly to Rails' remote IP handling.

---

# 52. Rack Application and Proxies

Suppose:

```text
Browser
   │
   ▼
Cloud Load Balancer
   │
   ▼
Nginx
   │
   ▼
Puma
   │
   ▼
Rails
```

The Rack application sees the environment produced by the server/proxy chain.

Therefore:

```ruby
env["REMOTE_ADDR"]
```

and forwarded headers may not mean what a developer casually assumes.

This becomes critical for:

* rate limiting
* auditing
* security
* geolocation
* IP allowlists

---

# 53. Content-Length and Response Semantics

The application may return headers such as:

```ruby
{
  "Content-Type" => "text/plain",
  "Content-Length" => "5"
}
```

But developers should understand that response framing is a broader responsibility involving the Rack server and HTTP server behavior.

Don't treat Rack response headers as an isolated Ruby data structure.

They eventually become network protocol behavior.

---

# 54. Rack Application and HTTP Method

The application can inspect:

```ruby
env["REQUEST_METHOD"]
```

Example:

```ruby
case env["REQUEST_METHOD"]
when "GET"
  ...
when "POST"
  ...
when "DELETE"
  ...
end
```

Rails routing eventually provides a richer abstraction over this.

But at the Rack level, the method is just part of the environment.

---

# 55. Rack Application and Path

The application can inspect:

```ruby
env["PATH_INFO"]
```

Example:

```ruby
case env["PATH_INFO"]
when "/"
when "/health"
when "/users"
end
```

A real router abstracts this into route matching.

---

# 56. Building a Tiny Router

```ruby
class App
  def call(env)
    case [env["REQUEST_METHOD"], env["PATH_INFO"]]

    when ["GET", "/"]
      response(200, "Home")

    when ["GET", "/health"]
      response(200, "OK")

    when ["POST", "/users"]
      response(201, "Created")

    else
      response(404, "Not Found")
    end
  end

  private

  def response(status, body)
    [
      status,
      { "Content-Type" => "text/plain" },
      [body]
    ]
  end
end
```

You've now built the primitive core of:

```text
routing
```

inside a Rack application.

---

# 57. Rack Application and Framework Abstraction

Compare:

```ruby
env["PATH_INFO"]
```

with Rails:

```ruby
get "/users/:id"
```

The Rails abstraction provides:

* route matching
* path parameters
* named routes
* constraints
* controller dispatch
* middleware integration
* request/response abstractions

Rack intentionally provides much less.

That's the point.

---

# 58. Rack Is Not a Framework

A major interview distinction:

> Rack is an interface/specification and ecosystem, not a full MVC framework.

Rack does not inherently provide:

* models
* controllers
* ORM
* database access
* authentication
* business logic
* Rails routing conventions

It provides the foundational HTTP application interface and ecosystem around it.

---

# 59. Rack Application vs Rack Middleware

This distinction is critical.

A Rack application:

```ruby
app.call(env)
```

A Rack middleware:

```ruby
middleware.call(env)
```

But middleware typically receives another Rack application:

```ruby
middleware = Middleware.new(app)
```

Therefore:

```text
Middleware
     │
     └── wraps another Rack application
```

And because the wrapped object itself implements:

```ruby
call(env)
```

middleware can be chained.

---

# 60. The Universal Composition Model

This is the most useful diagram to memorize:

```text
                call(env)
                   │
                   ▼
          Middleware A
                   │
                   ▼
          Middleware B
                   │
                   ▼
          Middleware C
                   │
                   ▼
        Final Rack Application
                   │
                   ▼
       [status, headers, body]
```

The same interface appears at every layer.

That is why the architecture is composable.

---

# 61. Rails as a Rack Composition

A useful mental model:

```text
                 Rails.application
                       │
          ┌────────────┴────────────┐
          │                         │
      Middleware                Middleware
          │                         │
          └────────────┬────────────┘
                       ▼
                 Rails Router
                       │
                       ▼
                  Controller
                       │
                       ▼
                    Action
```

The exact implementation is more sophisticated, but this is the correct architectural model.

---

# 62. Boot-Time vs Request-Time

Another senior-level distinction:

## Boot time

Rails constructs/configures the application and middleware stack.

Conceptually:

```text
Process starts
   ↓
Rails boot
   ↓
Load configuration
   ↓
Initialize framework
   ↓
Build middleware stack
   ↓
Application ready
```

## Request time

```text
HTTP request
   ↓
server
   ↓
app.call(env)
   ↓
middleware
   ↓
router/controller
   ↓
response
```

Do not confuse:

```text
building the Rack application
```

with:

```text
executing the Rack application
```

---

# 63. Why This Matters

Suppose you write:

```ruby
class ExpensiveMiddleware
  def initialize(app)
    @app = app
    @expensive = load_huge_configuration
  end
end
```

That initialization happens when the middleware stack is constructed, not necessarily on every request.

This can be desirable.

But if you put:

```ruby
load_huge_configuration
```

inside:

```ruby
def call(env)
```

you may pay that cost on every request.

Understanding the application lifecycle lets you reason about this correctly.

---

# 64. Rails `config.middleware`

Rails provides middleware configuration such as:

```ruby
config.middleware.use MyMiddleware
```

or:

```ruby
config.middleware.insert_before(
  SomeMiddleware,
  MyMiddleware
)
```

and:

```ruby
config.middleware.insert_after(
  SomeMiddleware,
  MyMiddleware
)
```

Rails documentation explicitly provides these mechanisms for modifying the middleware stack.

This is the bridge between:

```text
Rack abstraction
```

and:

```text
Rails configuration
```

---

# 65. Why Middleware Position Matters

Suppose you want to inspect the authenticated user.

If authentication middleware comes later:

```text
YourMiddleware
   ↓
Authentication
   ↓
Rails
```

then your middleware cannot assume authentication has already happened.

Moving it:

```text
Authentication
   ↓
YourMiddleware
   ↓
Rails
```

changes semantics.

Therefore:

> Middleware order is part of application behavior.

It isn't merely organizational.

---

# 66. Common Mistakes

## Junior mistakes

### Mistake 1: Thinking Rack is the web server

Incorrect:

> "Puma is Rack."

Better:

> Puma is a Rack-compatible web server; Rack defines the interface between the server and application.

---

### Mistake 2: Thinking Rails is unrelated to Rack

Incorrect:

> "Rack is an optional Rails dependency."

Better:

> Rails exposes a Rack-compatible application and uses Rack-compatible middleware infrastructure.

---

### Mistake 3: Returning the wrong response

Wrong:

```ruby
def call(env)
  "hello"
end
```

Expected Rack response shape:

```ruby
[
  200,
  { "Content-Type" => "text/plain" },
  ["hello"]
]
```

---

## Mid-level mistakes

### Mistake 4: Mutating shared application state

Bad:

```ruby
@app_state[:user] = ...
```

without concurrency reasoning.

---

### Mistake 5: Consuming `rack.input`

Bad middleware:

```ruby
env["rack.input"].read
@app.call(env)
```

without considering downstream consumers.

---

### Mistake 6: Assuming middleware order doesn't matter

It absolutely does.

---

## Senior-level mistakes

### Mistake 7: Adding middleware without measuring

Every layer adds:

* execution
* allocations
* possible logging
* potential I/O
* latency

---

### Mistake 8: Putting expensive work into `call`

If it can safely be initialized once, doing it per request can be unnecessarily expensive.

---

### Mistake 9: Trusting proxy headers blindly

Headers such as forwarded IP information require a trusted proxy model.

---

# 67. Performance Considerations

## 67.1 Minimize per-request allocations

Avoid unnecessary:

```ruby
Array.new
Hash.new
String.new
```

inside extremely hot middleware.

But don't prematurely optimize tiny allocations without profiling.

---

## 67.2 Avoid unnecessary middleware

Ask:

> Does every request need this middleware?

If not, consider whether it belongs:

* at the edge
* conditionally
* in a controller
* in a background job
* elsewhere

---

## 67.3 Avoid unnecessary body buffering

For huge responses:

```ruby
body = huge_string
```

can create memory pressure.

Streaming/lazy enumeration can sometimes reduce memory usage.

But streaming can increase:

* request duration
* worker occupancy
* connection lifetime

So it is a trade-off, not an automatic optimization.

---

## 67.4 Avoid blocking I/O in unnecessary layers

A middleware that performs:

```ruby
Redis.call(...)
```

for every request can become a bottleneck.

At scale:

```text
100k requests/sec
×
1 Redis operation/request
```

is a very different architecture from:

```text
Redis only for authenticated API requests
```

---

# 68. Security Considerations

Rack applications sit close to the network boundary.

Pay special attention to:

### Host validation

Don't blindly trust:

```ruby
env["HTTP_HOST"]
```

Rails has Host Authorization middleware for this purpose.

### Request size

Large bodies can create:

* memory pressure
* CPU pressure
* denial-of-service opportunities

### Authentication

Authentication should happen before sensitive downstream work.

### IP addresses

Understand trusted proxies.

### Headers

Treat client-controlled headers as untrusted input unless your infrastructure explicitly establishes trust.

---

# 69. Debugging Rack Applications

When debugging a request, start at the boundary.

## Step 1 — Inspect middleware

```bash
bin/rails middleware
```

This shows the configured Rails middleware stack.

---

## Step 2 — Log request metadata

Temporarily inspect:

```ruby
env["REQUEST_METHOD"]
env["PATH_INFO"]
env["QUERY_STRING"]
env["HTTP_HOST"]
env["REMOTE_ADDR"]
```

Be careful not to log secrets.

---

## Step 3 — Check whether middleware executes

Add instrumentation:

```ruby
Rails.logger.info("Entering MyMiddleware")
```

and:

```ruby
Rails.logger.info("Leaving MyMiddleware")
```

---

## Step 4 — Check short-circuiting

If the request never reaches Rails:

```text
Middleware
   ↓
response
```

find which layer returned early.

---

## Step 5 — Check request body consumption

If Rails suddenly receives an empty body:

```ruby
env["rack.input"]
```

is one of the first things to investigate.

---

# 70. Debugging Middleware Ordering

Suppose:

```text
Middleware A
Middleware B
Rails
```

Add logging:

```ruby
puts "A before"
response = @app.call(env)
puts "A after"
```

and:

```ruby
puts "B before"
response = @app.call(env)
puts "B after"
```

You should see:

```text
A before
B before
Rails
B after
A after
```

This is the fundamental nested-call model.

---

# 71. Exception Propagation

With:

```text
A
 ↓
B
 ↓
Rails
```

if Rails raises:

```text
Rails
  ↑
  exception
B
  ↑
  exception
A
  ↑
  exception
```

A higher middleware can rescue it:

```ruby
begin
  @app.call(env)
rescue => e
  ...
end
```

This explains why exception middleware placement matters.

---

# 72. A Useful Debugging Experiment

Create:

```ruby
class TraceMiddleware
  def initialize(name, app)
    @name = name
    @app = app
  end

  def call(env)
    puts "#{@name}: before"

    response = @app.call(env)

    puts "#{@name}: after"

    response
  end
end
```

Then:

```ruby
app = TraceMiddleware.new(
  "A",
  TraceMiddleware.new(
    "B",
    MyApp.new
  )
)
```

Calling:

```ruby
app.call(env)
```

produces:

```text
A: before
B: before
B: after
A: after
```

This tiny experiment is one of the best ways to internalize Rack composition.

---

# 73. Best Practices

1. Keep Rack applications small and composable.
2. Keep request state local.
3. Avoid mutable global/application-instance state.
4. Understand concurrency.
5. Treat incoming headers as untrusted.
6. Be careful when reading `rack.input`.
7. Preserve response semantics.
8. Keep middleware responsibilities narrow.
9. Measure middleware overhead.
10. Understand middleware ordering.
11. Avoid unnecessary per-request allocations.
12. Avoid unnecessary network calls.
13. Instrument important boundaries.
14. Use Rails abstractions where appropriate instead of manually reproducing them.
15. Know how to inspect the actual middleware stack.

---

# 74. Anti-Patterns

## Giant Rack application

Bad:

```ruby
class App
  def call(env)
    # authentication
    # authorization
    # routing
    # database
    # serialization
    # logging
    # metrics
    # business logic
    # ...
  end
end
```

Better:

```text
Rack boundary
   ↓
middleware
   ↓
router
   ↓
application layer
```

---

## Shared mutable request state

Bad:

```ruby
@app.current_user = user
```

Better:

```ruby
current_user = user
```

or appropriate request-scoped mechanisms.

---

## Reading the request body unnecessarily

Bad:

```ruby
body = env["rack.input"].read
```

just to log it.

Request bodies may contain:

* credentials
* personal data
* huge payloads
* secrets

Logging them can create both performance and security problems.

---

## Middleware that does too much

A middleware should have a focused responsibility.

Bad:

```text
Authentication + database writes + billing + serialization
```

in one middleware.

---

# 75. Comparison Table

| Concept           | Purpose                        | Interface           | Typical Layer  |
| ----------------- | ------------------------------ | ------------------- | -------------- |
| HTTP              | Network protocol               | Request/response    | Network        |
| Web server        | Accept HTTP traffic            | Server-specific     | Infrastructure |
| Rack              | Ruby web interface             | `call(env)`         | Boundary       |
| Rack Application  | Process request                | `call(env)`         | Application    |
| Rack Middleware   | Wrap another application       | `call(env)`         | Pipeline       |
| Rails Application | Full Rails web application     | Rack-compatible     | Framework      |
| Rails Router      | Map request to endpoint        | Rails routing API   | Framework      |
| Controller        | Handle application endpoint    | Action methods      | Application    |
| Service Object    | Encapsulate business operation | Ruby API            | Application    |
| Rack Environment  | Request context                | Hash-like structure | Boundary       |

---

# 76. Rack Application vs Middleware vs Controller

This distinction frequently appears in senior interviews.

| Property                | Rack Application | Middleware      | Rails Controller         |
| ----------------------- | ---------------- | --------------- | ------------------------ |
| Receives `env` directly | Yes              | Yes             | Usually indirectly       |
| Implements `call`       | Yes              | Yes             | No                       |
| Can short-circuit       | Yes              | Yes             | Through Rails mechanisms |
| Wraps another app       | Not necessarily  | Usually         | No                       |
| Operates before routing | Can              | Usually         | No                       |
| HTTP-level abstraction  | Very low         | Low             | Higher                   |
| Rails-specific          | No               | Not necessarily | Yes                      |

---

# 77. Interview Questions — Basic

### Q1

What is a Rack application?

**Expected answer:**

A Ruby object responding to `call(env)` and returning a Rack-compatible response consisting of status, headers, and body.

---

### Q2

What is the signature of a Rack application?

```ruby
call(env)
```

---

### Q3

What does a Rack application return?

```ruby
[
  status,
  headers,
  body
]
```

---

### Q4

Is Rack a web server?

No.

Rack provides the application/server interface and ecosystem; the web server accepts network connections and invokes the Rack application.

---

### Q5

Is Rails a Rack application?

Yes, `Rails.application` is the primary Rack application object for a Rails application.

---

# 78. Interview Questions — Intermediate

### Q6

Why is the Rack body enumerable?

Discuss:

* incremental response generation
* streaming
* memory efficiency
* server iteration

---

### Q7

Why does Rack use an environment hash?

Discuss:

* server/framework decoupling
* standardized keys
* extensibility
* HTTP metadata

---

### Q8

What is `rack.input`?

The request-body input stream exposed through the Rack environment.

---

### Q9

What happens if middleware consumes `rack.input`?

Downstream consumers may see an already-consumed stream unless the input is appropriately preserved/rewound.

---

### Q10

Why can a Rack application be a lambda?

Because Rack depends on behavior:

```ruby
call(env)
```

rather than inheritance.

---

# 79. Interview Questions — Senior

### Q11

Why is Rack composition powerful?

Strong answer:

> Because the same `call(env)` interface exists across the request pipeline. Middleware can wrap another Rack application without needing to understand the implementation behind it.

---

### Q12

Why does middleware order matter?

Discuss:

* authentication
* exception handling
* logging
* request mutation
* response mutation
* short-circuiting
* performance
* security

---

### Q13

Why can mutable instance variables in a Rack application be dangerous?

Because the application object can be long-lived and may serve concurrent requests.

---

### Q14

How would you debug a request that never reaches a Rails controller?

Strong approach:

1. inspect middleware
2. identify short-circuiting middleware
3. log before/after each relevant layer
4. inspect request environment
5. inspect exceptions
6. verify routing
7. verify server configuration

---

# 80. Interview Questions — Staff Level

### Q15

A Rails endpoint suddenly becomes 30 ms slower after a platform team's change. How would Rack help you investigate?

Expected reasoning:

```text
HTTP
 ↓
Load balancer
 ↓
Web server
 ↓
Rack middleware
 ↓
Rails
```

Measure latency at each boundary.

Investigate:

* newly added middleware
* network calls
* logging
* synchronization
* request body parsing
* allocations
* middleware ordering
* external dependencies

---

### Q16

A rate limiter is implemented as Rack middleware. What are the architectural advantages?

Discuss:

* enforcement before controllers
* centralized policy
* framework independence
* short-circuiting
* consistent behavior
* reusable composition

Then discuss drawbacks:

* middleware ordering
* extra latency
* Redis dependency
* distributed consistency
* proxy/IP trust

---

### Q17

Why might an application-level instance-variable cache behave differently after scaling from one process to ten?

Because process memory is not automatically shared.

```text
Process 1 → cache A
Process 2 → cache B
...
Process 10 → cache J
```

For shared state, use an external system.

---

### Q18

How would you design a Rack-level maintenance mode?

Possible design:

```text
Request
   ↓
Maintenance Middleware
   │
   ├── maintenance enabled → 503
   │
   └── disabled → Rails
```

Then discuss:

* health-check bypass
* admin bypass
* caching
* deployment consistency
* response headers
* observability

---

# 81. Practical Coding Examples

## Exercise 1 — Hello Rack

Implement:

```ruby
class HelloApp
  def call(env)
    # ...
  end
end
```

Requirements:

* return HTTP 200
* `Content-Type: text/plain`
* body `"Hello Rack"`

---

## Exercise 2 — Path Router

Support:

```text
GET /
GET /health
GET /about
```

Return:

```text
200 Home
200 OK
200 About
```

Everything else:

```text
404 Not Found
```

---

## Exercise 3 — Method Router

Support:

```text
GET /users
POST /users
DELETE /users
```

Return different status codes.

---

## Exercise 4 — Request Logger

Create a middleware that logs:

```text
METHOD PATH STATUS DURATION
```

---

## Exercise 5 — Authentication

Create middleware:

```text
Authorization: Bearer secret
```

If missing:

```text
401
```

If present:

```text
call downstream app
```

---

## Exercise 6 — Request Body

Build an application that reads:

```ruby
env["rack.input"]
```

and echoes the body.

Then investigate what happens if another middleware reads the body first.

---

# 82. Edge Cases

## Empty response body

Example:

```ruby
[
  204,
  {},
  []
]
```

Understand the semantics of status codes that should not contain response content.

---

## HEAD requests

A `HEAD` request has different response-body semantics from `GET`.

This is one reason middleware/server behavior around response bodies matters.

---

## Client disconnect

What happens if:

```text
server starts streaming
       ↓
client disconnects
```

Your application may encounter errors or termination behavior while producing the body.

---

## Huge request body

A malicious client may send a very large body.

Consider:

* memory
* parsing cost
* request limits
* timeout
* downstream storage

---

## Huge response

A huge response raises:

* memory
* streaming
* bandwidth
* worker occupancy
* timeout

questions.

---

## Concurrent requests

The same application object may process multiple requests concurrently depending on server configuration.

Never assume:

```text
one application object = one request
```

---

# 83. Real Production Architecture

A realistic Rails production system may look like:

```text
                    Internet
                       │
                       ▼
                 Load Balancer
                       │
              ┌────────┴────────┐
              ▼                 ▼
           Puma #1           Puma #2
              │                 │
              ▼                 ▼
       Rails Application  Rails Application
              │                 │
              ▼                 ▼
       Rack Middleware    Rack Middleware
              │                 │
              └────────┬────────┘
                       ▼
                    Router
                       │
                       ▼
                   Controller
                       │
                       ▼
                    Service
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
        PostgreSQL             Redis
```

Rack is the abstraction that helps make the web-server/application boundary standardized.

---

# 84. A More Accurate Rails Request Model

For interview purposes, use:

```text
Client
  │
  ▼
Load Balancer / Proxy
  │
  ▼
Puma
  │
  ▼
Rack env
  │
  ▼
Rails.application
  │
  ▼
Rails middleware stack
  │
  ▼
Routing
  │
  ▼
Controller
  │
  ▼
Application logic
  │
  ▼
Database / Redis / APIs
  │
  ▼
Response
  │
  ▼
Middleware unwinds
  │
  ▼
Puma
  │
  ▼
Client
```

That is much closer to how a senior engineer should reason about the request path.

---

# 85. The Most Important Insight

The deepest idea in Rack is not:

```ruby
def call(env)
```

The deeper idea is:

> **A standardized callable boundary makes independently developed components composable.**

This gives us:

```text
Server
   ↓
Rack Application
   ↓
Middleware
   ↓
Middleware
   ↓
Framework
   ↓
Application
```

without requiring each component to know the internal implementation of the next component.

That is the architectural reason Rack became so useful.

---

# 86. Related Topics

According to the curriculum, after:

```text
Rack
Rack Environment
Rack Application
```

the next topics are:

```text
Rack Middleware
Request Lifecycle
Rails Boot Process
Zeitwerk Autoloading
Initializers
Configuration
Railties
Engines
```

This ordering is important because each topic builds on the previous one.

Recommended immediate sequence:

```text
                    Rack
                     │
                     ▼
              Rack Environment
                     │
                     ▼
              Rack Application
                     │
                     ▼
              Rack Middleware
                     │
                     ▼
             Request Lifecycle
                     │
                     ▼
              Rails Boot Process
```

---

# 87. Summary

A Rack application:

```ruby
app.call(env)
```

receives:

```ruby
env
```

and returns:

```ruby
[
  status,
  headers,
  body
]
```

The core architecture is:

```text
Web Server
    ↓
Rack env
    ↓
Rack Application
    ↓
[status, headers, body]
    ↓
Web Server
```

Rails uses the same boundary:

```ruby
Rails.application
```

is the primary Rack application object.

Rails then builds a much larger pipeline around that interface:

```text
Rack
 ↓
Middleware
 ↓
Rails
 ↓
Router
 ↓
Controller
 ↓
Application
```

The most important senior-level concepts are:

1. Rack application = `call(env)`.
2. Rack is not the web server.
3. Rails is built on the Rack abstraction.
4. `env` is the request boundary.
5. Response = status + headers + body.
6. Body is iterable.
7. Rack applications are composable.
8. Middleware is itself a Rack application wrapper.
9. Application objects can be long-lived.
10. Shared mutable state can create concurrency bugs.
11. Middleware order changes behavior.
12. `rack.input` is a stream and must be handled carefully.
13. Rack is close to the HTTP/security boundary.
14. Middleware can short-circuit requests.
15. Every request-layer component has performance cost.

---

# 88. One-Page Cheat Sheet

```text
┌──────────────────────────────────────────────────────────┐
│                    RACK APPLICATION                      │
├──────────────────────────────────────────────────────────┤
│ Definition                                               │
│                                                          │
│ Ruby object responding to:                               │
│                                                          │
│     call(env)                                            │
│                                                          │
│ Returns:                                                 │
│                                                          │
│     [status, headers, body]                              │
├──────────────────────────────────────────────────────────┤
│ Request                                                  │
│                                                          │
│ HTTP → Web Server → Rack env → app.call(env)             │
├──────────────────────────────────────────────────────────┤
│ Environment                                              │
│                                                          │
│ REQUEST_METHOD                                           │
│ PATH_INFO                                                │
│ QUERY_STRING                                             │
│ HTTP_*                                                   │
│ rack.input                                               │
│ rack.errors                                              │
│ rack.url_scheme                                          │
├──────────────────────────────────────────────────────────┤
│ Response                                                 │
│                                                          │
│ status  → 200, 404, 500, ...                            │
│ headers → Content-Type, Cache-Control, ...              │
│ body    → enumerable response body                      │
├──────────────────────────────────────────────────────────┤
│ Rails                                                    │
│                                                          │
│ Rails.application                                        │
│        ↓                                                 │
│ Rails middleware                                         │
│        ↓                                                 │
│ Router                                                   │
│        ↓                                                 │
│ Controller                                               │
├──────────────────────────────────────────────────────────┤
│ Composition                                              │
│                                                          │
│ Middleware A                                             │
│      ↓                                                   │
│ Middleware B                                             │
│      ↓                                                   │
│ Rails                                                    │
│                                                          │
│ All communicate through call(env).                      │
├──────────────────────────────────────────────────────────┤
│ Important Risks                                          │
│                                                          │
│ • shared mutable state                                   │
│ • consuming rack.input                                  │
│ • incorrect middleware order                            │
│ • trusting proxy headers                                 │
│ • excessive middleware                                   │
│ • expensive work per request                             │
└──────────────────────────────────────────────────────────┘
```

---

# 89. Practice Exercises — Easy → Hard

## Easy

### 1. Hello World

Implement:

```ruby
GET /
```

returning:

```text
Hello Rack
```

---

### 2. Status Codes

Create endpoints:

```text
/ok       → 200
/created  → 201
/notfound → 404
```

---

### 3. Inspect `env`

Print:

```ruby
REQUEST_METHOD
PATH_INFO
QUERY_STRING
HTTP_HOST
```

---

# Medium

## 4. Tiny Router

Implement:

```text
GET /
GET /health
GET /users
POST /users
```

without Rails.

---

## 5. Logging Middleware

Record:

```text
timestamp
method
path
status
duration
```

---

## 6. Authentication Middleware

Require:

```http
Authorization: Bearer secret
```

Return:

```text
401
```

otherwise.

---

## 7. Response Header Middleware

Add:

```http
X-Application: MyApp
```

to every response.

---

# Hard

## 8. Rate Limiting Middleware

Design:

```text
100 requests / minute / client
```

Questions:

* Where is state stored?
* How do multiple processes share state?
* What happens when Redis is unavailable?
* What happens with multiple application servers?
* How do you identify clients behind proxies?

---

## 9. Streaming Application

Return 1,000 chunks without constructing one giant response string.

Analyze:

* memory
* latency
* connection lifetime
* client disconnects
* worker/thread occupancy

---

## 10. Production Request Tracer

Build a Rack middleware that measures:

```text
request ID
method
path
status
duration
```

Then add:

```text
database duration
external API duration
```

Finally explain where Rack ends and Rails begins.

---

# 90. Senior Interview Challenge

You should be able to answer this without looking at the guide:

> **"Explain exactly what happens when a request reaches a Rails application, starting from the web server and ending with the HTTP response. Where does Rack Application fit?"**

A strong answer should include:

```text
HTTP request
    ↓
Web server
    ↓
Rack environment
    ↓
Rails.application
    ↓
Rack/Rails middleware stack
    ↓
routing
    ↓
controller
    ↓
application logic
    ↓
response
    ↓
middleware unwinding
    ↓
web server
    ↓
HTTP response
```

If you can explain that clearly, you understand the architectural role of Rack Application.

---

# 91. Additional Resources

## Official Rails documentation

The Rails guide on Rack integration explains:

* `Rails.application`
* `config.ru`
* `rackup`
* Rails middleware
* middleware inspection
* middleware configuration

[Rails on Rack — Ruby on Rails Guides](https://guides.rubyonrails.org/v6.1/rails_on_rack.html?utm_source=chatgpt.com)

The Rails configuration guide covers middleware configuration and security-related middleware such as Host Authorization.

[Configuring Rails Applications — Rails Guides](https://guides.rubyonrails.org/configuring.html?utm_source=chatgpt.com)

## Source-code areas worth studying

After understanding the concepts above, inspect these Rails areas in your local Rails source:

```text
actionpack/
  lib/action_dispatch/

railties/
  lib/rails/application.rb

rails/
  application lifecycle / initialization code
```

Particularly useful concepts to trace:

```text
Rails.application
ActionDispatch::MiddlewareStack
build_middleware_stack
Rails::Application
ActionDispatch routing
```

Do not try to read all of Rails at once.

Trace one request path.

---

# 92. Final Mental Model

If you remember only one diagram, remember this:

```text
                         INTERNET
                            │
                            ▼
                     HTTP REQUEST
                            │
                            ▼
                       WEB SERVER
                       Puma/etc.
                            │
                            ▼
                     Rack Environment
                            │
                            ▼
                    Rails.application
                            │
                            ▼
                  ┌──────────────────┐
                  │ Rack Middleware  │
                  │                  │
                  │ Auth             │
                  │ Logging          │
                  │ Security         │
                  │ Sessions         │
                  │ Exceptions       │
                  │ Metrics          │
                  └────────┬─────────┘
                           │
                           ▼
                        ROUTER
                           │
                           ▼
                      CONTROLLER
                           │
                           ▼
                    APPLICATION CODE
                           │
                    ┌──────┴──────┐
                    ▼             ▼
                 PostgreSQL     Redis
                    │             │
                    └──────┬──────┘
                           ▼
                        RESPONSE
                           │
                           ▼
                    Middleware unwind
                           │
                           ▼
                       Web Server
                           │
                           ▼
                      HTTP RESPONSE
```

The central abstraction is:

```ruby
app.call(env)
```

The central architectural idea is:

```text
Everything at the Rack boundary is composable
because everything speaks the same interface.
```

That is the foundation you need before moving into **Rack Middleware**.
