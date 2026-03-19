---
title: "Part Seven | Functional Programming Through Elixir: Higher-Order Functions"
date: 2026-03-13
draft: false
tags: ["elixir", "functional-programming", "fp-series", "higher-order-functions", "reduce", "composition", "learning"]
description: "Seventh post in a series exploring functional programming concepts through Elixir. Higher-order functions, reduce, function factories, and composition vs OOP design patterns."
---

## Beyond Passing Functions Around

In [Part Six](/posts/fp-series-the-pipe-operator/), we saw how the pipe operator turns nested function calls into readable pipelines. In [Part Two](/posts/fp-series-functions-as-first-class-citizens/), we learned that functions are values. You can pass them to `Enum.map` and `Enum.filter` to transform and select data.

Now we'll go deeper. A **higher-order function** is a function that does at least one of these things:

1. **Takes a function as an argument** (like `Enum.map`)
2. **Returns a function as its result**

You've already used higher-order functions when calling `Enum.map` and `Enum.filter`. This post covers the most important one you haven't seen yet, `reduce`, then gets into writing your own higher-order functions and composing functions together.

<!--more-->

## Reduce: The One That Does Everything

`Enum.map` transforms each element. `Enum.filter` selects elements. **`Enum.reduce`** can do both of those and more. It walks through a collection and builds up a single result, step by step.

{{< highlight elixir >}}
# Sum a list of numbers
Enum.reduce([1, 2, 3, 4, 5], 0, fn number, accumulator ->
  number + accumulator
end)
# 15
{{< /highlight >}}

Here's what happens at each step:

| Step | `number` | `accumulator` | Result |
|------|----------|---------------|--------|
| 1    | 1        | 0             | 1      |
| 2    | 2        | 1             | 3      |
| 3    | 3        | 3             | 6      |
| 4    | 4        | 6             | 10     |
| 5    | 5        | 10            | 15     |

The function takes each element and the running accumulator, and returns the new accumulator. The `0` is the initial accumulator value.

### What Else Can Reduce Do?

Anything you can express as "walk through a collection and build up a result" is a reduce:

{{< highlight elixir >}}
# Find the maximum value
Enum.reduce([5, 3, 8, 1, 9, 2], fn number, max ->
  if number > max, do: number, else: max
end)
# 9 (no initial value, so it uses the first element as the starting accumulator)

# Build a frequency map
words = ["apple", "banana", "apple", "cherry", "banana", "apple"]

Enum.reduce(words, %{}, fn word, counts ->
  Map.update(counts, word, 1, fn existing -> existing + 1 end)
end)
# %{"apple" => 3, "banana" => 2, "cherry" => 1}

# Separate even and odd numbers
Enum.reduce([1, 2, 3, 4, 5, 6], %{even: [], odd: []}, fn number, acc ->
  if rem(number, 2) == 0 do
    %{acc | even: [number | acc.even]}
  else
    %{acc | odd: [number | acc.odd]}
  end
end)
# %{even: [6, 4, 2], odd: [5, 3, 1]}
{{< /highlight >}}

The accumulator doesn't have to be a number. It can be a list, a map, a tuple, whatever you need.

### Map and Filter Are Just Reduce

`map` and `filter` are special cases of `reduce`. You can build both from it:

{{< highlight elixir >}}
# map as reduce
defmodule MyEnum do
  def map(list, func) do
    list
    |> Enum.reduce([], fn element, acc -> [func.(element) | acc] end)
    |> Enum.reverse()
  end

  def filter(list, func) do
    list
    |> Enum.reduce([], fn element, acc ->
      if func.(element), do: [element | acc], else: acc
    end)
    |> Enum.reverse()
  end
end

MyEnum.map([1, 2, 3], fn x -> x * 2 end)
# [2, 4, 6]

MyEnum.filter([1, 2, 3, 4, 5], fn x -> rem(x, 2) == 0 end)
# [2, 4]
{{< /highlight >}}

You won't write your own `map` and `filter` in practice since `Enum` already has them. But understanding that reduce is underneath helps you reach for it when `map` and `filter` aren't enough.

### Contrast: OOP's Iterator Patterns

In OOP, when you need custom iteration, you typically use the **Iterator pattern**, an object that tracks position and provides `next()` and `hasNext()` methods:

{{< highlight java >}}
// Java - External iteration with Iterator
Iterator<Integer> iterator = numbers.iterator();
int sum = 0;
while (iterator.hasNext()) {
    sum += iterator.next();
}

// Or with enhanced for-loop (still external iteration)
int sum = 0;
for (int number : numbers) {
    sum += number;
}
{{< /highlight >}}

{{< highlight python >}}
# Python - imperative accumulation
numbers = [1, 2, 3, 4, 5]
sum = 0
for number in numbers:
    sum += number
# 15

# Python does have reduce, but it's tucked away
from functools import reduce
sum = reduce(lambda acc, x: acc + x, numbers, 0)
{{< /highlight >}}

The difference here matters:

- **OOP (external iteration):** Your code controls the loop. You manage the accumulator variable, mutate it on each step, and decide when to stop.
- **FP (internal iteration):** `reduce` controls the loop. You provide the logic for combining elements. No mutable variables, no loop management.

In OOP, the **caller** manages iteration state. In FP, the **function** manages it. You just describe what to do at each step.

## Returning Functions: Function Factories

Part Two showed passing functions as arguments. But higher-order functions can also **return** functions. This lets you create specialized functions from a template.

{{< highlight elixir >}}
defmodule Tax do
  # Returns a function that applies a specific tax rate
  def calculator(rate) do
    fn price -> price * (1 + rate) end
  end
end

# Create specialized tax calculators
us_tax = Tax.calculator(0.08)
uk_vat = Tax.calculator(0.20)
no_tax = Tax.calculator(0.0)

us_tax.(100.00)   # 108.0
uk_vat.(100.00)   # 120.0
no_tax.(100.00)   # 100.0

# Use them in pipelines
[29.99, 49.99, 99.99]
|> Enum.map(us_tax)
# [32.3892, 53.9892, 107.9892]
{{< /highlight >}}

The `calculator/1` function **closes over** the `rate` value. The returned function remembers the rate it was created with. This is a **closure**.

### Validators as Function Factories

{{< highlight elixir >}}
defmodule Validators do
  # Returns a function that checks minimum length
  def min_length(n) do
    fn string -> String.length(string) >= n end
  end

  # Returns a function that checks if a value is in a range
  def in_range(min, max) do
    fn value -> value >= min and value <= max end
  end

  # Returns a function that checks a string matches a pattern
  def matches(pattern) do
    fn string -> String.match?(string, pattern) end
  end
end

# Create validators
valid_password? = Validators.min_length(8)
valid_age? = Validators.in_range(18, 120)
valid_email? = Validators.matches(~r/@/)

valid_password?.("secret")      # false (only 6 chars)
valid_password?.("supersecret") # true
valid_age?.(25)                 # true
valid_age?.(15)                 # false
valid_email?.("user@example")   # true

# Use them with filter
passwords = ["abc", "password123", "hi", "secure_enough"]
Enum.filter(passwords, valid_password?)
# ["password123", "secure_enough"]
{{< /highlight >}}

### Contrast: OOP's Strategy Pattern

In OOP, the same configurable behavior requires the **Strategy pattern** with an interface, concrete implementations, and a context class:

{{< highlight java >}}
// Java - Strategy pattern
interface TaxStrategy {
    double calculate(double price);
}

class USTax implements TaxStrategy {
    public double calculate(double price) {
        return price * 1.08;
    }
}

class UKTax implements TaxStrategy {
    public double calculate(double price) {
        return price * 1.20;
    }
}

// Usage
TaxStrategy strategy = new USTax();
double total = strategy.calculate(100.00);  // 108.0
{{< /highlight >}}

That's an interface, two classes, and instantiation for what Elixir does with a two-line function that returns a function. Objects need class definitions to carry behavior. Functions just carry it directly.

## Writing Your Own Higher-Order Functions

Any function that accepts a function as an argument is a higher-order function. You write them when you want the caller to customize behavior.

{{< highlight elixir >}}
defmodule OrderProcessor do
  # Higher-order function: caller provides the pricing strategy
  def calculate_total(items, pricing_fn) do
    Enum.reduce(items, 0.0, fn item, total ->
      total + pricing_fn.(item)
    end)
  end
end

items = [
  %{name: "Laptop", price: 999.99, quantity: 1},
  %{name: "Mouse", price: 25.00, quantity: 3},
  %{name: "Cable", price: 8.00, quantity: 5}
]

# Standard pricing: price * quantity
standard = OrderProcessor.calculate_total(items, fn item ->
  item.price * item.quantity
end)
# 1114.99

# Bulk discount: 10% off if quantity > 2
bulk = OrderProcessor.calculate_total(items, fn item ->
  if item.quantity > 2 do
    item.price * item.quantity * 0.90
  else
    item.price * item.quantity
  end
end)
# 1069.49
{{< /highlight >}}

`calculate_total/2` doesn't know or care how pricing works. The caller injects that logic. This is inversion of control without any framework or interface.

### Another Example: Retry Logic

{{< highlight elixir >}}
defmodule Resilient do
  # Higher-order function: retries any operation
  def retry(func, max_attempts \\ 3) do
    do_retry(func, 1, max_attempts)
  end

  defp do_retry(func, attempt, max_attempts) do
    case func.() do
      {:ok, result} ->
        {:ok, result}

      {:error, reason} when attempt < max_attempts ->
        IO.puts("Attempt #{attempt} failed: #{reason}. Retrying...")
        do_retry(func, attempt + 1, max_attempts)

      {:error, reason} ->
        {:error, "Failed after #{max_attempts} attempts: #{reason}"}
    end
  end
end

# Pass any fallible operation
Resilient.retry(fn -> fetch_from_api("https://example.com/data") end)

# With custom max attempts
Resilient.retry(fn -> send_email(user, template) end, 5)
{{< /highlight >}}

`retry/2` doesn't know what it's retrying. It just knows the protocol: call the function, check for `{:ok, _}` or `{:error, _}`. The caller decides what operation to retry.

**OOP Comparison:** In OOP, you'd likely create a `RetryPolicy` class, a `Retryable` interface, and wire them together with dependency injection. Here it's just a function that takes a function.

## Function Composition

So far we've been piping data through functions: `data |> f() |> g() |> h()`. But sometimes you want to create a **new function** by combining existing ones, without applying them to data yet.

{{< highlight elixir >}}
defmodule Compose do
  # Compose two functions into one
  def compose(f, g) do
    fn x -> g.(f.(x)) end
  end

  # Compose a list of functions into one
  def pipeline(functions) do
    Enum.reduce(functions, fn x -> x end, fn f, acc ->
      fn x -> f.(acc.(x)) end
    end)
  end
end

trim = &String.trim/1
downcase = &String.downcase/1
capitalize = &String.capitalize/1

# Compose into a single function
clean_name = Compose.pipeline([trim, downcase, capitalize])

clean_name.("  ALICE  ")   # "Alice"
clean_name.("  BOB  ")     # "Bob"

# Now use the composed function in a pipeline
["  ALICE  ", "  BOB  ", "  CHARLIE  "]
|> Enum.map(clean_name)
# ["Alice", "Bob", "Charlie"]
{{< /highlight >}}

The difference between piping and composition:

- **Piping** (`|>`): Transforms data right now. `"  ALICE  " |> String.trim() |> String.downcase()`
- **Composition**: Creates a new function for later. `clean_name = compose(trim, downcase)`

Composition is useful when you need to pass a transformation as a single function, like to `Enum.map`.

### Practical Composition: Building Validators

{{< highlight elixir >}}
defmodule Validate do
  # Compose multiple validators into one
  def all(validators) do
    fn value ->
      Enum.all?(validators, fn validator -> validator.(value) end)
    end
  end

  # Compose validators where ANY must pass
  def any(validators) do
    fn value ->
      Enum.any?(validators, fn validator -> validator.(value) end)
    end
  end
end

# Small, focused validators
not_empty? = fn s -> String.length(s) > 0 end
has_at? = fn s -> String.contains?(s, "@") end
has_dot? = fn s -> String.contains?(s, ".") end

# Compose them
valid_email? = Validate.all([not_empty?, has_at?, has_dot?])

valid_email?.("user@example.com")   # true
valid_email?.("invalid")            # false
valid_email?.("")                   # false

# Use the composed validator
emails = ["alice@test.com", "bad", "bob@example.org", ""]
Enum.filter(emails, valid_email?)
# ["alice@test.com", "bob@example.org"]
{{< /highlight >}}

Small validators, composed into a bigger one, used anywhere a function is expected.

### Contrast: OOP's Composition Patterns

In OOP, combining behavior like this usually requires design patterns:

{{< highlight java >}}
// Java - Decorator pattern for composing validators
interface Validator {
    boolean validate(String value);
}

class NotEmptyValidator implements Validator {
    public boolean validate(String value) {
        return !value.isEmpty();
    }
}

class ContainsValidator implements Validator {
    private String required;

    public ContainsValidator(String required) {
        this.required = required;
    }

    public boolean validate(String value) {
        return value.contains(required);
    }
}

class CompositeValidator implements Validator {
    private List<Validator> validators;

    public CompositeValidator(List<Validator> validators) {
        this.validators = validators;
    }

    public boolean validate(String value) {
        return validators.stream().allMatch(v -> v.validate(value));
    }
}

// Usage
Validator emailValidator = new CompositeValidator(List.of(
    new NotEmptyValidator(),
    new ContainsValidator("@"),
    new ContainsValidator(".")
));

emailValidator.validate("user@example.com");  // true
{{< /highlight >}}

Three classes and an interface to do what Elixir does with a few functions. The OOP version isn't wrong, it's well-structured. But it's more machinery for the same result. With higher-order functions, composition is cheap enough that you just do it instead of building out a pattern.

## Key Takeaways

- **Higher-order functions** take functions as arguments or return functions as results
- **`Enum.reduce`** walks a collection and builds up any result. `map` and `filter` are special cases of reduce
- **Returning functions** (closures) lets you create specialized behavior from a template, replacing OOP's Strategy pattern
- **Writing your own HOFs** gives you inversion of control without interfaces or dependency injection
- **Function composition** creates new functions from existing ones, replacing OOP's Decorator and Composite patterns
- **OOP uses external iteration** (caller manages the loop) while **FP uses internal iteration** (the HOF manages the loop)

## Try It Yourself

Open `iex` and practice these higher-order function exercises.

### Exercise 1: Reduce

Use `Enum.reduce` to implement a function that takes a list of strings and returns the longest one:

{{< highlight elixir >}}
words = ["cat", "elephant", "dog", "hippopotamus", "ant"]

longest = Enum.reduce(words, ???, fn word, longest ->
  ???
end)
# Should return "hippopotamus"
{{< /highlight >}}

### Exercise 2: Function Factory

Write a function `multiplier/1` that takes a number and returns a function that multiplies its argument by that number:

{{< highlight elixir >}}
defmodule Math do
  def multiplier(factor) do
    ???
  end
end

double = Math.multiplier(2)
triple = Math.multiplier(3)

double.(5)   # 10
triple.(5)   # 15

# Bonus: use it with Enum.map
[1, 2, 3, 4, 5] |> Enum.map(Math.multiplier(10))
# [10, 20, 30, 40, 50]
{{< /highlight >}}

### Exercise 3: Compose Your Own HOF

Write a `transform_if/3` function that takes a list, a predicate function, and a transform function. It should apply the transform only to elements where the predicate returns true, leaving others unchanged:

{{< highlight elixir >}}
defmodule ListUtils do
  def transform_if(list, predicate, transform) do
    ???
  end
end

# Double only the even numbers
ListUtils.transform_if(
  [1, 2, 3, 4, 5],
  fn x -> rem(x, 2) == 0 end,
  fn x -> x * 2 end
)
# [1, 4, 3, 8, 5]

# Upcase only short strings
ListUtils.transform_if(
  ["hi", "hello", "hey", "greetings"],
  fn s -> String.length(s) <= 3 end,
  &String.upcase/1
)
# ["HI", "hello", "HEY", "greetings"]
{{< /highlight >}}

## Official Documentation to Help You Learn

- [Enum Module Documentation](https://hexdocs.pm/elixir/Enum.html)
- [Enum.reduce/3 Documentation](https://hexdocs.pm/elixir/Enum.html#reduce/3)
- [Elixir Getting Started: Enumerables](https://elixir-lang.org/getting-started/enumerables-and-streams.html)

---

*Part Seven | Functional Programming Through Elixir series*

**Previous in series:** [Part Six - The Pipe Operator](/posts/fp-series-the-pipe-operator/)

**Next in series:** [Part Eight - Guards and Pattern Matching in Function Heads](/posts/fp-series-guards-and-pattern-matching-in-function-heads/)
