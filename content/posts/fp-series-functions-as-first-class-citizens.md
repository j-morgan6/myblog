---
title: "Part Two | Functional Programming Through Elixir: Functions as First-Class Citizens"
date: 2026-02-04
draft: True
tags: ["elixir", "functional-programming", "fp-series", "first-class-functions", "learning"]
description: "Second post in a series exploring functional programming concepts through Elixir. Discover how treating functions as values unlocks powerful patterns for data transformation and code composition."
---

## Functions Are Values

In the [first post](/posts/fp-series-immutability-vs-mutable-state/), we explored how immutability changes the way we think about data. Now we'll tackle another fundamental shift: **in functional programming, functions are values just like numbers, strings, or lists**. You can pass them as arguments, return them from other functions, store them in variables, and put them in data structures.

If you're coming from object-oriented programming, you've probably encountered this concept through callbacks, lambdas, or method references. But in OOP, these often feel like workarounds or special cases. In functional programming languages like Elixir, treating functions as first-class citizens is natural and central to how you write code.

<!--more-->

## What Does "First-Class Citizen" Mean?

When we say functions are **first-class citizens**, we mean they have the same privileges as any other value in the language. Specifically, functions can be:

- **Assigned to variables**
- **Passed as arguments to other functions**
- **Returned from functions**
- **Stored in data structures** (like lists or maps)

Let's see this in action with Elixir:

{{< highlight elixir >}}
# Assign a function to a variable
add = fn a, b -> a + b end

# Call it like any other function
result = add.(5, 3)
IO.puts(result)  # 8

# Store functions in a list
operations = [
  fn x -> x + 1 end,
  fn x -> x * 2 end,
  fn x -> x - 3 end
]

# Use a function from the list
first_op = Enum.at(operations, 0)
IO.puts(first_op.(10))  # 11
{{< /highlight >}}

**Elixir Syntax Note:** When calling an anonymous function stored in a variable, you use the dot notation: `add.(5, 3)`. This distinguishes anonymous function calls from named function calls.

## Contrast: Functions in OOP

In object-oriented languages, functions (methods) are typically bound to classes or objects. While modern languages have added support for lambdas and function references, they often feel more verbose or constrained:

{{< highlight java >}}
// Java - Before Java 8, you needed verbose anonymous classes
Comparator<String> comparator = new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return a.length() - b.length();
    }
};

// Java 8+ - Better with lambdas, but still more ceremony
Comparator<String> lambdaComparator = (a, b) -> a.length() - b.length();

// You can't just assign a method to a variable as easily
// Methods are tied to their class context
{{< /highlight >}}

{{< highlight python >}}
# Python is more flexible, functions are first-class
def add(a, b):
    return a + b

# Can assign to variable
operation = add
result = operation(5, 3)  # 8

# But methods are bound to instances
class Calculator:
    def add(self, a, b):
        return a + b

calc = Calculator()
# Method carries its instance context
operation = calc.add  # Still tied to 'calc' instance
{{< /highlight >}}

Elixir's approach is cleaner because functions aren't tied to objects or classes - they're just functions.

## Anonymous Functions

Anonymous functions (also called lambdas) are functions without names. In Elixir, you create them with the `fn` keyword:

{{< highlight elixir >}}
# Anonymous function with one parameter
square = fn x -> x * x end
IO.puts(square.(5))  # 25

# Multiple parameters
multiply = fn a, b -> a * b end
IO.puts(multiply.(4, 7))  # 28

# Pattern matching in parameters!
greet = fn
  {:formal, name} -> "Good day, #{name}"
  {:casual, name} -> "Hey #{name}!"
  {:friendly, name} -> "Hi #{name}, great to see you!"
end

IO.puts(greet.({:formal, "Dr. Smith"}))   # Good day, Dr. Smith
IO.puts(greet.({:casual, "Alex"}))        # Hey Alex!
IO.puts(greet.({:friendly, "Jordan"}))    # Hi Jordan, great to see you!
{{< /highlight >}}

### The Capture Operator Shorthand

For simple anonymous functions, Elixir provides a shorthand using the capture operator `&`:

{{< highlight elixir >}}
# These are equivalent
double_long = fn x -> x * 2 end
double_short = &(&1 * 2)

IO.puts(double_long.(5))   # 10
IO.puts(double_short.(5))  # 10

# &1 refers to the first argument, &2 to the second, etc.
add_short = &(&1 + &2)
IO.puts(add_short.(3, 7))  # 10

# You can also capture named functions
# This creates an anonymous function that calls String.upcase/1
upcase_fn = &String.upcase/1
IO.puts(upcase_fn.("hello"))  # HELLO
{{< /highlight >}}

The `&(&1 * 2)` syntax might look strange at first, but it's a concise way to create simple transformations. You'll see this pattern frequently in Elixir code.

## Passing Functions as Arguments

Here's where first-class functions become truly powerful. You can pass functions to other functions, enabling highly reusable and composable code.

Let's start with a practical scenario: you have a list of products and need to transform or filter them.

{{< highlight elixir >}}
products = [
  %{name: "Laptop", price: 999.99, category: "electronics"},
  %{name: "Coffee Mug", price: 12.50, category: "home"},
  %{name: "Headphones", price: 149.99, category: "electronics"},
  %{name: "Desk Lamp", price: 45.00, category: "home"},
  %{name: "Keyboard", price: 89.99, category: "electronics"}
]
{{< /highlight >}}

### Using Enum.map: Transform Every Item

`Enum.map` takes a collection and a function, applies that function to each item, and returns a new list with the results:

{{< highlight elixir >}}
# Extract just the prices
prices = Enum.map(products, fn product -> product.price end)
IO.inspect(prices)  # [999.99, 12.5, 149.99, 45.0, 89.99]

# Using the capture operator shorthand
prices_short = Enum.map(products, &(&1.price))
IO.inspect(prices_short)  # [999.99, 12.5, 149.99, 45.0, 89.99]

# Apply a 20% discount to all prices
discounted = Enum.map(products, fn product ->
  %{product | price: product.price * 0.80}
end)

IO.inspect(discounted)
# All products now have price * 0.80
{{< /highlight >}}

### Using Enum.filter: Select Specific Items

`Enum.filter` takes a collection and a predicate function (returns true/false), keeping only items where the function returns `true`:

{{< highlight elixir >}}
# Find all electronics
electronics = Enum.filter(products, fn product ->
  product.category == "electronics"
end)

IO.inspect(electronics)
# [%{name: "Laptop", ...}, %{name: "Headphones", ...}, %{name: "Keyboard", ...}]

# Find products under $50
affordable = Enum.filter(products, fn product -> product.price < 50 end)
IO.inspect(affordable)
# [%{name: "Coffee Mug", ...}, %{name: "Desk Lamp", ...}]

# Using capture operator
affordable_short = Enum.filter(products, &(&1.price < 50))
{{< /highlight >}}

### Composing Operations

The real power emerges when you combine these operations:

{{< highlight elixir >}}
# Find affordable electronics
affordable_electronics =
  products
  |> Enum.filter(fn p -> p.category == "electronics" end)
  |> Enum.filter(fn p -> p.price < 200 end)

IO.inspect(affordable_electronics)
# [%{name: "Headphones", ...}, %{name: "Keyboard", ...}]

# Get names of affordable electronics
affordable_electronics_names =
  products
  |> Enum.filter(&(&1.category == "electronics"))
  |> Enum.filter(&(&1.price < 200))
  |> Enum.map(&(&1.name))

IO.inspect(affordable_electronics_names)
# ["Headphones", "Keyboard"]
{{< /highlight >}}

**Elixir Syntax Note:** The `|>` pipe operator takes the result from the left and passes it as the first argument to the function on the right. We'll explore this deeply in an upcoming post, but it's the key to readable data transformations.

### Contrast: The OOP Way

In object-oriented languages, you'd typically use loops and conditionals:

{{< highlight javascript >}}
// JavaScript - Imperative approach
let affordableElectronics = [];
for (let product of products) {
  if (product.category === "electronics" && product.price < 200) {
    affordableElectronics.push(product.name);
  }
}

// JavaScript - Functional approach (modern JS supports this!)
let affordableElectronicsNames = products
  .filter(p => p.category === "electronics")
  .filter(p => p.price < 200)
  .map(p => p.name);
{{< /highlight >}}

Modern JavaScript supports functional patterns too! But notice how the imperative version with loops requires you to manually manage the accumulator array and mixing filtering logic. The functional version separates concerns - each step does one thing.

## Passing Named Functions

You're not limited to anonymous functions. You can pass named functions too:

{{< highlight elixir >}}
defmodule StringHelpers do
  def shout(text), do: String.upcase(text) <> "!"
  def whisper(text), do: String.downcase(text) <> "..."
  def capitalize_words(text), do: text |> String.split() |> Enum.map(&String.capitalize/1) |> Enum.join(" ")
end

messages = ["hello world", "GOOD MORNING", "how are you"]

# Pass named functions using capture syntax
shouted = Enum.map(messages, &StringHelpers.shout/1)
IO.inspect(shouted)
# ["HELLO WORLD!", "GOOD MORNING!", "HOW ARE YOU!"]

whispered = Enum.map(messages, &StringHelpers.whisper/1)
IO.inspect(whispered)
# ["hello world...", "good morning...", "how are you..."]

capitalized = Enum.map(messages, &StringHelpers.capitalize_words/1)
IO.inspect(capitalized)
# ["Hello World", "Good Morning", "How Are You"]
{{< /highlight >}}

The `&ModuleName.function_name/arity` syntax captures a reference to a named function, turning it into an anonymous function you can pass around.

## Why This Matters: Building with Small Functions

First-class functions encourage you to write **small, focused, reusable functions** that can be combined in different ways. Instead of large methods that do everything, you compose behavior from simple building blocks:

{{< highlight elixir >}}
defmodule ProductPipeline do
  # Small, focused functions
  def is_electronics?(product), do: product.category == "electronics"
  def is_affordable?(product), do: product.price < 100
  def apply_discount(product, rate), do: %{product | price: product.price * (1 - rate)}
  def format_display(product), do: "#{product.name}: $#{Float.round(product.price, 2)}"

  # Compose them together
  def get_discounted_affordable_electronics(products, discount_rate) do
    products
    |> Enum.filter(&is_electronics?/1)
    |> Enum.filter(&is_affordable?/1)
    |> Enum.map(&apply_discount(&1, discount_rate))
    |> Enum.map(&format_display/1)
  end
end

products = [
  %{name: "Laptop", price: 999.99, category: "electronics"},
  %{name: "Mouse", price: 25.00, category: "electronics"},
  %{name: "USB Cable", price: 8.00, category: "electronics"},
  %{name: "Coffee Mug", price: 12.50, category: "home"}
]

result = ProductPipeline.get_discounted_affordable_electronics(products, 0.10)
IO.inspect(result)
# ["Mouse: $22.5", "USB Cable: $7.2"]
{{< /highlight >}}

Each function does one thing and does it well. You can test each independently, reuse them in different combinations, and reason about each piece in isolation.

**OOP Comparison:** In OOP, you might use the Strategy pattern with multiple classes and interfaces, or create a complex class with many methods. The functional approach is simpler - just functions.

## Key Takeaways

- **First-class functions** means functions are values that can be assigned, passed, and stored just like any other data
- **Anonymous functions** (`fn -> end`) and the **capture operator** (`&`) make creating functions lightweight and convenient
- **Passing functions to Enum** (`map`, `filter`) enables declarative, composable data transformations without loops
- You can pass both **anonymous and named functions** using the capture syntax `&ModuleName.function/arity`
- This approach encourages **small, focused, reusable functions** that compose together rather than large, monolithic methods
- **Compared to OOP:** Less ceremony than anonymous classes, more flexible than method chaining, cleaner than verbose loops and conditionals

## Try It Yourself

Open up `iex` and work through this exercise to practice using functions as first-class citizens.

### Exercise 1: Filter and Transform

Create a list of users with name, age, and role:

{{< highlight elixir >}}
users = [
  %{name: "Alice", age: 28, role: "admin"},
  %{name: "Bob", age: 34, role: "user"},
  %{name: "Charlie", age: 23, role: "user"},
  %{name: "Diana", age: 45, role: "admin"},
  %{name: "Eve", age: 29, role: "moderator"}
]
{{< /highlight >}}

**Your tasks:**
1. Use `Enum.filter` to get all users with the "admin" role
2. Use `Enum.map` to extract just the names
3. Use `Enum.filter` to find users over 30
4. Combine operations: get names of admins over 30

### Exercise 2: Building a Pipeline

Create three small functions:
1. `double(n)` - doubles a number
2. `add_ten(n)` - adds 10 to a number
3. `is_even?(n)` - returns true if number is even

Use them with `Enum.map` and `Enum.filter` to:
1. Start with `[1, 2, 3, 4, 5]`
2. Double each number
3. Add 10 to each
4. Filter to keep only even results

What's your final list?

## Official Documentation to Help You Learn

- [Elixir Anonymous Functions Guide](https://elixir-lang.org/getting-started/modules-and-functions.html#anonymous-functions)
- [Elixir Enum Module Documentation](https://hexdocs.pm/elixir/Enum.html)
- [Elixir Function Capturing](https://elixir-lang.org/getting-started/modules-and-functions.html#function-capturing)

---

*Part Two | Functional Programming Through Elixir series*

**Next in series:** Part Three - Pure Functions vs Side Effects
