# Mastering Zeitwerk Autoloading in Ruby on Rails

## 1. Overview
*   **What it is:** Zeitwerk is an efficient, thread-safe code loader for Ruby. Starting with Rails 6, it is the default autoloader, entirely replacing the legacy `classic` autoloader. It automatically maps file paths to constant names and loads files on demand.
*   **Why it exists:** The classic Rails autoloader relied on `Module#const_missing`. This was fundamentally flawed because of Ruby's constant resolution algorithms. It led to non-deterministic behavior, issues with nested namespaces, "circular dependency" errors, and lack of thread safety. Zeitwerk solves this by parsing the file tree upfront and leveraging Ruby's native `Kernel#autoload`.
*   **When to use it:** In Rails 6+, it is enabled by default. You "use" it by following standard Rails file naming conventions. It can also be used outside of Rails as a standalone code loader for any Ruby project.
*   **Common Misconceptions:**
    *   *Misconception 1: "Zeitwerk eager loads all code on boot."* Reality: In development, it lazy-loads code using `autoload`. Eager loading is only enabled in production environments for speed and thread safety.
    *   *Misconception 2: "Zeitwerk handles gems."* Reality: Zeitwerk only manages the files in directories explicitly added to its autoload paths (like `app/models`). Standard `require` is still needed for gems.

## 2. Core Concepts
*   **Autoloading:** The mechanism of delaying the loading of a file until the constant it defines is actually used for the first time.
*   **Eager Loading:** Loading every file in the application's eager load paths before the application starts accepting requests.
*   **Reloading:** Unloading all constants and reloading the files from disk. Useful in development so you don't need to restart the server on every code change.
*   **Autoload Paths:** The list of directories managed by Zeitwerk (e.g., `app/models`, `app/services`). Every file inside an autoload path must define a constant corresponding to its file path.
*   **Root Namespace:** The namespace assigned to an autoload path. By default, it is `Object` (top-level). This means `app/models/user.rb` should define `User`, not `Models::User`.
*   **Inflection:** The rules used to translate file names to constant names (`user.rb` -> `User`, `html_parser.rb` -> `HtmlParser`).
*   **Collapsed Directories:** Directories used purely for organization that do not act as namespaces. For example, `app/models/concerns` is collapsed by Rails, so `app/models/concerns/sluggable.rb` defines `Sluggable`, not `Concerns::Sluggable`.

## 3. Internal Working
Understanding the shift from `classic` to Zeitwerk is key to mastering this topic.

### The Classic Autoloader Flaw (`const_missing`)
1.  Ruby executes code and hits an unknown constant: `User`.
2.  Ruby calls `const_missing(:User)`.
3.  Rails catches this, guesses the file name (`user.rb`), and tries to `require` it.
4.  **The Flaw:** If you are inside `Admin::DashboardController` and reference `User`, Ruby's constant resolution might incorrectly resolve to a top-level `User` or throw an error before Rails' `const_missing` hook is triggered accurately.

### Zeitwerk's Approach (`Kernel#autoload`)
Zeitwerk uses Ruby's built-in `Kernel#autoload` to map paths to constants *before* code executes.
1.  **Boot (Setup):** Rails tells Zeitwerk which directories to manage (`app/`).
2.  **Scanning:** Zeitwerk recursively scans these directories.
3.  **Registration:** For every file, it sets an `autoload`. For `app/models/user.rb`, it internally calls:
    `autoload :User, "/path/to/app/models/user.rb"`
4.  **Execution:** When your code eventually references `User`, Ruby intercepts it natively, loads the file, defines the constant, and resumes execution seamlessly.
5.  **Namespaces (Directories):** When Zeitwerk sees a directory `app/services/billing`, it defines a module `Billing` on the fly and sets up `autoload` for files inside it using `TracePoint`.

## 4. Architecture
Zeitwerk sits at the foundation of the application initialization process.
*   **Rails Initialization:** During boot, `Rails::Application::Bootstrap` configures a Zeitwerk loader instance.
*   **ActionDispatch Middleware:** In development, a middleware clears the Zeitwerk loaded constants at the end of every request.
*   **Dependencies:** Replaces `ActiveSupport::Dependencies` which managed the classic autoloader.

## 5. Real Production Examples

### Custom Inflectors (GraphQL / API)
In a production app, you might have `app/services/api_client.rb`. Zeitwerk expects `ApiClient`. If you want `APIClient`, you configure an inflector.
```ruby
# config/initializers/zeitwerk.rb
Rails.autoloaders.main.inflector.inflect(
  "api_client" => "APIClient",
  "html_parser" => "HTMLParser"
)
```

### Autoloading lib/ directory
Many mature apps move code to `lib/`. By default, Rails doesn't autoload `lib/`. To use Zeitwerk for it:
```ruby
# config/application.rb
config.autoload_paths << Rails.root.join("lib")
```
*If `lib/my_lib.rb` exists, it must define `MyLib`. If you have utility scripts in `lib/` that don't define constants, Zeitwerk will throw an error.*

## 6. Common Mistakes
*   **Junior:** Mismatching file names and constant names (e.g., file `User.rb` instead of `user.rb`, or file `stripe_service.rb` defining `Stripe`).
*   **Mid-Level:** Putting scripts that don't define classes/modules in autoload paths (like Rake tasks inside `app/models`). Zeitwerk expects every file to define a constant matching its name.
*   **Senior:** Referencing reloadable constants inside initializers. Initializers run *before* eager loading is complete. If you cache a model class in an initializer, it won't be reloaded in development, leading to confusing `ArgumentError` (A copy of User has been removed from the module tree).

## 7. Performance Considerations
*   **Boot Time:** Zeitwerk is extremely fast because it only registers `autoload` calls. It doesn't evaluate file contents during boot in development.
*   **Memory:** By eager loading in production (`config.eager_load = true`), Rails loads everything upfront. This takes more memory but prevents threads in a multi-threaded web server (Puma) from colliding while trying to require the same file simultaneously.
*   **Trade-off:** Fast boot in development (lazy loading), safe concurrent execution in production (eager loading).

## 8. Security Considerations
*   **Path Traversal/Code Injection:** Because Zeitwerk maps file paths to Ruby constants, controlling the file system translates directly to application logic. However, since autoload paths are hardcoded in application configuration, this vector is securely locked down natively. Ensure `autoload_paths` never includes directories that accept user uploads.

## 9. Debugging
When Zeitwerk fails, it fails loudly (which is a feature).
*   **`NameError: uninitialized constant Foo`**: The file doesn't exist, isn't in an autoload path, or is misspelled.
*   **`Zeitwerk::NameError: expected file foo.rb to define constant Foo`**: The file was found and loaded, but it didn't define the expected constant. Check your spelling and camel casing.
*   **Tools:**
    *   `bin/rails zeitwerk:check`: A must-run command before upgrading or pushing to production. It attempts to eager load everything and verifies all file/constant mappings.
    *   Setting `Rails.autoloaders.logger = method(:puts)` in `config/application.rb` provides verbose logs of what Zeitwerk is registering and loading.

## 10. Best Practices
1.  **Stick to conventions:** Let `snake_case.rb` map to `CamelCase`.
2.  **Use `bin/rails zeitwerk:check` in CI:** Ensure your eager loading won't fail in production.
3.  **Use `ActiveSupport::Reloader.to_prepare`:** If you must use application constants on boot (e.g., dynamically defining methods on models), wrap the code in a `to_prepare` block so it runs upon boot AND upon every reload.

## 11. Anti-patterns
*   **Using `require` or `require_relative` inside `app/`:** Never do this for your own application files. Let Zeitwerk handle it. Mixing `require` and Zeitwerk breaks reloading and causes "constant already defined" warnings.
*   **Namespacing modules with `class_eval` to dodge file structures.** The folder structure *must* match the module nesting. (e.g., `Billing::Invoice` MUST be in `app/.../billing/invoice.rb`).

## 12. Interview Questions

### Basic
*   **Q: What is the difference between `require` and `autoload`?**
    *   **A:** `require` evaluates the file immediately. `autoload` registers the file path to a constant name, but postpones evaluating the file until the constant is actually referenced in the code.

### Intermediate
*   **Q: Why did Rails switch from the classic autoloader to Zeitwerk?**
    *   **A:** The classic autoloader relied on `const_missing`, which caused issues with nested module resolution and circular dependencies. Zeitwerk uses native `Kernel#autoload`, making it predictable, matching file paths exactly, and strictly thread-safe.

### Senior
*   **Q: How do you handle acronyms like "API" or "HTML" in file names with Zeitwerk?**
    *   **A:** By using `Rails.autoloaders.main.inflector.inflect("api" => "API")` in an initializer, so `api_controller.rb` loads `APIController` instead of `ApiController`.

### Staff
*   **Q: Explain how eager loading works in production with Puma (a multi-threaded web server) and why lazy loading would be dangerous.**
    *   **A:** If we lazy loaded in production, two concurrent requests might trigger `require` for the same file simultaneously. While Ruby's `require` is generally thread-safe via mutexes, complex cross-dependencies can lead to deadlocks or race conditions. Eager loading evaluates all application files before the web server forks or spawns threads, ensuring the entire object graph is in read-only memory, maximizing thread safety and leveraging Copy-on-Write (CoW) memory optimization.

## 13. Practical Coding Examples

**Example 1: Standard Autoloading**
```ruby
# app/services/payment_processor.rb
class PaymentProcessor
  # Zeitwerk expects exactly this class name based on the file name.
end
```

**Example 2: Nested Namespaces**
```ruby
# app/services/billing/invoice_generator.rb
module Billing
  class InvoiceGenerator
    # The folder 'billing' maps to the module 'Billing'
  end
end
```

**Example 3: Collapsed Directories**
```ruby
# app/models/concerns/searchable.rb
module Searchable
  # Note it is NOT Concerns::Searchable. 
  # Rails configures Zeitwerk to "collapse" the 'concerns' directory.
end
```

## 14. Edge Cases
*   **Single Table Inheritance (STI):** If you query a parent class (e.g., `Vehicle.all`), ActiveRecord needs to know all child classes (`Car`, `Truck`) to instantiate the correct objects. In development (lazy loading), `Car` might not be loaded yet!
    *   *Solution:* Explicitly preload STI subclasses or use `require_dependency` (though deprecated, specialized patterns exist to eager load specific directories in development).
*   **Defining multiple constants in one file:** Zeitwerk only cares that the *primary* constant matching the filename is defined. Defining auxiliary constants in the same file works, but they won't be autoloadable independently.

## 15. Comparison Table

| Feature | `require` (Ruby) | `classic` (Old Rails) | Zeitwerk (Modern Rails) |
| :--- | :--- | :--- | :--- |
| **Mechanism** | Immediate execution | `Module#const_missing` | `Kernel#autoload` |
| **Thread Safe?** | Yes (mostly) | No | Yes |
| **Reloading?** | No | Yes | Yes |
| **File <-> Constant Mapping** | Manual | Guessed, loose | Strict, predictable |
| **Performance** | Memory heavy on boot | Slow on misses | Fast, memory efficient |

## 16. Related Topics to Study Next
*   **Ruby Constant Resolution:** Understand how Ruby looks up constants lexically and via the inheritance chain (this explains *why* the classic autoloader failed).
*   **Rack and Middleware:** Understand where Zeitwerk reloading fits into the Rack request lifecycle.
*   **Copy-on-Write (CoW) memory:** Understand how eager loading saves RAM across multiple Puma worker processes.

## 17. Summary (Revision Sheet)
*   **Zeitwerk** = Default Rails 6+ autoloader.
*   **Replaces** = `classic` autoloader (`const_missing`).
*   **Uses** = `Kernel#autoload` (lazy load dev) & `require` (eager load prod).
*   **Rule** = File paths must perfectly match Constant names.
*   **Directory** = Namespace (Module).
*   **Command** = `bin/rails zeitwerk:check`.

## 18. Cheat Sheet
```ruby
# Check compatibility
bin/rails zeitwerk:check

# Custom Inflection (config/initializers/zeitwerk.rb)
Rails.autoloaders.main.inflector.inflect("pdf_generator" => "PDFGenerator")

# Add to Autoload Paths (config/application.rb)
config.autoload_paths << Rails.root.join('lib')

# Safely run code on boot AND reload
ActiveSupport::Reloader.to_prepare do
  # Code here
end
```

## 19. Practice Exercises
*   **Easy:** Create a class `ExportService` in `app/services/data/export_service.rb`. What is its full module namespace? (Answer: `Data::ExportService`).
*   **Medium:** You added a folder `app/poros` and put `UserStats.rb` in it, but Rails throws an error. Why? (Answer: The file name is capitalized. It must be `user_stats.rb`).
*   **Hard:** You have an initializer setting `APP_CONFIG = Configuration.new`. `Configuration` is a class in `app/models/configuration.rb`. In development, if you change `configuration.rb`, `APP_CONFIG` doesn't reflect the changes, and sometimes you get `ArgumentError`. Fix this architecture.

## 20. Additional Resources
*   [Official Zeitwerk GitHub Repository](https://github.com/fxn/zeitwerk) (Written by Xavier Noria, Rails Core).
*   [Rails Guides: Autoloading and Reloading Constants](https://guides.rubyonrails.org/autoloading_and_reloading_constants.html).
*   **Book:** *Rebuilding Rails* by Noah Gibbs (for understanding autoloading conceptually).
