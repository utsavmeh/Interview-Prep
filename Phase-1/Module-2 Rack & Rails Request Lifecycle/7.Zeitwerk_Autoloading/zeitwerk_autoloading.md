# Mastering Zeitwerk Autoloading in Ruby on Rails

## 1. Overview

### What is Zeitwerk?

Zeitwerk is the system Rails uses to **automatically find and load your Ruby classes and modules** when they are needed.

For example, suppose you have this file:

```text
app/models/user.rb
```

and inside it you have:

```ruby
class User
end
```

When your code uses:

```ruby
User.new
```

Rails knows that the `User` class should come from:

```text
app/models/user.rb
```

You don't normally have to write:

```ruby
require "user"
```

Rails handles it for you.

Zeitwerk is:

* Fast
* Thread-safe
* Automatic
* The default autoloader in Rails 6 and newer

It completely replaced Rails' older `classic` autoloader. 

---

### Why does Zeitwerk exist?

Before Zeitwerk, Rails used something called the **classic autoloader**.

The classic autoloader depended heavily on Ruby's:

```ruby
Module#const_missing
```

method.

The basic idea was:

> "If Ruby can't find a class, Rails will notice that and try to figure out which file should contain that class."

For example, if Ruby encountered:

```ruby
User
```

but `User` wasn't loaded yet, Rails would try to guess:

```text
User → user.rb
```

and load that file.

The problem was that Ruby's rules for finding constants can be complicated, especially when **modules and namespaces are involved**.

This could cause:

* Confusing constant-loading behavior
* Problems with nested modules
* Circular dependency problems
* Different behavior depending on where a constant was referenced
* Thread-safety problems

Zeitwerk solves this differently.

Instead of waiting until Ruby gets confused and then using `const_missing`, Zeitwerk **sets up the relationship between files and constants in advance**.

It uses Ruby's built-in:

```ruby
Kernel#autoload
```

mechanism.

So Zeitwerk basically tells Ruby:

> "If someone asks for `User`, load this particular file."

That makes the process much more predictable. 

---

### When do you use Zeitwerk?

If you're using Rails 6 or newer, you are already using Zeitwerk by default.

You generally don't have to manually "turn it on."

Instead, you use it by following Rails' normal naming conventions.

For example:

```text
app/models/user.rb
```

should contain:

```ruby
class User
end
```

Zeitwerk can also be used **outside Rails** as a standalone Ruby code loader. 

---

### Common Misconceptions

#### Misconception 1: "Zeitwerk loads everything when Rails boots."

Not exactly.

In development, Rails generally uses **lazy loading**.

That means a file is loaded **only when the constant inside that file is actually needed**.

For example, if you have:

```text
app/models/user.rb
app/models/order.rb
app/models/product.rb
```

and your application only uses:

```ruby
User
```

Rails doesn't necessarily need to execute all three files immediately.

It can load `user.rb` when `User` is first used.

In production, Rails normally uses **eager loading**, where the application loads its code before it starts handling requests.

This improves performance and thread safety in production. 

---

#### Misconception 2: "Zeitwerk loads my gems too."

No.

Zeitwerk manages the directories that Rails has configured as **autoload paths**.

For example:

```text
app/models
app/services
```

If you install a gem, you generally still use Ruby's normal:

```ruby
require
```

mechanism to load that gem.

So think of it like this:

```text
Your application code
        ↓
     Zeitwerk
        ↓
Automatically loads your classes/modules
```

while gems are generally loaded through:

```ruby
require
```



---

# 2. Core Concepts

Before understanding how Zeitwerk works internally, you need to understand a few basic terms. 

---

### Autoloading

**Autoloading means loading a Ruby file only when the constant defined inside that file is actually needed.**

For example:

```text
app/models/user.rb
```

contains:

```ruby
class User
end
```

You don't need to manually load the file.

When Ruby encounters:

```ruby
User
```

Zeitwerk makes sure the correct file gets loaded.

So:

```text
User is needed
      ↓
Zeitwerk finds user.rb
      ↓
user.rb is loaded
      ↓
User becomes available
```

This is called **lazy loading**.

---

### Eager Loading

Eager loading is basically the opposite of lazy loading.

Instead of waiting until a class is used, Rails loads all the application's relevant files **before the application starts handling requests**.

For example:

```text
user.rb
order.rb
payment.rb
product.rb
```

All of these can be loaded during application startup.

This is especially useful in production because the application knows about all of its classes before multiple threads start processing requests. 

---

### Reloading

Rails automatically handles reloading in development—when you modify a file, Rails unloads the changed constants and reloads the file on the next request. No manual action is needed; just save the file, and reload your browser or API call. This speeds up development by avoiding full server restarts.

Reloading means:

1. Rails removes the constants that were loaded.
2. Rails loads the files again from disk.

This is extremely useful during development.

For example, suppose you have:

```ruby
class User
  def name
    "Utsav"
  end
end
```

You change it to:

```ruby
class User
  def name
    "John"
  end
end
```

You don't want to restart your Rails server every time you change a Ruby file.

Rails can reload the changed code for you.

That's what **reloading** is about. 

---

### Autoload Paths

An **autoload path** is a directory that Zeitwerk is responsible for managing.

For example:

```text
app/models
app/services
```

Rails tells Zeitwerk:

> "These directories contain application code. Please manage the files inside them."

There is an important rule:

> Every Ruby file inside an autoload path must define the constant that matches its file path.

For example:

```text
app/models/user.rb
```

should define:

```ruby
User
```

And:

```text
app/services/payment_processor.rb
```

should define:

```ruby
PaymentProcessor
```

This file-to-constant relationship is one of the most important things to understand about Zeitwerk. 

---

### Root Namespace

Every autoload path has a **root namespace**.

By default, the root namespace is:

```ruby
Object
```

which basically means the **top-level namespace**.

So if you have:

```text
app/models/user.rb
```

Zeitwerk expects:

```ruby
class User
end
```

It does **not** expect:

```ruby
module Models
  class User
  end
end
```

In other words:

```text
app/models/user.rb
        ↓
      User
```

not:

```text
app/models/user.rb
        ↓
Models::User
```

This is because `app/models` is an autoload root, not a namespace called `Models`. 

---

### Inflection

Inflection is the process of converting a file name into a Ruby constant name.

For example:

```text
user.rb
```

becomes:

```ruby
User
```

And:

```text
html_parser.rb
```

normally becomes:

```ruby
HtmlParser
```

Notice something important:

```text
html_parser
    ↓
HtmlParser
```

Zeitwerk doesn't automatically assume that `HTML` should remain uppercase.

If your application wants:

```ruby
HTMLParser
```

you need to configure that explicitly.

We'll see how to do that later. 

---

### Collapsed Directories

Sometimes a directory exists only to **organize files** and does not represent a Ruby namespace.

A common Rails example is:

```text
app/models/concerns
```

You might have:

```text
app/models/concerns/searchable.rb
```

Rails treats `concerns` as a **collapsed directory**.

Therefore, the file should define:

```ruby
module Searchable
end
```

not:

```ruby
module Concerns
  module Searchable
  end
end
```

So:

```text
app/models/concerns/searchable.rb
                ↓
           Searchable
```

not:

```text
app/models/concerns/searchable.rb
                ↓
      Concerns::Searchable
```

Rails configures this behavior for the `concerns` directory. 

---

# 3. How Zeitwerk Works Internally

Understanding the difference between the old `classic` autoloader and Zeitwerk is important if you want to understand **why Zeitwerk was introduced**. 

---

## The Problem With the Classic Autoloader

The old Rails autoloader used:

```ruby
const_missing
```

Let's see what that means.

Imagine your code contains:

```ruby
User
```

but Ruby hasn't loaded the `User` class yet.

The process was roughly:

### Step 1 — Ruby encounters an unknown constant

Ruby sees:

```ruby
User
```

and doesn't know what `User` is.

---

### Step 2 — Ruby calls `const_missing`

Ruby then calls:

```ruby
const_missing(:User)
```

This is Ruby's way of saying:

> "I couldn't find the `User` constant. Maybe someone knows where it is?"

---

### Step 3 — Rails tries to guess the file

Rails would intercept this and think:

```text
User
 ↓
user.rb
```

Then it would try to load:

```text
user.rb
```

---

### Step 4 — The problem

This sounds simple, but Ruby's constant lookup rules are not always simple.

For example, imagine you are inside:

```ruby
Admin::DashboardController
```

and you write:

```ruby
User
```

Ruby has rules for determining **which `User` you're referring to**.

Depending on the namespace and inheritance structure, Ruby might find:

```ruby
Admin::User
```

or:

```ruby
User
```

or fail to find the constant.

The problem was that Rails' `const_missing` approach happened **after Ruby had already performed its own constant lookup**.

That made the behavior difficult to predict in complicated namespace situations.

It could result in:

* Incorrect constant resolution
* Circular dependency errors
* Different behavior depending on where the constant was referenced



---

# Zeitwerk's Approach

Zeitwerk takes a much cleaner approach.

Instead of waiting for Ruby to say:

> "I can't find this constant."

Zeitwerk prepares the mappings **before your application starts executing normal application code**.

It uses Ruby's built-in:

```ruby
Kernel#autoload
```

mechanism. 

Let's go through the process.

---

### Step 1 — Boot / Setup

When Rails starts, it tells Zeitwerk which directories it should manage.

For example:

```text
app/
```

and the directories under it that are configured for autoloading.

So conceptually:

```text
Rails
  ↓
Zeitwerk
  ↓
"Manage these application directories"
```



---

### Step 2 — Zeitwerk scans the directories

Zeitwerk looks through those directories and examines their structure.

For example:

```text
app/models/
├── user.rb
├── order.rb
└── payment.rb
```

Zeitwerk sees:

```text
user.rb
order.rb
payment.rb
```

and determines that they should correspond to:

```text
User
Order
Payment
```



---

### Step 3 — Zeitwerk registers autoloads

For every file, Zeitwerk registers an `autoload`.

For example:

```text
app/models/user.rb
```

gets mapped internally to something conceptually like:

```ruby
autoload :User, "/path/to/app/models/user.rb"
```

This means Ruby now knows:

> "If somebody asks for `User`, load this specific file."

Notice the important difference from the old system.

The mapping is already known:

```text
User → user.rb
```

Ruby doesn't have to wait until something goes wrong and then ask Rails to guess the file. 

---

### Step 4 — Your application uses the constant

Later, your code does:

```ruby
User.new
```

Ruby sees:

```text
User
```

and knows that an autoload has been registered for it.

So Ruby loads:

```text
app/models/user.rb
```

The file defines:

```ruby
class User
end
```

and now the `User` constant exists.

Your code continues running normally.

So the whole process is:

```text
Rails starts
     ↓
Zeitwerk scans files
     ↓
Zeitwerk creates mappings
     ↓
User → user.rb
     ↓
Your code uses User
     ↓
Ruby loads user.rb
     ↓
User class is available
     ↓
Code continues
```



---

## What happens with directories?

Directories can represent Ruby namespaces.

For example:

```text
app/services/
└── billing/
    └── invoice_generator.rb
```

Zeitwerk sees the directory:

```text
billing
```

and understands that it represents:

```ruby
Billing
```

Then:

```text
billing/invoice_generator.rb
```

maps to:

```ruby
Billing::InvoiceGenerator
```

So your code can use:

```ruby
Billing::InvoiceGenerator
```

without manually requiring the file.

Zeitwerk sets up the necessary autoloading structure for the namespace and the files inside it. 

---

# 4. Zeitwerk's Architecture in Rails

Zeitwerk is not some completely separate thing that runs only when you call a class.

It is deeply connected to the **Rails application initialization process**. 

### Rails Initialization

When Rails boots, `Rails::Application::Bootstrap` configures a Zeitwerk loader.

In simple terms:

```text
Rails starts
    ↓
Rails initialization
    ↓
Zeitwerk loader is configured
    ↓
Zeitwerk prepares application code
```



---

### ActionDispatch Middleware

In development, Rails also has a mechanism that works with the request lifecycle.

At the end of a request, Rails can clear/reload the constants managed by Zeitwerk.

This is one of the things that allows you to change your Ruby code and see the changes without restarting the server every time. 

---

### What did Zeitwerk replace?

Zeitwerk replaced the old:

```ruby
ActiveSupport::Dependencies
```

system that was responsible for the classic autoloader.

So, at a high level:

```text
Old Rails
ActiveSupport::Dependencies
        ↓
   classic loader
        ↓
   const_missing


Modern Rails
Zeitwerk
        ↓
   Kernel#autoload
```



---

# 5. Real Production Examples

Now let's look at situations you can actually encounter in a Rails application. 

---

## Custom Inflectors — GraphQL / API

Suppose you have:

```text
app/services/api_client.rb
```

By default, Zeitwerk converts:

```text
api_client
```

into:

```ruby
ApiClient
```

But maybe your application wants:

```ruby
APIClient
```

because `API` is an acronym.

You can tell Zeitwerk about this special naming rule.

For example:

```ruby
# config/initializers/zeitwerk.rb

Rails.autoloaders.main.inflector.inflect(
  "api_client" => "APIClient",
  "html_parser" => "HTMLParser"
)
```

Now Zeitwerk understands:

```text
api_client.rb
      ↓
APIClient
```

instead of:

```text
api_client.rb
      ↓
ApiClient
```

And:

```text
html_parser.rb
      ↓
HTMLParser
```

instead of:

```text
html_parser.rb
      ↓
HtmlParser
```

This is useful when your application uses acronyms such as:

* API
* HTML
* PDF
* XML

and you want those acronyms to remain uppercase. 

---

# Autoloading the `lib/` Directory

Many Rails applications eventually put custom code inside:

```text
lib/
```

For example:

```text
lib/my_lib.rb
```

However, Rails does **not automatically treat `lib/` as an autoload path in the same way as directories such as `app/models`.

If you want Zeitwerk to manage `lib/`, you can add it to the autoload paths.

For example, in:

```text
config/application.rb
```

you can write:

```ruby
config.autoload_paths << Rails.root.join("lib")
```

Now Rails/Zeitwerk will manage that directory.



---

### Important rule for `lib/`

Once you add `lib/` as an autoload path, the same Zeitwerk rules apply.

For example:

```text
lib/my_lib.rb
```

must define:

```ruby
MyLib
```

If it contains something like:

```ruby
puts "Hello"
```

without defining the expected constant, Zeitwerk can complain.

This is important because `lib/` often contains utility scripts or files that aren't actually classes/modules.

Those files don't necessarily belong in an autoload path.



---

# 6. Common Mistakes

These mistakes tend to happen at different experience levels. 

---

## Junior mistake: File name doesn't match the constant

Suppose you create:

```text
User.rb
```

instead of:

```text
user.rb
```

or you have:

```text
stripe_service.rb
```

but the file defines:

```ruby
class Stripe
end
```

Zeitwerk expects the file and constant names to match.

For:

```text
stripe_service.rb
```

it expects:

```ruby
StripeService
```

So you should follow:

```text
snake_case.rb
      ↓
CamelCase
```

---

## Mid-level mistake: Putting scripts inside autoload paths

Imagine you put a Rake task or some utility script inside:

```text
app/models/
```

For example:

```text
app/models/import_data.rb
```

but the file doesn't define:

```ruby
ImportData
```

Zeitwerk will have a problem because it assumes:

```text
import_data.rb
       ↓
ImportData
```

Every file in an autoload path is expected to follow the file-to-constant mapping.

So files that don't define constants should generally not be placed in these autoload paths. 

---

## Senior-level mistake: Using reloadable constants inside initializers

This is a more subtle problem.

Imagine an initializer does something like:

```ruby
APP_CONFIG = Configuration.new
```

where:

```text
app/models/configuration.rb
```

contains:

```ruby
class Configuration
end
```

The problem is that `Configuration` is a **reloadable application constant**.

During development, Rails may remove and reload that class when the code changes.

But the initializer has already stored a reference to the old class/object.

You can then end up with confusing errors such as:

```text
ArgumentError:
A copy of User has been removed from the module tree
```

The basic problem is:

```text
Initializer
    ↓
stores reference to reloadable class
    ↓
Rails reloads application
    ↓
old class is removed
    ↓
initializer still points to old class
    ↓
confusing errors
```

This is why you need to be careful when referencing reloadable application constants during initialization. 

---

# 7. Performance Considerations

Zeitwerk is designed to make Rails' loading process both convenient and efficient. 

---

## Boot Time

In development, Zeitwerk doesn't need to execute every application file during boot.

Instead, it mostly registers the `autoload` mappings.

For example:

```text
User → user.rb
Order → order.rb
Payment → payment.rb
```

The files themselves can wait until the constants are actually used.

This helps keep development boot time relatively fast. 

---

## Memory

Production is different.

Rails generally uses:

```ruby
config.eager_load = true
```

This means Rails loads the application's code upfront.

That requires more memory during startup because more code is loaded immediately.

However, this has an important advantage for multi-threaded servers such as Puma.

The application doesn't have multiple threads simultaneously trying to discover and load application code for the first time. 

---

## The Trade-off

So you can think of the development/production difference like this:

### Development

```text
Lazy loading
    ↓
Files loaded when needed
    ↓
Faster boot
    ↓
Easy reloading
```

### Production

```text
Eager loading
    ↓
Everything loaded upfront
    ↓
More memory used
    ↓
Safer concurrent execution
```

So the trade-off is:

> **Development prioritizes fast boot and reloading, while production prioritizes predictable and safe concurrent execution.** 

---

# 8. Security Considerations

Zeitwerk maps files on your filesystem to Ruby constants.

That means filesystem structure can directly influence what Ruby code gets loaded.

For example:

```text
some_file.rb
     ↓
SomeFile
```

Because of this, you should be careful about which directories you add to `autoload_paths`.

You should **never add a directory that accepts arbitrary user uploads** to your autoload paths.

Imagine users can upload files to:

```text
uploads/
```

and you accidentally configure:

```text
uploads/
```

as an autoload path.

Now files controlled by users could potentially become part of your application's code-loading process.

The normal Rails configuration is safe because your application controls its autoload paths.

The important rule is:

> Keep autoload paths restricted to directories containing trusted application code. 

---

# 9. Debugging Zeitwerk

One of the nice things about Zeitwerk is that it tends to **fail loudly**.

Instead of silently guessing what you meant, it tells you when your file/constant structure doesn't follow its rules. 

---

## `NameError: uninitialized constant Foo`

If you see:

```text
NameError: uninitialized constant Foo
```

possible causes include:

* The file doesn't exist.
* The file isn't inside an autoload path.
* The constant name is misspelled.
* The file name doesn't match the constant.
* The constant wasn't defined inside the file.

For example:

```text
app/models/foo.rb
```

but the file contains:

```ruby
class Bar
end
```

Zeitwerk expects:

```ruby
Foo
```

but finds:

```ruby
Bar
```

---

## `Zeitwerk::NameError`

You might see an error such as:

```text
Zeitwerk::NameError:
expected file foo.rb to define constant Foo
```

This error is actually very helpful.

It means:

> "I found `foo.rb`, but after loading it, I couldn't find the `Foo` constant I expected."

So check the file.

You probably have something like:

```ruby
class Bar
end
```

when Zeitwerk expected:

```ruby
class Foo
end
```

Check:

* Spelling
* File name
* Constant name
* CamelCase
* Namespaces



---

## Useful Debugging Tools

### `bin/rails zeitwerk:check`

One of the most useful commands is:

```bash
bin/rails zeitwerk:check
```

This checks whether your application's file names and constants match correctly.

It essentially tries to verify that your application can be eager loaded successfully.

It's a good idea to run this:

* Before upgrading Rails
* Before deploying
* In CI
* When debugging autoloading problems



---

### Enable Zeitwerk logs

If you want to see what Zeitwerk is doing, you can enable its logger.

For example, in:

```text
config/application.rb
```

you can use:

```ruby
Rails.autoloaders.logger = method(:puts)
```

This gives you verbose output showing what Zeitwerk is registering and loading.

This can be very useful when you're trying to understand:

> "Why isn't Rails loading this class?"



---

# 10. Best Practices

Here are the main rules you should follow when working with Zeitwerk. 

### 1. Follow Rails naming conventions

The simplest rule is:

```text
snake_case.rb
      ↓
CamelCase
```

For example:

```text
payment_processor.rb
        ↓
PaymentProcessor
```

Following this convention means you usually don't need any custom configuration.

---

### 2. Use `zeitwerk:check` in CI

Add:

```bash
bin/rails zeitwerk:check
```

to your CI process.

This helps catch file/constant mismatches before your application reaches production.

You don't want production deployment to fail because:

```text
payment_service.rb
```

actually defines:

```ruby
Payment
```

instead of:

```ruby
PaymentService
```

---

### 3. Use `ActiveSupport::Reloader.to_prepare`

Sometimes you genuinely need to run code that uses application constants during boot.

For example, you might want to dynamically add methods to a model.

In those situations, you can use:

```ruby
ActiveSupport::Reloader.to_prepare do
  # Code here
end
```

The important part is that this code runs:

1. During application boot.
2. Again whenever Rails reloads the application.

This prevents problems caused by keeping references to old versions of reloadable classes. 

---

# 11. Anti-Patterns

An anti-pattern is basically a way of doing something that may appear to work but is **not the correct approach** for the system.

---

## Don't use `require` or `require_relative` for your own files inside `app/`

Suppose you have:

```text
app/models/user.rb
```

and another application file does:

```ruby
require_relative "../models/user"
```

You generally should **not do this**.

Zeitwerk is already responsible for loading your application files.

Instead of:

```ruby
require_relative "../models/user"
```

just use:

```ruby
User
```

and let Zeitwerk handle the loading.

Mixing manual `require` calls with Zeitwerk can interfere with:

* Reloading
* Constant tracking
* File ownership
* Loading behavior

It can also lead to warnings such as:

```text
constant already defined
```



---

## Don't use `class_eval` to avoid proper folder structure

Your folder structure should match your module structure.

For example:

```text
app/services/billing/invoice.rb
```

should correspond to:

```ruby
module Billing
  class Invoice
  end
end
```

So:

```text
billing/invoice.rb
       ↓
Billing::Invoice
```

Don't try to keep the file somewhere unrelated and then use `class_eval` or similar tricks to make the namespace work.

The basic rule is:

> **Your directory structure should represent your Ruby namespace structure.** 

---

# 12. Interview Questions

These are useful questions to understand for Rails interviews.

---

## Basic

### Q: What is the difference between `require` and `autoload`?

### Answer

`require` loads and executes a Ruby file **immediately**.

For example:

```ruby
require "user"
```

means Ruby loads the file right away.

`autoload` is different.

It registers a relationship between:

```text
constant → file
```

but doesn't execute the file immediately.

For example:

```ruby
autoload :User, "/path/to/user.rb"
```

means:

> "If someone actually uses `User`, load this file."

So:

```text
require
  ↓
Load now


autoload
  ↓
Register now
  ↓
Load later when needed
```



---

## Intermediate

### Q: Why did Rails switch from the classic autoloader to Zeitwerk?

### Answer

The classic autoloader relied on:

```ruby
const_missing
```

This caused problems with:

* Nested namespaces
* Constant resolution
* Circular dependencies
* Predictability
* Thread safety

Zeitwerk uses Ruby's native:

```ruby
Kernel#autoload
```

mechanism.

It establishes the file-to-constant relationship ahead of time.

That makes autoloading:

* More predictable
* Easier to reason about
* Strict about file naming
* Better suited for concurrent applications



---

## Senior

### Q: How do you handle acronyms such as `API` or `HTML` in file names with Zeitwerk?

### Answer

By configuring a custom inflector.

For example:

```ruby
Rails.autoloaders.main.inflector.inflect(
  "api" => "API"
)
```

Then a file such as:

```text
api_controller.rb
```

can map to:

```ruby
APIController
```

instead of the default:

```ruby
ApiController
```

The same idea can be used for acronyms such as `HTML`, `PDF`, or `XML`. 

---

## Staff

### Q: How does eager loading work in production with Puma, and why can lazy loading be dangerous?

Puma is a **multi-threaded web server**.

Imagine your application is running and two requests arrive at almost exactly the same time.

If the application is lazily loading a particular file, both requests could potentially trigger loading behavior around the same code.

Ruby's `require` itself is generally thread-safe because Ruby uses synchronization around it, but complicated dependencies between files can still create difficult situations such as:

* Deadlocks
* Race conditions
* Complex cross-dependencies

That's one reason Rails uses eager loading in production.

With eager loading:

```text
Application starts
       ↓
All application files are loaded
       ↓
Puma starts handling requests
       ↓
Threads already have access to loaded classes
```

This means the application's object graph is already established before the application starts handling concurrent work.

It also works well with **Copy-on-Write (CoW)** memory optimization when using multiple Puma worker processes.

The basic idea is:

> Load the application once, then allow worker processes to share memory pages where possible instead of each worker independently loading the same application code.



---

# 13. Practical Coding Examples

Now let's connect the rules to actual Rails code. 

---

## Example 1: Standard Autoloading

File:

```text
app/services/payment_processor.rb
```

Inside:

```ruby
class PaymentProcessor
  # Code here
end
```

Zeitwerk sees:

```text
payment_processor.rb
```

and expects:

```ruby
PaymentProcessor
```

So the mapping is:

```text
app/services/payment_processor.rb
              ↓
       PaymentProcessor
```

This is the standard Zeitwerk convention. 

---

## Example 2: Nested Namespaces

Suppose you have:

```text
app/services/billing/invoice_generator.rb
```

Inside the file:

```ruby
module Billing
  class InvoiceGenerator
    # Code here
  end
end
```

The directory:

```text
billing/
```

represents:

```ruby
Billing
```

and the file:

```text
invoice_generator.rb
```

represents:

```ruby
InvoiceGenerator
```

Together:

```text
billing/invoice_generator.rb
          ↓
Billing::InvoiceGenerator
```



---

## Example 3: Collapsed Directories

Consider:

```text
app/models/concerns/searchable.rb
```

The `concerns` directory is collapsed by Rails.

Therefore:

```ruby
module Searchable
end
```

is correct.

You should **not** write:

```ruby
module Concerns
  module Searchable
  end
end
```

So:

```text
app/models/concerns/searchable.rb
              ↓
         Searchable
```

not:

```text
app/models/concerns/searchable.rb
              ↓
     Concerns::Searchable
```



---

# 14. Edge Cases

There are some situations where the normal lazy-loading rules can become more complicated. 

---

## Single Table Inheritance — STI

Suppose you have:

```ruby
class Vehicle < ApplicationRecord
end
```

and:

```ruby
class Car < Vehicle
end
```

and:

```ruby
class Truck < Vehicle
end
```

Your database might have:

```text
vehicles
----------------
id
type
name
```

The `type` column might contain:

```text
Car
Truck
```

When you run:

```ruby
Vehicle.all
```

ActiveRecord needs to know about the child classes:

```ruby
Car
Truck
```

so it can look at the `type` value and create the correct Ruby object.

The problem is that in development, Zeitwerk may be using **lazy loading**.

That means:

```ruby
Car
```

may not have been loaded yet.

So STI can sometimes require special handling.

The source recommends explicitly preloading STI subclasses or using specialized loading patterns such as `require_dependency` where appropriate, although `require_dependency` is deprecated and specialized approaches exist for eager-loading particular directories during development. 

---

## Multiple Constants in One File

You can define multiple constants in a single file.

For example:

```text
payment.rb
```

could contain:

```ruby
class Payment
end

class PaymentError
end
```

Zeitwerk primarily cares that the **main constant matching the file name** exists.

So:

```text
payment.rb
```

must define:

```ruby
Payment
```

The additional:

```ruby
PaymentError
```

can exist too.

However, there's an important limitation:

> `PaymentError` will not automatically have its own independent autoload mapping from this file.

In both development and production, PaymentError will only be loaded if some code loads payment.rb (e.g., by referencing Payment, which matches the file name). Zeitwerk won't autoload PaymentError directly if it's never referenced, and referencing PaymentError alone won't trigger loading of payment.rb. To access PaymentError, you must first ensure payment.rb is loaded (typically via Payment). This behavior is the same in both environments due to Zeitwerk’s file-to-constant mapping rule.

In other words, Zeitwerk expects the primary constant to match the file name. Auxiliary constants in the same file won't become independently autoloadable just because they are defined there. 

---

# 15. Comparison Table

| Feature                     | `require`                       | `classic` Autoloader            | Zeitwerk                  |
| --------------------------- | ------------------------------- | ------------------------------- | ------------------------- |
| **Mechanism**               | Immediately executes the file   | Uses `Module#const_missing`     | Uses `Kernel#autoload`    |
| **Thread Safe?**            | Yes, mostly                     | No                              | Yes                       |
| **Reloading?**              | No                              | Yes                             | Yes                       |
| **File ↔ Constant Mapping** | You manage it manually          | Guessed / loose                 | Strict / predictable      |
| **Performance**             | Can use more memory during boot | Slow when constants are missing | Fast and memory efficient |

The important thing to remember is that these systems work differently:

```text
require
  ↓
Load immediately


classic
  ↓
Wait for missing constant
  ↓
Try to guess the file


Zeitwerk
  ↓
Map file → constant ahead of time
  ↓
Load when constant is needed
```



---

# 16. Related Topics to Study Next

If you want to understand Zeitwerk at a deeper Rails-engineer level, these are the next topics worth studying. 

### 1. Ruby Constant Resolution

Learn how Ruby decides what this means:

```ruby
User
```

depending on where the code is written.

You should understand:

* Lexical constant lookup
* Inheritance-based lookup
* Nested modules
* Top-level constants

This helps you understand **why the old `const_missing` approach could fail**.

---

### 2. Rack and Middleware

Learn how Rails requests move through:

```text
Rack
 ↓
Middleware
 ↓
Rails
 ↓
Controller
```

This helps explain where Rails' reloading behavior fits into the request lifecycle.

---

### 3. Copy-on-Write Memory

Learn how **Copy-on-Write (CoW)** works with multiple Puma worker processes.

This explains why eager loading the application before workers are created can help reduce duplicated memory usage.



---

# 17. Summary — Revision Sheet

If you need to quickly revise Zeitwerk before an interview, remember these points:

* **Zeitwerk** = The default Rails autoloader from Rails 6 onwards.
* **Replaces** = The old `classic` autoloader that relied on `const_missing`.
* **Uses** = `Kernel#autoload` for lazy loading and `require`/eager loading behavior when the application is eager loaded.
* **Rule** = File paths and constant names must match.
* **Directory** = Usually represents a namespace/module.
* **Command** = `bin/rails zeitwerk:check`.



The single most important rule to remember is:

```text
FILE PATH
   ↓
CONSTANT NAME
```

For example:

```text
app/services/payment_processor.rb
              ↓
       PaymentProcessor
```

and:

```text
app/services/billing/invoice_generator.rb
              ↓
       Billing::InvoiceGenerator
```

---

# 18. Cheat Sheet

### Check whether your application follows Zeitwerk conventions

```bash
bin/rails zeitwerk:check
```

---

### Configure a custom acronym/inflection

In:

```text
config/initializers/zeitwerk.rb
```

you can write:

```ruby
Rails.autoloaders.main.inflector.inflect(
  "pdf_generator" => "PDFGenerator"
)
```

This makes:

```text
pdf_generator.rb
        ↓
PDFGenerator
```

instead of:

```text
PdfGenerator
```

---

### Add `lib/` to the autoload paths

In:

```text
config/application.rb
```

you can write:

```ruby
config.autoload_paths << Rails.root.join("lib")
```

Then Zeitwerk can manage the code inside `lib/`.

---

### Safely run code on boot and after reloads

Use:

```ruby
ActiveSupport::Reloader.to_prepare do
  # Code here
end
```

This allows the code to run:

```text
Application boot
      +
Every application reload
```



---

# 19. Practice Exercises

These exercises test whether you actually understand the file-to-constant mapping rules.

---

### Easy

You create:

```text
app/services/data/export_service.rb
```

and the file contains:

```ruby
class ExportService
end
```

**Question:**

What is the full module namespace of this class?

**Answer:**

```ruby
Data::ExportService
```

Why?

Because:

```text
data/
 ↓
Data

export_service.rb
 ↓
ExportService
```

Together:

```ruby
Data::ExportService
```



---

### Medium

You create:

```text
app/poros/UserStats.rb
```

and Rails gives you a Zeitwerk error.

Why?

Because the file name is:

```text
UserStats.rb
```

Zeitwerk expects Rails/Ruby file names to use `snake_case`.

It should be:

```text
user_stats.rb
```

which maps to:

```ruby
UserStats
```

So:

```text
user_stats.rb
      ↓
UserStats
```



---

### Hard

Suppose you have an initializer:

```ruby
APP_CONFIG = Configuration.new
```

and:

```text
app/models/configuration.rb
```

contains:

```ruby
class Configuration
end
```

In development, you change:

```text
configuration.rb
```

but:

```ruby
APP_CONFIG
```

still points to the old version.

Sometimes you also get:

```text
ArgumentError
```

because a copy of the class has been removed from the module tree.

**Question:**

How should you fix this architecture?

The core issue is that the initializer is keeping a reference to a **reloadable application constant**.

The solution is to structure the initialization/reloading behavior so that application constants are resolved again when Rails reloads, rather than keeping stale references to the old class.

`ActiveSupport::Reloader.to_prepare` is the relevant Rails mechanism for code that needs to run during boot and again after reloads.



---

# 20. Additional Resources

The original material recommends these resources for going deeper:

* **Official Zeitwerk GitHub repository** — Zeitwerk is written by Xavier Noria, a Rails Core contributor.
* **Rails Guides — Autoloading and Reloading Constants** — the official Rails documentation for understanding how Rails handles constants and reloading.
* **Book: *Rebuilding Rails* by Noah Gibbs** — useful for understanding Rails internals and the concepts behind autoloading.



---

## The whole concept in one simple picture

If you remember nothing else, remember this:

```text
                 RAILS STARTS
                      │
                      ▼
                 Zeitwerk starts
                      │
                      ▼
             Scans application files
                      │
                      ▼
       ┌──────────────┴──────────────┐
       │                             │
       ▼                             ▼
user.rb                       billing/
       │                             │
       ▼                             ▼
    User                    Billing module
                                     │
                                     ▼
                           invoice_generator.rb
                                     │
                                     ▼
                         Billing::InvoiceGenerator
```

Then when your code says:

```ruby
User.new
```

Zeitwerk already knows:

```text
User → user.rb
```

So Ruby loads the correct file.

And when your code says:

```ruby
Billing::InvoiceGenerator.new
```

Zeitwerk already knows:

```text
Billing
   ↓
billing/

Billing::InvoiceGenerator
   ↓
billing/invoice_generator.rb
```

That's essentially the **core idea behind Zeitwerk**:

> **Rails looks at your folder/file structure, maps it to Ruby constants, and lets Ruby load those files automatically when the constants are needed.**
