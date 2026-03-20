---
title: "Part Nine | Functional Programming Through Elixir: Module Organization vs Class Hierarchies"
date: 2026-03-20
draft: false
tags: ["elixir", "functional-programming", "fp-series", "modules", "behaviours", "protocols", "learning"]
description: "Ninth post in a series exploring functional programming concepts through Elixir. Modules as namespaces, behaviours as contracts, protocols as polymorphism, and why Elixir doesn't need inheritance hierarchies."
---

## Where Does Code Live?

In OOP, the answer is always: in a class. Classes are the organizing unit for everything. They hold state, define behavior, inherit from parent classes, and form hierarchies that model relationships between concepts.

In Elixir, the organizing unit is the **module**. Modules hold functions. That's it. They don't hold state, they don't inherit from other modules, and they don't form hierarchies. This changes how you think about organizing code.

<!--more-->

## Modules as Namespaces

An Elixir module is a namespace for functions. You group related functions together and give the group a name.

{{< highlight elixir >}}
defmodule MyApp.Accounts do
  def create_user(attrs) do
    # ...
  end

  def get_user(id) do
    # ...
  end

  def authenticate(email, password) do
    # ...
  end
end

defmodule MyApp.Accounts.Permissions do
  def can_access?(user, resource) do
    # ...
  end

  def grant(user, role) do
    # ...
  end
end
{{< /highlight >}}

The dot in `MyApp.Accounts.Permissions` looks like a hierarchy, but it's just a naming convention. There's no parent-child relationship between `MyApp.Accounts` and `MyApp.Accounts.Permissions`. They don't share state, they don't inherit functions, and neither one knows the other exists unless it explicitly calls a function from it.

### Contrast: OOP Class Hierarchies

In OOP, nesting implies relationships:

{{< highlight java >}}
// Java - class hierarchy with inheritance
public abstract class Account {
    protected String email;

    public abstract boolean authenticate(String password);

    public String getEmail() {
        return email;
    }
}

public class UserAccount extends Account {
    private String hashedPassword;

    @Override
    public boolean authenticate(String password) {
        return BCrypt.verify(password, hashedPassword);
    }
}

public class AdminAccount extends Account {
    private String hashedPassword;
    private List<String> permissions;

    @Override
    public boolean authenticate(String password) {
        return BCrypt.verify(password, hashedPassword);
    }

    public boolean canAccess(String resource) {
        return permissions.contains(resource);
    }
}
{{< /highlight >}}

{{< highlight python >}}
# Python - class hierarchy
class Account:
    def __init__(self, email):
        self.email = email

    def authenticate(self, password):
        raise NotImplementedError

class UserAccount(Account):
    def __init__(self, email, hashed_password):
        super().__init__(email)
        self.hashed_password = hashed_password

    def authenticate(self, password):
        return bcrypt.verify(password, self.hashed_password)

class AdminAccount(Account):
    def __init__(self, email, hashed_password, permissions):
        super().__init__(email)
        self.hashed_password = hashed_password
        self.permissions = permissions

    def authenticate(self, password):
        return bcrypt.verify(password, self.hashed_password)

    def can_access(self, resource):
        return resource in self.permissions
{{< /highlight >}}

The OOP approach ties data and behavior together in a tree. `AdminAccount` inherits from `Account`, which means it gets `email` and the contract for `authenticate`, but also everything else `Account` carries. If `Account` changes, every subclass changes too. This is the fragile base class problem.

### The Elixir Way: Flat Modules, Explicit Data

{{< highlight elixir >}}
defmodule MyApp.Accounts do
  def authenticate(%{hashed_password: hashed} = _user, password) do
    Bcrypt.verify_pass(password, hashed)
  end
end

defmodule MyApp.Accounts.Permissions do
  def can_access?(%{role: :admin, permissions: perms}, resource) do
    resource in perms
  end

  def can_access?(_, _), do: false
end
{{< /highlight >}}

Data flows in as arguments. Functions pattern match on what they need. There's no shared mutable state between modules, no inheritance chain to trace, and no hidden coupling through a base class. If `Accounts` changes, `Permissions` is unaffected unless you change the shape of the data they both operate on.

## Organizing a Real Project

In OOP, you model your domain as a class hierarchy. In Elixir, you model it as groups of functions organized around a concept. Phoenix calls these **contexts**.

{{< highlight text >}}
OOP Project                          Elixir Project
───────────                          ──────────────
models/                              lib/my_app/
  User.java                            accounts/
  AdminUser.java (extends User)          user.ex          (schema)
  Post.java                             permissions.ex   (functions)
  Comment.java (extends Post?)         accounts.ex        (context)
services/                              blog/
  UserService.java                       post.ex          (schema)
  PostService.java                       comment.ex       (schema)
  AuthService.java                     blog.ex            (context)
controllers/
  UserController.java
{{< /highlight >}}

The Elixir side is flat. `User` doesn't extend anything. `AdminUser` doesn't exist as a separate type. The difference between a regular user and an admin is a field on the struct and pattern matching in `Permissions`.

{{< highlight elixir >}}
defmodule MyApp.Accounts.User do
  defstruct [:id, :email, :hashed_password, :role, :permissions]
end

defmodule MyApp.Accounts do
  alias MyApp.Accounts.User

  def create_user(attrs) do
    %User{
      email: attrs.email,
      hashed_password: hash(attrs.password),
      role: :user,
      permissions: []
    }
  end

  def promote_to_admin(user, permissions) do
    %{user | role: :admin, permissions: permissions}
  end
end

defmodule MyApp.Accounts.Permissions do
  def can_access?(%{role: :admin, permissions: perms}, resource) do
    resource in perms
  end

  def can_access?(%{role: :user}, _resource), do: false
end

# Usage
user = MyApp.Accounts.create_user(%{email: "alice@example.com", password: "secret"})
admin = MyApp.Accounts.promote_to_admin(user, ["dashboard", "reports"])

MyApp.Accounts.Permissions.can_access?(admin, "dashboard")  # true
MyApp.Accounts.Permissions.can_access?(user, "dashboard")   # false
{{< /highlight >}}

No inheritance. No abstract base class. One struct with a role field, and functions that pattern match on it. Adding a new role means adding function clauses, not a new class in a hierarchy.

## Behaviours: Contracts Without Inheritance

OOP uses abstract classes and interfaces to define contracts. Elixir uses **behaviours**.

A behaviour defines a set of functions that a module must implement. It's a contract, not a parent class. It doesn't provide implementation. It doesn't share state. It just says "if you claim to be this kind of module, you must have these functions."

{{< highlight elixir >}}
defmodule MyApp.Notifier do
  @doc "Send a notification to a user"
  @callback notify(user :: map(), message :: String.t()) :: :ok | {:error, String.t()}

  @doc "Check if the notifier is available"
  @callback available?() :: boolean()
end
{{< /highlight >}}

Now any module can implement this behaviour:

{{< highlight elixir >}}
defmodule MyApp.Notifier.Email do
  @behaviour MyApp.Notifier

  @impl true
  def notify(user, message) do
    # send email to user.email
    :ok
  end

  @impl true
  def available?, do: true
end

defmodule MyApp.Notifier.SMS do
  @behaviour MyApp.Notifier

  @impl true
  def notify(user, message) do
    # send SMS to user.phone
    :ok
  end

  @impl true
  def available?, do: true
end

defmodule MyApp.Notifier.Slack do
  @behaviour MyApp.Notifier

  @impl true
  def notify(user, message) do
    # post to user's Slack channel
    :ok
  end

  @impl true
  def available? do
    # check if Slack integration is configured
    Application.get_env(:my_app, :slack_token) != nil
  end
end
{{< /highlight >}}

If a module declares `@behaviour MyApp.Notifier` but doesn't implement `notify/2` or `available?/0`, the compiler warns you. The `@impl true` annotation makes it explicit which functions satisfy the contract.

### Using Behaviours for Dispatch

{{< highlight elixir >}}
defmodule MyApp.Notifications do
  @notifiers [
    MyApp.Notifier.Email,
    MyApp.Notifier.SMS,
    MyApp.Notifier.Slack
  ]

  def send_all(user, message) do
    @notifiers
    |> Enum.filter(& &1.available?())
    |> Enum.each(& &1.notify(user, message))
  end

  def send_via(notifier, user, message) do
    if notifier.available?() do
      notifier.notify(user, message)
    else
      {:error, "Notifier not available"}
    end
  end
end

# Send through all available channels
MyApp.Notifications.send_all(user, "Your order shipped!")

# Send through a specific channel
MyApp.Notifications.send_via(MyApp.Notifier.Email, user, "Reset your password")
{{< /highlight >}}

Modules are values in Elixir. You can store them in a list, pass them to functions, and call their functions dynamically. This gives you polymorphic dispatch without any inheritance.

### Contrast: OOP Interfaces and Abstract Classes

{{< highlight java >}}
// Java - interface with implementations
public interface Notifier {
    void notify(User user, String message);
    boolean isAvailable();
}

public class EmailNotifier implements Notifier {
    @Override
    public void notify(User user, String message) {
        // send email
    }

    @Override
    public boolean isAvailable() {
        return true;
    }
}

public class SMSNotifier implements Notifier {
    @Override
    public void notify(User user, String message) {
        // send SMS
    }

    @Override
    public boolean isAvailable() {
        return true;
    }
}

// Usage - need to instantiate objects
List<Notifier> notifiers = List.of(
    new EmailNotifier(),
    new SMSNotifier()
);

for (Notifier n : notifiers) {
    if (n.isAvailable()) {
        n.notify(user, "Your order shipped!");
    }
}
{{< /highlight >}}

The structure is similar. But there's a difference in what's happening underneath. In Java, `Notifier` is a type. `EmailNotifier` is a subtype. Dispatch happens through the object's vtable at runtime. In Elixir, `MyApp.Notifier` is just a set of callback definitions. `MyApp.Notifier.Email` is a module that happens to implement those callbacks. Dispatch happens because you called a function on a module, the same way you call any function.

The practical difference: Java's approach ties you to an object hierarchy. You need an instance of `EmailNotifier` to call `notify`. Elixir's approach is just modules and functions. You can swap, compose, and configure them without worrying about object lifecycle.

## Protocols: Polymorphism on Data

Behaviours let you define contracts that modules implement. **Protocols** let you define contracts that data types implement. This is Elixir's answer to the question OOP solves with inheritance-based polymorphism: how do you make the same function work differently depending on what you pass it?

{{< highlight elixir >}}
defprotocol Displayable do
  @doc "Convert a value to a display string"
  def display(value)
end
{{< /highlight >}}

Now implement it for different types:

{{< highlight elixir >}}
defimpl Displayable, for: Map do
  def display(map) do
    map
    |> Enum.map(fn {k, v} -> "#{k}: #{v}" end)
    |> Enum.join(", ")
  end
end

defimpl Displayable, for: List do
  def display(list) do
    "[#{Enum.join(list, ", ")}]"
  end
end

defimpl Displayable, for: BitString do
  def display(string), do: string
end

defimpl Displayable, for: Integer do
  def display(number), do: Integer.to_string(number)
end
{{< /highlight >}}

{{< highlight elixir >}}
Displayable.display(%{name: "Alice", age: 30})  # "age: 30, name: Alice"
Displayable.display([1, 2, 3])                   # "[1, 2, 3]"
Displayable.display("hello")                     # "hello"
Displayable.display(42)                          # "42"
{{< /highlight >}}

The same function name, different behavior based on the data type. No class hierarchy required.

### Protocols for Your Own Structs

Where protocols really shine is when you define them for your own data types:

{{< highlight elixir >}}
defmodule MyApp.User do
  defstruct [:name, :email, :role]
end

defmodule MyApp.Product do
  defstruct [:name, :price, :sku]
end

defimpl Displayable, for: MyApp.User do
  def display(user) do
    "#{user.name} (#{user.role})"
  end
end

defimpl Displayable, for: MyApp.Product do
  def display(product) do
    "#{product.name} - $#{product.price}"
  end
end
{{< /highlight >}}

{{< highlight elixir >}}
user = %MyApp.User{name: "Alice", email: "alice@test.com", role: :admin}
product = %MyApp.Product{name: "Widget", price: 9.99, sku: "W-001"}

Displayable.display(user)     # "Alice (admin)"
Displayable.display(product)  # "Widget - $9.99"

# Works in collections with mixed types
[user, product]
|> Enum.map(&Displayable.display/1)
# ["Alice (admin)", "Widget - $9.99"]
{{< /highlight >}}

### Protocols vs Behaviours: When to Use Which

| | Behaviours | Protocols |
|--|-----------|-----------|
| **Dispatch on** | Which module you call | What data you pass |
| **Define with** | `@callback` in a module | `defprotocol` |
| **Implement with** | `@behaviour` + `@impl` | `defimpl` |
| **Use when** | Multiple modules need the same API (notifiers, storage backends) | One function needs to work on multiple data types |

Behaviours answer: "these modules all do the same kind of thing." Protocols answer: "this operation works differently on different kinds of data."

### Contrast: OOP Inheritance-Based Polymorphism

In OOP, polymorphism comes from inheritance:

{{< highlight java >}}
// Java - polymorphism through inheritance
public abstract class Displayable {
    public abstract String display();
}

public class User extends Displayable {
    private String name;
    private String role;

    @Override
    public String display() {
        return name + " (" + role + ")";
    }
}

public class Product extends Displayable {
    private String name;
    private double price;

    @Override
    public String display() {
        return name + " - $" + price;
    }
}

// Usage
List<Displayable> items = List.of(new User("Alice", "admin"), new Product("Widget", 9.99));
for (Displayable item : items) {
    System.out.println(item.display());
}
{{< /highlight >}}

This works, but there's a catch. What if `User` already extends `Person` and `Product` already extends `Inventory`? Java doesn't have multiple inheritance. You'd need to switch to an interface, which means changing the existing class hierarchy.

Elixir protocols don't have this problem. You can implement any protocol for any type at any time, without modifying the type itself. The implementation lives in a separate `defimpl` block, not inside the struct definition. You can even implement protocols for types you didn't write.

{{< highlight elixir >}}
# Implement Displayable for Elixir's built-in Date type
defimpl Displayable, for: Date do
  def display(date) do
    Calendar.strftime(date, "%B %d, %Y")
  end
end

Displayable.display(~D[2026-03-20])  # "March 20, 2026"
{{< /highlight >}}

Try doing that in Java without modifying the `Date` class or wrapping it in a new one.

## The Bigger Picture

Here's how OOP and Elixir answer the same organizational questions:

| Question | OOP Answer | Elixir Answer |
|----------|-----------|---------------|
| Where do I put related functions? | In a class | In a module |
| How do I share behavior across types? | Inheritance | Behaviours or protocols |
| How do I define a contract? | Interface / abstract class | Behaviour (`@callback`) |
| How do I get polymorphism? | Subclass + override | Protocol (`defimpl`) |
| How do I extend a type I didn't write? | Wrapper class / adapter | `defimpl` for the protocol |
| How do I model "is-a" relationships? | Class hierarchy | You usually don't. Use a field and pattern match. |

The last row is the key insight. OOP models relationships as types: `AdminUser` is a kind of `User`. Elixir models relationships as data: a user has a `:role` field that might be `:admin`. The difference is that data is easier to change. Adding a new role is adding a function clause, not a new class with its own constructor, methods, and place in the hierarchy.

## Key Takeaways

- **Modules are namespaces**, not classes. Dotted names like `MyApp.Accounts.User` are naming conventions, not inheritance
- **No inheritance.** Elixir doesn't have it because it doesn't need it. Composition through functions replaces it
- **Behaviours** define callback contracts that modules must implement. They replace OOP interfaces and abstract classes
- **Protocols** define functions that dispatch on data type. They replace OOP's inheritance-based polymorphism
- **Behaviours dispatch on which module you call.** Protocols dispatch on what data you pass
- **You can implement a protocol for any type**, including types you didn't write, without modifying them
- **"Is-a" relationships become data.** Instead of `AdminUser extends User`, you have a user struct with a role field and functions that pattern match on it

## Try It Yourself

### Exercise 1: Context Module

Build a `Library` context module with a `Book` struct. Books have a title, author, and status (`:available`, `:checked_out`, or `:reserved`). Write functions to create a book, check it out, return it, and list only available books from a collection:

{{< highlight elixir >}}
defmodule Library.Book do
  defstruct [:title, :author, :status]
end

defmodule Library do
  alias Library.Book

  def new_book(title, author) do
    ???
  end

  def check_out(%Book{status: :available} = book) do
    ???
  end

  # What should check_out return for a book that's not available?
  def check_out(%Book{} = book) do
    ???
  end

  def return_book(%Book{status: :checked_out} = book) do
    ???
  end

  def available_books(books) do
    ???
  end
end

# Library.new_book("Elixir in Action", "Sasa Juric")
# => %Library.Book{title: "Elixir in Action", author: "Sasa Juric", status: :available}
#
# book = Library.new_book("Elixir in Action", "Sasa Juric")
# {:ok, checked_out} = Library.check_out(book)
# Library.check_out(checked_out)
# => {:error, "Book is not available"}
{{< /highlight >}}

### Exercise 2: Behaviour

Define a `Storage` behaviour with two callbacks: `save/2` (takes a key and value, returns `:ok` or `{:error, reason}`) and `fetch/1` (takes a key, returns `{:ok, value}` or `{:error, :not_found}`). Then implement it with two modules: one that uses an in-memory map (Agent), and one that writes to a file.

{{< highlight elixir >}}
defmodule MyApp.Storage do
  @callback save(key :: String.t(), value :: any()) :: :ok | {:error, String.t()}
  @callback fetch(key :: String.t()) :: {:ok, any()} | {:error, :not_found}
end

defmodule MyApp.Storage.Memory do
  @behaviour MyApp.Storage

  # Hint: use Agent to hold state
  ???
end

defmodule MyApp.Storage.File do
  @behaviour MyApp.Storage

  # Hint: use File.write/2 and File.read/1
  # Store in /tmp/storage/<key>
  ???
end

# Both should work the same way:
# MyApp.Storage.Memory.save("user:1", "Alice")  # :ok
# MyApp.Storage.Memory.fetch("user:1")           # {:ok, "Alice"}
# MyApp.Storage.Memory.fetch("user:99")          # {:error, :not_found}
{{< /highlight >}}

### Exercise 3: Protocol

Define a `Summary` protocol with a `summarize/1` function. Implement it for three types: a `BlogPost` struct (return the first 50 characters of the body plus "..."), a `User` struct (return the name and role), and plain maps (return the number of keys). Then write a function that takes a mixed list and summarizes everything in it.

{{< highlight elixir >}}
defprotocol Summary do
  def summarize(value)
end

defmodule BlogPost do
  defstruct [:title, :body, :author]
end

defmodule User do
  defstruct [:name, :role]
end

# Implement Summary for BlogPost, User, and Map
???

# Usage:
# items = [
#   %BlogPost{title: "FP", body: "Functional programming is a paradigm that treats...", author: "Alice"},
#   %User{name: "Bob", role: :editor},
#   %{a: 1, b: 2, c: 3}
# ]
#
# Enum.map(items, &Summary.summarize/1)
# ["Functional programming is a paradigm that treats...",
#  "Bob (editor)",
#  "Map with 3 keys"]
{{< /highlight >}}

## Official Documentation to Help You Learn

- [Modules and Functions](https://elixir-lang.org/getting-started/modules-and-functions.html)
- [Behaviours Documentation](https://hexdocs.pm/elixir/behaviours.html)
- [Protocols Documentation](https://hexdocs.pm/elixir/protocols.html)
- [Typespecs and Behaviours](https://elixir-lang.org/getting-started/typespecs-and-behaviours.html)

---

*Part Nine | Functional Programming Through Elixir series*

**Previous in series:** [Part Eight - Guards and Pattern Matching in Function Heads](/posts/fp-series-guards-and-pattern-matching-in-function-heads/)

**Next in series:** Part Ten - Error Handling: Tagged Tuples and Railway-Oriented Programming (Coming Soon)
