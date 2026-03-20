---
title: "Part Eight | Functional Programming Through Elixir: Guards and Pattern Matching in Function Heads"
date: 2026-03-19
draft: false
tags: ["elixir", "functional-programming", "fp-series", "guards", "pattern-matching", "learning"]
description: "Eighth post in a series exploring functional programming concepts through Elixir. Deep dive into guard clauses, custom guards with defguard, the pin operator, and replacing if/else chains with function heads."
---

## Taking Pattern Matching Further

In [Part Four](/posts/fp-series-pattern-matching/), we introduced pattern matching and saw basic guards in function clauses. We wrote things like `when age < 13` and moved on. This post picks up where that left off.

Guards and multi-clause functions are how you structure decision-making in Elixir. Where OOP reaches for if/else chains, switch statements, or the strategy pattern, Elixir uses function heads with guards. Once you're comfortable with this approach, you'll find that most conditional logic just becomes more functions.

<!--more-->

## What Goes in a Guard

Not everything is allowed in guard expressions. Elixir restricts guards to a specific set of operations that are guaranteed to be side-effect free and fast. This matters because the runtime evaluates guards during function dispatch.

### Allowed in Guards

{{< highlight elixir >}}
# Comparison operators
def classify(n) when n > 0, do: :positive
def classify(n) when n < 0, do: :negative
def classify(0), do: :zero

# Type checks
def process(val) when is_binary(val), do: "It's a string: #{val}"
def process(val) when is_integer(val), do: "It's an integer: #{val}"
def process(val) when is_list(val), do: "It's a list with #{length(val)} elements"
def process(val) when is_map(val), do: "It's a map"
def process(val) when is_atom(val), do: "It's an atom: #{val}"

# Boolean operators: and, or, not (not &&, ||, !)
def access?(role, active?) when role == :admin and active?, do: true
def access?(_, _), do: false

# Arithmetic
def shipping_tier(weight) when weight * 2 > 100, do: :heavy
def shipping_tier(_), do: :standard

# A few built-in functions
def first_letter(name) when byte_size(name) > 0, do: String.first(name)
def first_letter(_), do: nil
{{< /highlight >}}

Here's the full list of what's allowed:

| Category | Examples |
|----------|---------|
| Comparison | `==`, `!=`, `<`, `>`, `<=`, `>=`, `===`, `!==` |
| Boolean | `and`, `or`, `not` |
| Arithmetic | `+`, `-`, `*`, `/` |
| Type checks | `is_atom/1`, `is_binary/1`, `is_integer/1`, `is_float/1`, `is_list/1`, `is_map/1`, `is_tuple/1`, `is_nil/1`, `is_boolean/1`, `is_number/1` |
| Built-in functions | `abs/1`, `byte_size/1`, `div/2`, `elem/2`, `hd/1`, `length/1`, `map_size/1`, `rem/2`, `round/1`, `tl/1`, `trunc/1`, `tuple_size/1` |
| Membership | `in/2` (for compile-time lists and ranges) |

### Not Allowed in Guards

{{< highlight elixir >}}
# Custom functions - WILL NOT COMPILE
def process(val) when my_custom_check(val), do: :ok

# String functions
def process(name) when String.starts_with?(name, "A"), do: :a_name
# ** (CompileError) cannot invoke String.starts_with?/2 inside a guard

# Enum functions
def process(list) when Enum.count(list) > 5, do: :long_list
# ** (CompileError) cannot invoke Enum.count/1 inside a guard
{{< /highlight >}}

This restriction is intentional. Guards run in a special evaluation context where failures are treated as "this clause doesn't match" rather than raising errors. Allowing arbitrary functions would make that guarantee impossible.

### Guard Failures Are Quiet

When a guard expression raises an error, Elixir doesn't crash. It treats the clause as non-matching and tries the next one:

{{< highlight elixir >}}
defmodule Safe do
  def check(val) when length(val) > 3, do: "long list"
  def check(val) when is_binary(val), do: "a string"
  def check(_), do: "something else"
end

# length/1 fails on non-lists, but the guard just moves to the next clause
Safe.check("hello")  # "a string" (not a crash)
Safe.check([1, 2, 3, 4])  # "long list"
Safe.check(42)  # "something else"
{{< /highlight >}}

`length("hello")` would normally raise an error. Inside a guard, it causes that clause to be skipped. This is useful but can also hide bugs if you're not careful about clause ordering.

## The Pin Operator in Patterns

In [Part Four](/posts/fp-series-pattern-matching/) we saw that `=` matches patterns and binds variables. But what if you want to match against an existing variable's value instead of rebinding it? That's what the pin operator `^` does.

{{< highlight elixir >}}
expected_status = :active

# Without pin - this rebinds `expected_status` to whatever value is there
{expected_status, user} = {:banned, "Alice"}
IO.puts(expected_status)  # :banned (rebound!)

# With pin - this matches against the current value of `expected_status`
expected_status = :active
{^expected_status, user} = {:active, "Alice"}
IO.puts(user)  # "Alice" (match succeeded)

{^expected_status, user} = {:banned, "Bob"}
# ** (MatchError) - :banned doesn't match :active
{{< /highlight >}}

### Pin in Function Heads

The pin operator works in function clauses too. This is useful when a function needs to match against a value passed as another argument or captured in a closure:

{{< highlight elixir >}}
defmodule Permissions do
  def check(user_role, required_role) when user_role == required_role do
    :allowed
  end

  def check(_, _), do: :denied
end

Permissions.check(:admin, :admin)  # :allowed
Permissions.check(:user, :admin)   # :denied
{{< /highlight >}}

That works, but you can also write it with pin:

{{< highlight elixir >}}
defmodule Permissions do
  def check(role, role), do: :allowed
  def check(_, _), do: :denied
end

Permissions.check(:admin, :admin)  # :allowed
Permissions.check(:user, :admin)   # :denied
{{< /highlight >}}

When the same variable name appears twice in a pattern, Elixir requires both positions to have the same value. This is even cleaner than using `^` in many cases.

## Custom Guards with defguard

When you find yourself repeating the same guard conditions, you can extract them with `defguard`:

{{< highlight elixir >}}
defmodule Checks do
  defguard is_positive(n) when is_number(n) and n > 0
  defguard is_adult(age) when is_integer(age) and age >= 18
  defguard is_valid_score(s) when is_number(s) and s >= 0 and s <= 100
end
{{< /highlight >}}

Now use them anywhere you'd use a built-in guard:

{{< highlight elixir >}}
defmodule Account do
  import Checks

  def deposit(amount) when is_positive(amount) do
    {:ok, "Deposited #{amount}"}
  end

  def deposit(_), do: {:error, "Amount must be a positive number"}

  def create_user(name, age) when is_adult(age) do
    {:ok, %{name: name, age: age}}
  end

  def create_user(_, age) when is_integer(age) do
    {:error, "Must be 18 or older"}
  end

  def create_user(_, _), do: {:error, "Age must be an integer"}
end

Account.deposit(50)     # {:ok, "Deposited 50"}
Account.deposit(-10)    # {:error, "Amount must be a positive number"}
Account.deposit("ten")  # {:error, "Amount must be a positive number"}

Account.create_user("Alice", 25)  # {:ok, %{name: "Alice", age: 25}}
Account.create_user("Bob", 15)    # {:error, "Must be 18 or older"}
{{< /highlight >}}

`defguard` macros are expanded at compile time, so they follow the same restrictions as inline guards. You can only use the allowed operations inside them.

Use `defguardp` for guards that should stay private to the module.

## Replacing if/else with Function Heads

This is the shift in thinking. In OOP, conditional logic lives inside a function body. In Elixir, it moves to the function heads.

### The OOP Way

{{< highlight java >}}
// Java
public String describeTemperature(int temp) {
    if (temp <= 0) {
        return "Freezing";
    } else if (temp <= 15) {
        return "Cold";
    } else if (temp <= 25) {
        return "Comfortable";
    } else if (temp <= 35) {
        return "Hot";
    } else {
        return "Extreme heat";
    }
}
{{< /highlight >}}

{{< highlight python >}}
# Python
def describe_temperature(temp):
    if temp <= 0:
        return "Freezing"
    elif temp <= 15:
        return "Cold"
    elif temp <= 25:
        return "Comfortable"
    elif temp <= 35:
        return "Hot"
    else:
        return "Extreme heat"
{{< /highlight >}}

### The Elixir Way

{{< highlight elixir >}}
defmodule Weather do
  def describe(temp) when temp <= 0, do: "Freezing"
  def describe(temp) when temp <= 15, do: "Cold"
  def describe(temp) when temp <= 25, do: "Comfortable"
  def describe(temp) when temp <= 35, do: "Hot"
  def describe(_temp), do: "Extreme heat"
end

Weather.describe(-5)   # "Freezing"
Weather.describe(20)   # "Comfortable"
Weather.describe(40)   # "Extreme heat"
{{< /highlight >}}

Each clause is independent and self-contained. You can read any single clause and understand exactly what it handles without scanning through the rest.

### A More Realistic Example

{{< highlight elixir >}}
defmodule Subscription do
  # Free tier - no payment info needed
  def features(:free, _user), do: [:basic_search, :public_profiles]

  # Pro tier - active users get full features
  def features(:pro, %{status: :active}) do
    [:basic_search, :public_profiles, :advanced_filters, :export, :api_access]
  end

  # Pro tier - expired users get downgraded
  def features(:pro, %{status: :expired}) do
    features(:free, nil)
  end

  # Enterprise - needs active status and a company
  def features(:enterprise, %{status: :active, company: company}) when is_binary(company) do
    [:basic_search, :public_profiles, :advanced_filters, :export,
     :api_access, :sso, :audit_log, :custom_branding]
  end

  # Catch-all
  def features(_, _), do: {:error, :unknown_plan}
end

Subscription.features(:free, nil)
# [:basic_search, :public_profiles]

Subscription.features(:pro, %{status: :active})
# [:basic_search, :public_profiles, :advanced_filters, :export, :api_access]

Subscription.features(:pro, %{status: :expired})
# [:basic_search, :public_profiles]

Subscription.features(:enterprise, %{status: :active, company: "Acme"})
# [:basic_search, :public_profiles, :advanced_filters, :export,
#  :api_access, :sso, :audit_log, :custom_branding]
{{< /highlight >}}

In OOP, this kind of logic often ends up in a method with nested conditionals checking the plan type, then the user status, then additional fields. Here, each combination is a distinct function clause. Adding a new plan means adding new clauses, not modifying existing ones.

## Replacing the Strategy Pattern

In [Part Seven](/posts/fp-series-higher-order-functions/) we saw how function factories replace the strategy pattern. Guards give you another angle on the same idea: instead of passing in a strategy, you dispatch on data.

### OOP Strategy Pattern

{{< highlight java >}}
// Java
interface DiscountStrategy {
    double apply(double price, int quantity);
}

class NoDiscount implements DiscountStrategy {
    public double apply(double price, int quantity) {
        return price * quantity;
    }
}

class BulkDiscount implements DiscountStrategy {
    public double apply(double price, int quantity) {
        if (quantity >= 100) return price * quantity * 0.80;
        if (quantity >= 50) return price * quantity * 0.90;
        return price * quantity;
    }
}

class SeasonalDiscount implements DiscountStrategy {
    public double apply(double price, int quantity) {
        return price * quantity * 0.85;
    }
}

// Usage
DiscountStrategy strategy = new BulkDiscount();
double total = strategy.apply(10.00, 75);  // 675.0
{{< /highlight >}}

### Elixir: Function Heads with Guards

{{< highlight elixir >}}
defmodule Pricing do
  def total(price, quantity, :no_discount) do
    price * quantity
  end

  def total(price, quantity, :bulk) when quantity >= 100 do
    price * quantity * 0.80
  end

  def total(price, quantity, :bulk) when quantity >= 50 do
    price * quantity * 0.90
  end

  def total(price, quantity, :bulk) do
    price * quantity
  end

  def total(price, quantity, :seasonal) do
    price * quantity * 0.85
  end
end

Pricing.total(10.00, 75, :bulk)       # 675.0
Pricing.total(10.00, 100, :bulk)      # 800.0
Pricing.total(10.00, 30, :seasonal)   # 255.0
Pricing.total(10.00, 30, :no_discount) # 300.0
{{< /highlight >}}

No interface. No classes. No instantiation. The "strategy" is an atom, and the dispatch happens through pattern matching and guards on the function heads. If you need a new discount type, you add clauses.

The function factory approach from Part Seven and the guard-based approach here aren't mutually exclusive. Use guards when the behavior branches on data you already have. Use function factories when you need to build and pass around a reusable strategy.

## Combining Pattern Matching and Guards

The real power shows up when you combine structural pattern matching with guard conditions:

{{< highlight elixir >}}
defmodule EventHandler do
  # Keyboard events - only care about specific keys
  def handle(%{type: :keydown, key: key}) when key in [:enter, :escape, :tab] do
    "Special key: #{key}"
  end

  def handle(%{type: :keydown, key: key}) when is_atom(key) do
    "Regular key: #{key}"
  end

  # Mouse events - check bounds
  def handle(%{type: :click, x: x, y: y}) when x >= 0 and y >= 0 do
    "Click at (#{x}, #{y})"
  end

  # Timer events - only process if interval is reasonable
  def handle(%{type: :tick, interval: ms}) when ms > 0 and ms <= 60_000 do
    "Timer tick every #{ms}ms"
  end

  # Catch-all
  def handle(event) do
    "Unhandled event: #{inspect(event)}"
  end
end

EventHandler.handle(%{type: :keydown, key: :enter})
# "Special key: enter"

EventHandler.handle(%{type: :keydown, key: :a})
# "Regular key: a"

EventHandler.handle(%{type: :click, x: 100, y: 200})
# "Click at (100, 200)"

EventHandler.handle(%{type: :tick, interval: 1000})
# "Timer tick every 1000ms"
{{< /highlight >}}

The pattern match destructures the event and extracts fields. The guard adds constraints on those fields. Together they give you precise, readable dispatch without a single `if` statement.

## Clause Ordering Matters

Elixir tries clauses top to bottom and uses the first match. Getting the order wrong gives you unexpected results or compiler warnings:

{{< highlight elixir >}}
defmodule BadOrder do
  # This catches everything - clauses below never run
  def classify(_n), do: :other
  def classify(0), do: :zero
  def classify(n) when n > 0, do: :positive
end

# Elixir warns: "this clause cannot match because a previous clause always matches"
{{< /highlight >}}

The fix is to put more specific clauses first:

{{< highlight elixir >}}
defmodule GoodOrder do
  def classify(0), do: :zero
  def classify(n) when n > 0, do: :positive
  def classify(_n), do: :negative
end

GoodOrder.classify(0)    # :zero
GoodOrder.classify(5)    # :positive
GoodOrder.classify(-3)   # :negative
{{< /highlight >}}

Think of it like a funnel. Specific cases at the top, general catch-all at the bottom.

## Key Takeaways

- **Guards** are restricted to side-effect-free expressions: comparisons, type checks, arithmetic, and a small set of built-in functions
- **Guard failures don't crash.** A failing guard expression causes Elixir to skip that clause and try the next one
- **The pin operator** (`^`) matches against an existing variable's value instead of rebinding it
- **defguard** lets you extract repeated guard logic into reusable, named guards
- **Function heads replace if/else.** Conditional logic moves from inside the function body to the function signatures
- **Pattern matching + guards replaces the strategy pattern.** Dispatch on data shape and value instead of instantiating strategy objects
- **Clause ordering matters.** Specific clauses go first, catch-all goes last

## Try It Yourself

### Exercise 1: Custom Guard

Write a `defguard` called `is_valid_age` that checks if a value is an integer between 0 and 150. Then use it in a function:

{{< highlight elixir >}}
defmodule People do
  defguard is_valid_age(age) when ???

  def register(name, age) when is_valid_age(age) do
    {:ok, %{name: name, age: age}}
  end

  def register(_, _), do: {:error, "Invalid age"}
end

# People.register("Alice", 25)  # {:ok, %{name: "Alice", age: 25}}
# People.register("Bob", -5)    # {:error, "Invalid age"}
# People.register("Eve", "old") # {:error, "Invalid age"}
{{< /highlight >}}

### Exercise 2: Multi-Clause Dispatch

Write a `format_amount/2` function that formats a number differently based on a currency atom. Use pattern matching on the atom and guards on the amount:

{{< highlight elixir >}}
defmodule Currency do
  # Negative amounts should return an error tuple for all currencies
  # :usd -> "$10.00"
  # :eur -> "€10.00"
  # :jpy -> "¥10" (no decimals for yen)
  def format_amount(amount, currency) do
    ???
  end
end

# Currency.format_amount(10, :usd)   # "$10.00"
# Currency.format_amount(10, :eur)   # "€10.00"
# Currency.format_amount(10, :jpy)   # "¥10"
# Currency.format_amount(-5, :usd)   # {:error, "Negative amount"}
{{< /highlight >}}

Hint: you'll need at least 4 clauses. Put the negative amount check first since it applies to all currencies.

### Exercise 3: Event Router

Write a module that routes events using pattern matching and guards. Handle these cases:

- `%{type: :login, attempts: n}` where `n > 3` returns `"Account locked"`
- `%{type: :login, attempts: n}` where `n <= 3` returns `"Login attempt #{n}"`
- `%{type: :purchase, amount: a}` where `a > 1000` returns `"Large purchase: requires approval"`
- `%{type: :purchase, amount: a}` where `a > 0` returns `"Purchase: $#{a}"`
- Everything else returns `"Unknown event"`

{{< highlight elixir >}}
defmodule Router do
  def route(event) do
    ???
  end
end

# Router.route(%{type: :login, attempts: 5})
# "Account locked"

# Router.route(%{type: :purchase, amount: 50})
# "Purchase: $50"
{{< /highlight >}}

## Official Documentation to Help You Learn

- [Guards Documentation](https://hexdocs.pm/elixir/guards.html)
- [Elixir Case, Cond, and If](https://elixir-lang.org/getting-started/case-cond-and-if.html)
- [Kernel Guards Reference](https://hexdocs.pm/elixir/Kernel.html#guards)
- [defguard/1 Documentation](https://hexdocs.pm/elixir/Kernel.html#defguard/1)

---

*Part Eight | Functional Programming Through Elixir series*

**Previous in series:** [Part Seven - Higher-Order Functions](/posts/fp-series-higher-order-functions/)

**Next in series:** [Part Nine - Module Organization vs Class Hierarchies](/posts/fp-series-module-organization-vs-class-hierarchies/)
