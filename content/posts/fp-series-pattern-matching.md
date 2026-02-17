---
title: "Part Four | Functional Programming Through Elixir: Pattern Matching"
date: 2026-02-17
draft: false
tags: ["elixir", "functional-programming", "fp-series", "pattern-matching", "learning"]
description: "Fourth post in a series exploring functional programming concepts through Elixir. Learn how pattern matching replaces conditionals and enables elegant data destructuring."
---

## A Different Way to Work with Data

In [Part Three](/posts/fp-series-pure-functions-vs-side-effects/), we learned about pure functions and side effects. Now we'll tackle one of the most distinctive features of functional programming: **pattern matching**.

Pattern matching is Elixir's superpower for working with data. It's not just a fancy way to assign variables - it's a fundamentally different approach to extracting, validating, and routing data through your program. Once you understand pattern matching, you'll find conditionals, type checking, and data extraction in OOP languages feel unnecessarily verbose.

<!--more-->

## What is Pattern Matching?

In most languages, `=` means "assignment" - put a value into a variable. In Elixir, `=` is the **match operator** - it tries to make the left side match the right side, binding variables as needed.

{{< highlight elixir >}}
# Simple match - looks like assignment
x = 5
IO.puts(x)  # 5

# But it's actually matching!
5 = x  # This works! Left side matches right side
IO.puts("Match succeeded!")

# This fails because 6 doesn't match 5
6 = x  # ** (MatchError) no match of right hand side value: 5
{{< /highlight >}}

The `5 = x` line might look strange, but it makes sense when you think of `=` as "match" rather than "assign". Elixir checks: "Does the left side (5) match the right side (the value of x, which is 5)?" Yes, it does, so the match succeeds.

## Destructuring Data Structures

Pattern matching becomes truly powerful when working with complex data structures. You can extract values in one elegant expression:

### Matching Lists

{{< highlight elixir >}}
# Match the first element and the rest
[first | rest] = [1, 2, 3, 4, 5]
IO.puts(first)     # 1
IO.inspect(rest)   # [2, 3, 4, 5]

# Match specific positions
[a, b, c] = [10, 20, 30]
IO.puts(a)  # 10
IO.puts(b)  # 20
IO.puts(c)  # 30

# Match and ignore with _
[first, _second, third] = [1, 2, 3]
IO.puts(first)   # 1
IO.puts(third)   # 3

# Fails if structure doesn't match
[x, y] = [1, 2, 3]  # ** (MatchError) - expected 2 elements, got 3
{{< /highlight >}}

**Contrast with OOP:** In most languages, you'd need explicit indexing:

{{< highlight python >}}
# Python
numbers = [1, 2, 3, 4, 5]
first = numbers[0]
rest = numbers[1:]

# Or destructuring (Python does support this!)
first, *rest = [1, 2, 3, 4, 5]
{{< /highlight >}}

Python's destructuring is similar, but Elixir's pattern matching goes much deeper, as we'll see.

### Matching Maps and Structs

{{< highlight elixir >}}
# Match and extract specific keys from a map
user = %{name: "Alice", age: 30, role: "admin"}

# Extract just what you need
%{name: username, role: user_role} = user
IO.puts(username)    # Alice
IO.puts(user_role)   # admin

# You don't need to specify all keys
%{name: n} = user
IO.puts(n)  # Alice

# Match fails if key doesn't exist
%{email: e} = user  # ** (MatchError) - no :email key
{{< /highlight >}}

**Compare with OOP:**

{{< highlight javascript >}}
// JavaScript
const user = { name: "Alice", age: 30, role: "admin" };

// Destructuring (modern JS)
const { name: username, role: user_role } = user;

// Traditional way
const username = user.name;
const user_role = user.role;

// JavaScript won't fail if key doesn't exist - just returns undefined
const { email } = user;  // email is undefined, not an error
{{< /highlight >}}

Elixir's pattern matching is **stricter** - if the pattern doesn't match exactly, you get an error. This catches bugs early.

### Matching Tuples

Tuples are commonly used for returning multiple values or tagged data:

{{< highlight elixir >}}
# Match a tuple
{status, value} = {:ok, 42}
IO.puts(status)  # :ok
IO.puts(value)   # 42

# Common pattern: matching function results
case File.read("config.json") do
  {:ok, contents} ->
    IO.puts("File contents: #{contents}")

  {:error, reason} ->
    IO.puts("Error reading file: #{reason}")
end
{{< /highlight >}}

This `{:ok, result}` / `{:error, reason}` pattern is ubiquitous in Elixir. Pattern matching makes it elegant to handle both success and failure cases.

**OOP Comparison:** In OOP, you might use exceptions or return codes:

{{< highlight java >}}
// Java - using exceptions
try {
    String contents = readFile("config.json");
    System.out.println("File contents: " + contents);
} catch (IOException e) {
    System.out.println("Error: " + e.getMessage());
}
{{< /highlight >}}

Exceptions can be thrown from anywhere deep in the call stack, making control flow harder to follow. Elixir's tagged tuples make success and failure explicit at every level.

## Pattern Matching in Function Clauses

Here's where pattern matching truly shines: you can define multiple function clauses with different patterns, and Elixir automatically routes to the right one:

{{< highlight elixir >}}
defmodule Greeter do
  # Match empty list
  def greet([]) do
    "No one to greet!"
  end

  # Match list with one person
  def greet([person]) do
    "Hello, #{person}!"
  end

  # Match list with two people
  def greet([first, second]) do
    "Hello, #{first} and #{second}!"
  end

  # Match list with more than two - use [head | tail]
  def greet([first | rest]) do
    others = Enum.join(rest, ", ")
    "Hello, #{first}, #{others}, and everyone else!"
  end
end

IO.puts(Greeter.greet([]))                    # No one to greet!
IO.puts(Greeter.greet(["Alice"]))            # Hello, Alice!
IO.puts(Greeter.greet(["Bob", "Charlie"]))   # Hello, Bob and Charlie!
IO.puts(Greeter.greet(["Dan", "Eve", "Frank"]))
# Hello, Dan, Eve, Frank, and everyone else!
{{< /highlight >}}

Elixir tries each clause **in order from top to bottom** until it finds one that matches. This replaces long if/else chains or switch statements with elegant, declarative code.

### Replacing Conditionals with Pattern Matching

{{< highlight elixir >}}
defmodule Payment do
  # Match different payment result patterns
  def process_result({:ok, transaction_id}) do
    "Payment successful! Transaction ID: #{transaction_id}"
  end

  def process_result({:error, :insufficient_funds}) do
    "Payment failed: Insufficient funds"
  end

  def process_result({:error, :network_timeout}) do
    "Payment failed: Network timeout, please retry"
  end

  def process_result({:error, reason}) do
    "Payment failed: #{reason}"
  end
end

# Each call routes to the appropriate clause
IO.puts(Payment.process_result({:ok, "TXN123456"}))
# Payment successful! Transaction ID: TXN123456

IO.puts(Payment.process_result({:error, :insufficient_funds}))
# Payment failed: Insufficient funds

IO.puts(Payment.process_result({:error, :card_declined}))
# Payment failed: card_declined
{{< /highlight >}}

**OOP Equivalent:** You'd typically use if/else or switch:

{{< highlight java >}}
// Java
public String processResult(PaymentResult result) {
    if (result.isSuccess()) {
        return "Payment successful! Transaction ID: " + result.getTransactionId();
    } else if (result.getError() == ErrorType.INSUFFICIENT_FUNDS) {
        return "Payment failed: Insufficient funds";
    } else if (result.getError() == ErrorType.NETWORK_TIMEOUT) {
        return "Payment failed: Network timeout, please retry";
    } else {
        return "Payment failed: " + result.getError();
    }
}
{{< /highlight >}}

The pattern matching version is more declarative - each clause is a complete, self-contained case. No nested conditions to parse.

## Pattern Matching with Guards

You can add **guards** to patterns for additional conditions:

{{< highlight elixir >}}
defmodule Pricing do
  # Guard: when age < 13
  def ticket_price(age) when age < 13 do
    5.00
  end

  # Guard: when age >= 13 and age < 65
  def ticket_price(age) when age >= 13 and age < 65 do
    15.00
  end

  # Guard: when age >= 65
  def ticket_price(age) when age >= 65 do
    10.00
  end
end

IO.puts(Pricing.ticket_price(10))   # 5.0  (child)
IO.puts(Pricing.ticket_price(30))   # 15.0 (adult)
IO.puts(Pricing.ticket_price(70))   # 10.0 (senior)
{{< /highlight >}}

Guards extend pattern matching with runtime checks. Common guard expressions include comparisons, type checks (`is_integer`, `is_list`), and simple functions.

### Combining Patterns and Guards

{{< highlight elixir >}}
defmodule OrderProcessor do
  # Match tuple pattern with guard
  def process({:order, items, total}) when total > 100 do
    "Large order: #{length(items)} items, total $#{total} - free shipping!"
  end

  def process({:order, items, total}) when total > 0 do
    "Order: #{length(items)} items, total $#{total}"
  end

  def process({:order, _items, total}) when total <= 0 do
    "Invalid order: total must be positive"
  end

  def process({:refund, amount}) when amount > 0 do
    "Processing refund of $#{amount}"
  end

  def process(_) do
    "Unknown operation"
  end
end

IO.puts(OrderProcessor.process({:order, ["item1", "item2"], 150.00}))
# Large order: 2 items, total $150.0 - free shipping!

IO.puts(OrderProcessor.process({:order, ["item1"], 50.00}))
# Order: 1 items, total $50.0

IO.puts(OrderProcessor.process({:refund, 25.00}))
# Processing refund of $25.0
{{< /highlight >}}

## Pattern Matching Replaces Polymorphism

In OOP, you'd use inheritance or interfaces to handle different types:

{{< highlight java >}}
// Java - using polymorphism
interface Shape {
    double area();
}

class Circle implements Shape {
    private double radius;

    public double area() {
        return Math.PI * radius * radius;
    }
}

class Rectangle implements Shape {
    private double width, height;

    public double area() {
        return width * height;
    }
}

// Client code
double calculateArea(Shape shape) {
    return shape.area();  // Polymorphic dispatch
}
{{< /highlight >}}

In Elixir, you use pattern matching with tagged tuples or structs:

{{< highlight elixir >}}
defmodule Geometry do
  def area({:circle, radius}) do
    3.14159 * radius * radius
  end

  def area({:rectangle, width, height}) do
    width * height
  end

  def area({:triangle, base, height}) do
    0.5 * base * height
  end
end

IO.puts(Geometry.area({:circle, 5}))           # 78.53975
IO.puts(Geometry.area({:rectangle, 4, 6}))     # 24
IO.puts(Geometry.area({:triangle, 10, 8}))     # 40.0
{{< /highlight >}}

**Key difference:** OOP uses runtime polymorphism with object types. FP uses compile-time pattern matching with data shapes. Both achieve the same goal - handling different cases - but FP does it with data and functions rather than classes and methods.

### Using Structs for Stronger Typing

For more complex scenarios, you can use structs:

{{< highlight elixir >}}
defmodule Circle do
  defstruct [:radius]
end

defmodule Rectangle do
  defstruct [:width, :height]
end

defmodule Geometry do
  def area(%Circle{radius: r}) do
    3.14159 * r * r
  end

  def area(%Rectangle{width: w, height: h}) do
    w * h
  end
end

circle = %Circle{radius: 5}
rectangle = %Rectangle{width: 4, height: 6}

IO.puts(Geometry.area(circle))      # 78.53975
IO.puts(Geometry.area(rectangle))   # 24
{{< /highlight >}}

Pattern matching on struct types provides similar benefits to OOP polymorphism while keeping everything explicit and functional.

## Case and With: Control Flow with Pattern Matching

### Case: Multiple Pattern Matches

{{< highlight elixir >}}
user_input = {:login, "alice", "secret123"}

result = case user_input do
  {:login, username, password} ->
    "Logging in #{username}..."

  {:register, username, email} ->
    "Registering #{username} with email #{email}"

  {:logout} ->
    "Logging out"

  _ ->
    "Unknown command"
end

IO.puts(result)  # Logging in alice...
{{< /highlight >}}

### With: Sequential Pattern Matching with Early Exit

The `with` statement is perfect for chaining operations that might fail:

{{< highlight elixir >}}
defmodule UserService do
  def fetch_user_profile(user_id) do
    with {:ok, user} <- fetch_user(user_id),
         {:ok, profile} <- fetch_profile(user.profile_id),
         {:ok, preferences} <- fetch_preferences(user.id) do
      {:ok, %{user: user, profile: profile, preferences: preferences}}
    else
      {:error, reason} -> {:error, "Failed to load user: #{reason}"}
    end
  end

  # Simulated functions
  defp fetch_user(1), do: {:ok, %{id: 1, profile_id: 100}}
  defp fetch_user(_), do: {:error, :not_found}

  defp fetch_profile(100), do: {:ok, %{name: "Alice", bio: "Developer"}}
  defp fetch_profile(_), do: {:error, :profile_missing}

  defp fetch_preferences(1), do: {:ok, %{theme: "dark", lang: "en"}}
  defp fetch_preferences(_), do: {:error, :prefs_missing}
end

case UserService.fetch_user_profile(1) do
  {:ok, data} ->
    IO.puts("Profile loaded: #{data.profile.name}")

  {:error, reason} ->
    IO.puts("Error: #{reason}")
end
{{< /highlight >}}

If any step fails, `with` short-circuits and jumps to the `else` clause. No nested if/else or try/catch needed.

**OOP Comparison:** You'd typically use exceptions or nested null checks:

{{< highlight java >}}
// Java - exception-based approach
try {
    User user = fetchUser(userId);
    Profile profile = fetchProfile(user.getProfileId());
    Preferences prefs = fetchPreferences(user.getId());
    return new UserProfile(user, profile, prefs);
} catch (NotFoundException e) {
    throw new RuntimeException("Failed to load user: " + e.getMessage());
}
{{< /highlight >}}

The `with` statement makes the happy path clear and handles errors explicitly without exception handling ceremony.

## Why Pattern Matching Matters

### 1. Declarative Data Handling

Pattern matching lets you declare what data structure you expect, and Elixir validates it automatically:

{{< highlight elixir >}}
defmodule API do
  def handle_response({:ok, %{status: 200, body: body}}) do
    {:success, body}
  end

  def handle_response({:ok, %{status: 404}}) do
    {:error, :not_found}
  end

  def handle_response({:ok, %{status: status}}) when status >= 500 do
    {:error, :server_error}
  end

  def handle_response({:error, reason}) do
    {:error, reason}
  end
end
{{< /highlight >}}

Each function clause is self-documenting - you can see exactly what data shape it handles.

### 2. Impossible States Become Impossible

Pattern matching catches mismatches at runtime, preventing subtle bugs:

{{< highlight elixir >}}
# This will fail if result isn't a 2-element tuple
{:ok, value} = fetch_data()

# This enforces that users must have both name and email
%{name: name, email: email} = user
{{< /highlight >}}

If the data doesn't match your expectations, you get an immediate, clear error instead of a null pointer or undefined value later.

### 3. Eliminates Type Checking Code

{{< highlight elixir >}}
# Instead of:
def process(input) do
  if is_list(input) do
    # handle list
  else if is_map(input) do
    # handle map
  else
    # handle other
  end
end

# Use pattern matching:
def process(input) when is_list(input) do
  # handle list
end

def process(input) when is_map(input) do
  # handle map
end

def process(_input) do
  # handle other
end
{{< /highlight >}}

The structure of your code matches the structure of your data.

## Key Takeaways

- **Pattern matching** uses `=` to match patterns and bind variables, not just assign values
- **Destructuring** extracts values from lists, tuples, and maps in one expression
- **Function clauses** with different patterns replace if/else chains and switch statements
- **Guards** add runtime conditions to patterns (`when age > 18`)
- **Pattern matching replaces OOP polymorphism** - use data shapes instead of class hierarchies
- **Case and with** provide control flow based on pattern matching
- **Benefits:**
  - Declarative, self-documenting code
  - Early error detection when data doesn't match expectations
  - Eliminates verbose type checking and null handling

## Try It Yourself

Open `iex` and practice pattern matching with these exercises.

### Exercise 1: Basic Destructuring

{{< highlight elixir >}}
# Given this data:
user = %{name: "Bob", age: 25, email: "bob@example.com", role: :user}

# Extract just name and email using pattern matching
# Your code here: %{name: ???, email: ???} = user
{{< /highlight >}}

### Exercise 2: Function Clauses

Write a `classify` function that uses pattern matching to classify numbers:
- Empty list → "No numbers"
- List with one item → "Single: [number]"
- List with multiple items → "Multiple: [count] numbers"

{{< highlight elixir >}}
defmodule Classifier do
  def classify([]) do
    # Your code here
  end

  def classify([single]) do
    # Your code here
  end

  def classify(list) do
    # Your code here
  end
end

# Test it:
# Classifier.classify([])
# Classifier.classify([42])
# Classifier.classify([1, 2, 3])
{{< /highlight >}}

### Exercise 3: Pattern Matching HTTP Responses

Create a function that handles different HTTP response patterns:

{{< highlight elixir >}}
defmodule HTTP do
  def handle({:ok, %{status: 200, body: body}}) do
    # Success case
  end

  def handle({:ok, %{status: 404}}) do
    # Not found case
  end

  def handle({:error, reason}) do
    # Error case
  end
end

# Test with:
# HTTP.handle({:ok, %{status: 200, body: "Hello"}})
# HTTP.handle({:ok, %{status: 404}})
# HTTP.handle({:error, :timeout})
{{< /highlight >}}

## Official Documentation to Help You Learn

- [Elixir Pattern Matching Guide](https://elixir-lang.org/getting-started/pattern-matching.html)
- [Elixir Case, Cond, and If](https://elixir-lang.org/getting-started/case-cond-and-if.html)
- [Guards Documentation](https://hexdocs.pm/elixir/guards.html)

---

*Part Four | Functional Programming Through Elixir series*

**Previous in series:** [Part Three - Pure Functions vs Side Effects](/posts/fp-series-pure-functions-vs-side-effects/)
