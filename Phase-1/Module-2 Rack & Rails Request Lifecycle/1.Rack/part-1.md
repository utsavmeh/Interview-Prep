Perfect. According to your curriculum, **Rack** is the first topic of **Module 2 – Rack & Rails Request Lifecycle**, after completing Internet & HTTP fundamentals. 

Rack is one of the most important topics for senior Rails interviews because it explains **how Rails actually communicates with web servers**. Many Rails developers use Rack every day without realizing it. Companies like Shopify, GitLab, Basecamp, and Stripe expect senior engineers to understand Rack beyond simply saying "it's an interface between Ruby web servers and frameworks."

This study guide is too large for a single response (it would exceed the maximum response size by several times), so I'll deliver it as a **book** in multiple parts. We'll cover every section you requested without skipping details.

# Rack Master Study Guide

## Planned Parts

### Part 1

* Overview
* History
* Why Rack Exists
* Problems Rack Solves
* Rack Architecture
* Rack Specification
* Rack Application
* Rack Environment
* Hello World Rack App
* Request Flow

### Part 2

* Rack Request
* Rack Response
* Rack::Builder
* Rackup
* Middleware Basics
* Middleware Chain
* Request Lifecycle

### Part 3

* Rails + Rack
* ActionDispatch
* Rails Middleware Stack
* Middleware Ordering
* Custom Middleware
* Production Examples

### Part 4

* Internal Implementation
* Rack Source Code Walkthrough
* Thread Safety
* Concurrency
* Streaming
* Hijacking
* Async
* HTTP/2 Considerations

### Part 5

* Performance
* Security
* Debugging
* Logging
* Common Mistakes
* Best Practices
* Anti-patterns

### Part 6

* Advanced Examples
* Edge Cases
* Comparison Tables
* Interview Questions
* Coding Exercises
* Cheat Sheet
* Summary
* Resources

---

# Part 1 — Foundations of Rack

# 1. Overview

## What is Rack?

Rack is a **standard interface (contract)** between:

* Ruby web servers
* Ruby web frameworks
* Ruby web applications
* Middleware

Instead of every server knowing how to run every framework, everyone agrees to speak **Rack**.

Think of Rack as:

> "The HTTP language that every Ruby web component understands."

---

Without Rack:

```
Puma
 ├── Rails integration
 ├── Sinatra integration
 ├── Hanami integration
 ├── Grape integration
 ├── Roda integration
 └── Every future framework...
```

Every server would need special code for every framework.

That becomes impossible.

---

With Rack:

```
             Rack

Puma  <------------> Rails
Falcon<------------> Sinatra
Unicorn<-----------> Hanami
Passenger<---------> Grape
```

Everyone only needs to know Rack.

---

## Why does Rack exist?

Imagine the year before Rack.

Suppose there are:

```
3 servers

Puma
Unicorn
Passenger
```

and

```
3 frameworks

Rails
Sinatra
Camping
```

Without Rack:

Each server needs adapters for every framework.

```
Puma → Rails
Puma → Sinatra
Puma → Camping

Unicorn → Rails
Unicorn → Sinatra
Unicorn → Camping

Passenger → Rails
Passenger → Sinatra
Passenger → Camping
```

Connections needed:

```
Servers × Frameworks

3 × 3 = 9 integrations
```

If another framework appears:

```
Hanami
```

Now:

```
4 frameworks

3 × 4 = 12 integrations
```

This grows as **O(n × m)**.

Rack reduces it to:

```
Every server → Rack
Every framework → Rack
```

Only:

```
Servers + Frameworks
```

connections.

---

## Why is this important?

Because now:

Changing servers doesn't require changing Rails.

Example:

```
Puma

↓

Falcon

↓

Passenger

↓

Unicorn
```

Rails code remains unchanged.

Similarly,

A server can run:

* Rails
* Sinatra
* Grape
* Roda
* Hanami

without knowing their internal implementations.

---

## Real analogy

Think about USB.

Before USB:

Every printer had its own cable.

Every keyboard had another cable.

Every mouse had another cable.

Every manufacturer had different connectors.

---

USB standardized communication.

Now:

```
Keyboard

↓

USB

↓

Computer
```

```
Mouse

↓

USB

↓

Computer
```

```
Printer

↓

USB

↓

Computer
```

Rack is the USB of Ruby web development.

---

## What Rack is NOT

A very common interview trap.

Many developers think Rack is:

> "A web server."

Wrong.

Rack is **not**:

* a server
* a framework
* Rails
* Puma
* Unicorn

Rack is only the **interface**.

---

### Another misconception

People say:

> "Rack receives requests."

Not exactly.

Actually:

```
Browser

↓

Puma

↓

Rack interface

↓

Rails
```

Rack itself doesn't listen on ports.

Servers do.

---

### Another misconception

People think Rack belongs to Rails.

It doesn't.

Rack is an independent Ruby project.

Rails depends on Rack.

Not the other way around.

---

## When should you use Rack?

You use Rack every day—even if you never write Rack code.

Examples:

Rails:

```ruby
class ApplicationController < ActionController::Base
end
```

Behind the scenes:

```
Controller

↓

ActionDispatch

↓

Rack

↓

Puma
```

---

Sinatra:

```ruby
get "/users" do
end
```

Also Rack.

---

Grape APIs?

Rack.

---

Roda?

Rack.

---

Hanami?

Rack.

---

Even if you never require `"rack"` yourself…

You're already using it.

---

# Why senior engineers care about Rack

Understanding Rack helps you reason about:

* why middleware order matters
* how authentication works
* how logging works
* how cookies work
* how sessions work
* why Rails middleware behaves the way it does
* why request IDs are added
* how rate limiting is implemented
* how CORS works
* how request compression works
* why Rack::Attack works
* why Rack::Cache works
* why ActionDispatch exists

Many production concerns are solved **before your controller runs**, in the Rack layer.

---

# Where Rack fits in the request flow

At a high level:

```text
Client (Browser / Mobile App)
          │
          ▼
   Web Server (Puma)
          │
          ▼
   Rack Interface
          │
          ▼
 Rack Middleware Stack
          │
          ▼
 Rails (ActionDispatch)
          │
          ▼
      Router
          │
          ▼
    Controller
          │
          ▼
      Model / DB
          │
          ▼
      Response
          ▲
          │
 Controller ← Router ← Rails ← Middleware ← Rack ← Puma
          │
          ▼
        Client
```

This is the mental model you should keep throughout the rest of the study guide.

---

# Rack in One Sentence

If you remember only one thing from this part:

> **Rack is the standardized interface that allows Ruby web servers, middleware, and frameworks to communicate using a common request/response contract.**

---

In **Part 2**, we'll go much deeper into the core mechanics of Rack, including:

* The Rack specification (`call` and the `[status, headers, body]` response)
* The Rack environment (`env`) hash in detail
* Building a Rack application from scratch
* The complete request lifecycle through Rack
* `Rack::Request`, `Rack::Response`, `Rack::Builder`, and `config.ru`
* Internal request flow before Rails ever sees the request


# What is unicorn, passenger, Sinatra, Camping, Hanami

Great question. Before diving deeper into Rack, you should know what these are because Rack sits **between web servers and web frameworks**.

Let's classify them first.

| Name          | Type               | Purpose                                                          |
| ------------- | ------------------ | ---------------------------------------------------------------- |
| **Puma**      | Web Server         | Runs your Ruby application                                       |
| **Unicorn**   | Web Server         | Runs Ruby apps using multiple processes                          |
| **Passenger** | Application Server | Runs Ruby apps and integrates with web servers like Apache/Nginx |
| **Falcon**    | Web Server         | Modern async Rack server                                         |
| **Rails**     | Web Framework      | Full-stack web framework                                         |
| **Sinatra**   | Web Framework      | Lightweight web framework                                        |
| **Hanami**    | Web Framework      | Modular Ruby web framework                                       |
| **Camping**   | Web Framework      | Extremely small Ruby web framework                               |
| **Roda**      | Web Framework      | Fast routing-focused web framework                               |
| **Grape**     | API Framework      | Framework for building REST APIs                                 |

---

# 1. Unicorn

## What is it?

**Unicorn** is a **Rack-compatible web server**.

Its job is simple:

* Listen for HTTP requests
* Pass them to a Rack application
* Return the response

Example:

```
Browser
     │
     ▼
 Unicorn
     │
     ▼
 Rack
     │
     ▼
 Rails
```

### How does it work?

Unicorn uses a **master-worker process model**.

```
            Master
          /    |    \
         /     |     \
 Worker  Worker Worker
```

Each worker is a completely separate Ruby process.

### Why?

Ruby (MRI/CRuby) has the **Global VM Lock (GVL)**, which limits true parallel execution of Ruby code within a process. Unicorn avoids this by using multiple OS processes rather than multiple threads.

If one worker crashes:

```
Worker 1 ❌

Master
  │
Creates new Worker
```

The application keeps running.

### Downsides

* Each worker consumes significant memory.
* Does not support persistent connections well.
* Not ideal for WebSockets.
* Less efficient than modern threaded servers for many workloads.

Today, **Puma** has largely replaced Unicorn as the default Rails server.

---

# 2. Passenger

Passenger is also a server for Rack applications, but it is often described as an **application server** because it integrates closely with existing web servers.

Instead of:

```
Puma
```

You can have:

```
Nginx
   │
Passenger
   │
Rails
```

or

```
Apache
   │
Passenger
   │
Rails
```

Passenger manages:

* Starting your application
* Worker processes
* Restarts
* Load balancing
* Monitoring

Historically, it made deploying Rails applications much simpler.

---

# 3. Sinatra

Sinatra is **not** a server.

It is a **Ruby web framework**.

Think of it as a very lightweight alternative to Rails.

Rails:

```ruby
class UsersController < ApplicationController
  def index
  end
end
```

Sinatra:

```ruby
require "sinatra"

get "/users" do
  "Hello"
end
```

That's enough to create a web application.

Internally:

```
Browser
    │
Puma
    │
Rack
    │
Sinatra
```

Sinatra implements the Rack interface, just like Rails does.

### When do people use Sinatra?

* Small APIs
* Microservices
* Internal tools
* Prototypes
* Learning Rack

It intentionally provides fewer features than Rails.

---

# 4. Camping

Camping is a very small Ruby web framework created by _why the lucky stiff.

Its goal was to fit an entire web application into a tiny amount of code.

A Camping app might look like:

```ruby
Camping.goes :Blog

module Blog::Controllers
  class Index < R '/'
    def get
      render :index
    end
  end
end
```

Today, Camping is mostly of historical interest.

It helped inspire ideas about minimal Ruby web frameworks.

---

# 5. Hanami

Hanami is a modern Ruby web framework designed with modularity and explicit architecture in mind.

Rails emphasizes convention and an "everything included" approach.

Hanami emphasizes:

* Small components
* Explicit dependencies
* Better separation of concerns
* Fast boot times
* Testability

A simplified Hanami route might look like:

```ruby
get "/books", to: "books.index"
```

Internally:

```
Browser
    │
Puma
    │
Rack
    │
Hanami
```

Like Rails and Sinatra, Hanami is a Rack application.

---

# The Big Picture

Here's how they relate:

```
                Browser
                   │
             HTTP Request
                   │
        ┌──────────┴──────────┐
        │                     │
     Puma               Unicorn
        │               Passenger
        │                 Falcon
        └──────────┬──────────┘
                   │
                 Rack
                   │
     ┌─────────────┼─────────────┐
     │             │             │
   Rails        Sinatra       Hanami
     │             │             │
     └─────────────┴─────────────┘
          Your Application Code
```

The key takeaway is:

* **Puma, Unicorn, Passenger, and Falcon** are responsible for **running** your application and speaking HTTP.
* **Rack** defines the common interface between servers and applications.
* **Rails, Sinatra, Hanami, Roda, and Grape** implement that interface so they can run on any Rack-compatible server.

This separation is exactly why you can switch from Unicorn to Puma without changing your Rails application code.


Excellent. Now that we've established **why Rack exists**, it's time to learn **how Rack actually works**.

This is the point where many Rails developers start getting confused. Don't worry—we'll build the understanding step by step.

---

# Part 2 — The Rack Specification

Everything in Rack revolves around **one incredibly simple interface**.

If you understand this interface, you'll understand 80% of Rack.

---

# The Golden Rule of Rack

A Rack application is **any Ruby object** that:

1. Responds to `call`
2. Accepts one argument (`env`)
3. Returns an array with exactly three elements

```ruby
call(env)
```

returns

```ruby
[status, headers, body]
```

That's it.

That's the entire Rack specification at its core.

Every Rack-compatible framework—including Rails—implements this contract.

---

# Why is the Interface So Simple?

Imagine you're writing a web server like Puma.

Puma shouldn't need to know:

* What Rails is
* What Sinatra is
* What Hanami is
* How controllers work
* How routing works
* How ActiveRecord works

Puma only needs one promise:

> "If I call `call(env)`, you'll give me an HTTP response."

So Puma can do this:

```ruby
response = app.call(env)
```

without caring what `app` actually is.

That `app` could be:

* Rails
* Sinatra
* Hanami
* Your own application
* A middleware

Everything looks identical to Puma.

---

# The Rack Contract

Every Rack application follows:

```text
Request
   │
   ▼
call(env)
   │
   ▼
[status, headers, body]
```

Think of it like a function:

```
Input
↓

Process

↓

Output
```

where

```
Input  = HTTP request

Output = HTTP response
```

---

# Smallest Rack Application

Let's build the smallest possible Rack app.

```ruby
class HelloWorld
  def call(env)
    [
      200,
      { "Content-Type" => "text/plain" },
      ["Hello World"]
    ]
  end
end
```

That's it.

Believe it or not…

This is a complete web application.

No Rails.

No controllers.

No routes.

No models.

Just Rack.

---

Let's examine every line.

---

# Step 1

```ruby
class HelloWorld
```

Rack doesn't require a class.

This also works:

```ruby
app = Proc.new do |env|
end
```

or

```ruby
app = lambda do |env|
end
```

or any object that responds to `call`.

Rack doesn't care.

---

# Step 2

```ruby
def call(env)
```

This is mandatory.

Rack always calls:

```ruby
app.call(env)
```

Never

```ruby
execute
```

Never

```ruby
run
```

Never

```ruby
perform
```

Only

```ruby
call
```

---

# Step 3

The parameter

```ruby
env
```

is the HTTP request.

Not the body.

Not the headers.

The **entire request**.

Everything arrives inside this hash.

For example:

```ruby
env = {
  "REQUEST_METHOD" => "GET",
  "PATH_INFO" => "/users",
  "QUERY_STRING" => "page=2",
  "REMOTE_ADDR" => "127.0.0.1"
}
```

We'll study `env` in depth shortly.

---

# Step 4

Return value:

```ruby
[
  200,
  { "Content-Type" => "text/plain" },
  ["Hello World"]
]
```

This is the HTTP response.

It has exactly three parts:

```
Status

Headers

Body
```

Let's understand each.

---

# Status

```ruby
200
```

This is simply the HTTP status code.

Examples:

```ruby
200
```

OK

```ruby
201
```

Created

```ruby
204
```

No Content

```ruby
301
```

Moved Permanently

```ruby
302
```

Redirect

```ruby
400
```

Bad Request

```ruby
401
```

Unauthorized

```ruby
403
```

Forbidden

```ruby
404
```

Not Found

```ruby
422
```

Unprocessable Entity

```ruby
500
```

Internal Server Error

Exactly the same status codes you've already studied in HTTP.

Rack doesn't invent new ones.

---

# Headers

Next:

```ruby
{
  "Content-Type" => "text/plain"
}
```

These become HTTP headers.

Browser receives

```http
HTTP/1.1 200 OK
Content-Type: text/plain
```

You can add anything.

```ruby
{
  "Content-Type" => "application/json",
  "Cache-Control" => "no-cache",
  "X-Version" => "1.2.0"
}
```

Rack simply forwards them.

---

# Body

This surprises almost everyone.

The body is **not**:

```ruby
"Hello World"
```

Instead it is:

```ruby
["Hello World"]
```

Why?

Why an array?

This is one of the smartest parts of Rack.

---

# Why Isn't the Body a String?

Suppose you're returning:

```
5 GB video
```

Would you do:

```ruby
body = File.read(...)
```

No.

That would load the entire video into memory.

Instead Rack says:

> "Give me something I can iterate over."

An Array is iterable.

```ruby
["Hello"]
```

works because Puma can do:

```ruby
body.each do |chunk|
  socket.write(chunk)
end
```

Notice something important.

Puma doesn't know:

* if it's one chunk
* ten chunks
* one million chunks

It simply asks:

```ruby
each
```

---

# The Real Requirement

Rack doesn't actually require an Array.

It requires an object that responds to:

```ruby
each
```

This means all of these are valid:

```ruby
["Hello"]
```

```ruby
FileBody.new
```

```ruby
Enumerator.new
```

```ruby
CustomStream.new
```

As long as it implements:

```ruby
def each
```

Rack is happy.

---

# Example

Imagine:

```ruby
class MyBody
  def each
    yield "Hello "
    yield "World"
  end
end
```

Rack response:

```ruby
[
  200,
  {},
  MyBody.new
]
```

Puma executes:

```ruby
body.each do |chunk|
  socket.write(chunk)
end
```

Output becomes:

```
Hello World
```

without loading everything into one huge string.

This design enables **streaming**, which we'll explore in a later part.

---

# Why Return an Array Instead of a Response Object?

A natural question is:

> "Why didn't Rack define a `Response` class?"

For example:

```ruby
Response.new(
  status: 200,
  headers: {},
  body: "Hello"
)
```

It certainly could have.

Instead, Rack chose a plain Ruby array because:

* It's lightweight.
* It has no dependencies.
* It's easy to construct.
* It's extremely fast.
* Any Ruby object can return it.

Frameworks like Rails wrap this with richer abstractions (such as `ActionDispatch::Response`), but at the Rack boundary they are converted back into the required three-part response.

---

# End-to-End Flow

Let's connect everything together.

Suppose a browser requests:

```http
GET /hello HTTP/1.1
Host: example.com
```

The flow is:

```text
Browser
   │
   ▼
Puma receives the HTTP request
   │
   ▼
Puma builds the `env` hash
   │
   ▼
Puma calls:
app.call(env)
   │
   ▼
Your Rack application returns:
[
  200,
  { "Content-Type" => "text/plain" },
  ["Hello World"]
]
   │
   ▼
Puma converts the response into HTTP
   │
   ▼
Browser receives:
HTTP/1.1 200 OK
Content-Type: text/plain

Hello World
```

Notice the separation of responsibilities:

* **Puma** understands sockets and the HTTP protocol.
* **Rack** defines the interface between the server and the application.
* **Your application** only needs to understand the `env` hash and produce `[status, headers, body]`.

---

# Key Takeaways

By the end of this section, you should remember these five facts:

1. A Rack application is **any object that responds to `call`**.
2. `call` accepts a single argument: **`env`**, the request environment.
3. `call` returns exactly **`[status, headers, body]`**.
4. The **body must be an iterable object** (it responds to `each`), not just a string.
5. This minimal contract is what allows **Puma, Unicorn, Falcon, Rails, Sinatra, Hanami, and middleware** to work together seamlessly.

---

In **Part 3**, we'll dive into the **`env` hash**—arguably the most important object in Rack. We'll examine every major key (`REQUEST_METHOD`, `PATH_INFO`, `QUERY_STRING`, `rack.input`, `rack.errors`, `rack.url_scheme`, and many more), how Puma constructs it from an HTTP request, and how Rails transforms it into the `ActionDispatch::Request` object you interact with in controllers.


Perfect. That's exactly the workflow I recommend. We'll treat each part like a chapter in a book so every chat stays focused and doesn't hit the context limit.

Let's first finish **Part 1**.

---

# Part 1 (Final Section): Rack Environment (`env`)

If the Rack specification answers **"How do a server and an application communicate?"**, then the Rack environment answers **"What information is passed from the server to the application?"**

The `env` hash is the single most important object in Rack.

When Puma receives an HTTP request, it converts the request into a Ruby `Hash` called `env` and passes it to your Rack application.

```
Browser
    │
HTTP Request
    │
    ▼
Puma
    │
Converts HTTP request into `env`
    │
    ▼
app.call(env)
```

Your application never reads directly from a socket. It never parses raw HTTP. Puma does all of that and hands your application a structured hash.

---

# What is `env`?

It is a Ruby `Hash`.

Example:

```ruby
{
  "REQUEST_METHOD" => "GET",
  "PATH_INFO"      => "/users",
  "QUERY_STRING"   => "page=2&sort=name",
  "SERVER_NAME"    => "localhost",
  "SERVER_PORT"    => "3000",
  "rack.version"   => [3, 0],
  "rack.url_scheme"=> "http",
  "rack.input"     => #<IO...>,
  "rack.errors"    => #<IO...>
}
```

Notice the keys are strings, not symbols.

That is intentional and part of the Rack specification.

---

# Where does `env` come from?

Suppose the browser sends:

```http
GET /users?page=2 HTTP/1.1
Host: example.com
User-Agent: Chrome
Accept: application/json
```

Puma roughly performs these steps:

```
Read socket

↓

Parse HTTP

↓

Extract method

↓

Extract path

↓

Extract headers

↓

Extract body

↓

Build env hash

↓

app.call(env)
```

The application never sees the raw HTTP request.

---

# Categories of `env` Keys

The environment contains three categories of information:

```
env
├── CGI variables
├── Rack-specific variables
└── HTTP headers
```

Let's examine each.

---

# 1. CGI Variables

Rack inherited many environment variable names from the CGI specification to maintain compatibility.

Some of the most common keys are:

| Key              | Meaning                                                 |
| ---------------- | ------------------------------------------------------- |
| `REQUEST_METHOD` | GET, POST, PUT, DELETE                                  |
| `PATH_INFO`      | Request path (`/users/10`)                              |
| `QUERY_STRING`   | Everything after `?`                                    |
| `SERVER_NAME`    | Hostname                                                |
| `SERVER_PORT`    | Port number                                             |
| `REMOTE_ADDR`    | Client IP (often the proxy unless configured correctly) |
| `SCRIPT_NAME`    | Mount point of the application                          |

Example:

```
GET /users/15?page=2
```

becomes:

```ruby
env["REQUEST_METHOD"] # "GET"
env["PATH_INFO"]      # "/users/15"
env["QUERY_STRING"]   # "page=2"
```

---

# 2. Rack Variables

These start with `rack.`.

They are reserved by the Rack specification.

## rack.version

Which Rack specification version is being used.

```ruby
env["rack.version"]
```

Example:

```ruby
[3,0]
```

---

## rack.url_scheme

```
http

or

https
```

Rails uses this to determine whether:

```ruby
request.ssl?
```

returns true.

---

## rack.input

This one is extremely important.

It is **not the parsed request body**.

It is an IO-like object containing the raw request body.

Imagine:

```http
POST /users

{
  "name":"Alice"
}
```

Puma does **not** parse the JSON.

Instead:

```ruby
env["rack.input"]
```

contains something like:

```ruby
#<StringIO>
```

Reading it:

```ruby
body = env["rack.input"].read
```

returns:

```json
{"name":"Alice"}
```

Rails later parses that JSON into `params`.

A common interview question is:

> **Does Rack parse JSON request bodies?**

Answer:

> No. Rack exposes the raw body through `rack.input`. Frameworks such as Rails are responsible for parsing it.

---

## rack.errors

An IO stream for writing error messages.

Example:

```ruby
env["rack.errors"].puts("Something went wrong")
```

This is mainly used by middleware and servers.

---

## rack.multithread

Indicates whether the server may process multiple requests concurrently using threads.

```ruby
true
```

on Puma.

Historically:

```ruby
false
```

on Unicorn.

This matters for thread safety.

---

## rack.multiprocess

Indicates whether multiple processes may be handling requests.

Unicorn:

```
true
```

Puma:

Can be true if multiple workers are configured.

---

## rack.run_once

Mostly historical.

Originally intended for CGI environments where the application handled one request and exited.

Usually:

```ruby
false
```

---

# 3. HTTP Headers

This often surprises people.

Suppose the browser sends:

```http
User-Agent: Chrome
```

Rack stores it as:

```ruby
env["HTTP_USER_AGENT"]
```

Notice:

```
User-Agent

↓

Uppercase

↓

Hyphen becomes underscore

↓

HTTP_ prefix
```

Another example:

```
Authorization: Bearer token

↓

HTTP_AUTHORIZATION
```

```
Accept-Language

↓

HTTP_ACCEPT_LANGUAGE
```

```
X-Request-Id

↓

HTTP_X_REQUEST_ID
```

Almost every incoming HTTP header (except a few special cases like `Content-Type` and `Content-Length`) follows this convention.

---

# Why Isn't `env` an Object?

A reasonable question is:

> Why not create a `Request` class?

For example:

```ruby
request.path
request.method
request.headers
```

Rack intentionally chose a simple `Hash` because:

* Any Ruby application can use it.
* No inheritance is required.
* No framework dependency exists.
* It's lightweight.
* Middleware can freely add, remove, or modify keys.

Rails later wraps the hash in `ActionDispatch::Request` to provide convenient methods like `request.path`, `request.method`, and `request.headers`.

---

# How Middleware Uses `env`

One of Rack's greatest strengths is that middleware can inspect or modify the environment before the application sees it.

For example:

```ruby
class RequestTimer
  def initialize(app)
    @app = app
  end

  def call(env)
    env["request.started_at"] = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    @app.call(env)
  end
end
```

The downstream application (or another middleware) can later read:

```ruby
env["request.started_at"]
```

This ability to enrich the request environment is what makes middleware such a powerful abstraction.

---

# How Rails Uses `env`

When Rails receives the request, it does not work directly with the raw hash throughout the framework.

Instead, `ActionDispatch::Request` wraps it:

```ruby
request = ActionDispatch::Request.new(env)
```

Now you can write:

```ruby
request.path
request.request_method
request.ip
request.user_agent
request.headers
```

Internally, these methods still read from the original `env` hash.

For example (simplified):

```ruby
def request_method
  @env["REQUEST_METHOD"]
end
```

The `ActionDispatch::Request` object is essentially a convenient wrapper around the Rack environment.

---

# Debugging the Environment

A simple way to inspect the Rack environment is:

```ruby
class DebugApp
  def call(env)
    pp env

    [
      200,
      { "Content-Type" => "text/plain" },
      ["OK"]
    ]
  end
end
```

Or inside a Rails controller:

```ruby
request.env
```

You can also inspect specific keys:

```ruby
request.env["HTTP_USER_AGENT"]
request.env["PATH_INFO"]
request.env["rack.url_scheme"]
```

This is invaluable when debugging proxy issues, missing headers, authentication, or custom middleware.

---

# Common Interview Questions

### Why does Rack use a Hash for `env`?

Because it is a simple, framework-independent, mutable data structure that any Rack application or middleware can understand and modify.

---

### Does Rack parse JSON request bodies?

No. It provides the raw request body via `rack.input`. Parsing is performed by higher-level frameworks like Rails.

---

### How are HTTP headers represented in `env`?

Most headers are uppercased, prefixed with `HTTP_`, and hyphens are replaced with underscores. For example:

```
Authorization

↓

HTTP_AUTHORIZATION
```

---

### Can middleware modify `env`?

Yes. Middleware commonly reads from, writes to, or augments the environment before passing it further down the chain.

---

# Mental Model

```
Browser
     │
HTTP Request
     │
     ▼
Puma
     │
     ▼
Builds env Hash
     │
     ▼
Middleware 1
     │
     ▼
Middleware 2
     │
     ▼
Rails
     │
ActionDispatch::Request
     │
Controller
```

Every layer is working with the same underlying request information, either directly through the `env` hash or through objects that wrap it.

---

# 🎉 Part 1 Complete

You now have a solid understanding of:

* ✅ What Rack is
* ✅ Why Rack exists
* ✅ The problems it solves
* ✅ Rack architecture
* ✅ The Rack specification (`call(env) -> [status, headers, body]`)
* ✅ Rack applications
* ✅ The Rack environment (`env`)
* ✅ A Hello World Rack application
* ✅ The complete high-level request flow

This foundation is essential because everything else in Rack—middleware, `Rack::Request`, `Rack::Response`, Rails' request lifecycle—builds on these concepts.
