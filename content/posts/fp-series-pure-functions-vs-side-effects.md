---
title: "Part Three | Functional Programming Through Elixir: Pure Functions vs Side Effects"
date: 2026-02-13
draft: false
tags: ["elixir", "functional-programming", "fp-series", "pure-functions", "side-effects", "learning"]
description: "Third post in a series exploring functional programming concepts through Elixir. Learn what makes a function pure, why it matters, and how to manage necessary side effects."
---

## The Foundation of Predictable Code

In the [first post](/posts/fp-series-immutability-vs-mutable-state/), we explored immutability. In the [second post](/posts/fp-series-functions-as-first-class-citizens/), we saw how functions are values. Now we'll tackle a concept that ties them together: **pure functions**.

Pure functions are the building blocks of functional programming. They're predictable, testable, and easy to reason about. But real applications need side effects - saving to databases, making HTTP requests, printing to the console. The key is knowing how to write pure functions and where to isolate side effects.

<!--more-->

## What is a Pure Function?

A **pure function** is a function that:

1. **Always returns the same output for the same input** (deterministic)
2. **Has no side effects** (doesn't modify anything outside itself)

Let's see this in action:

{{< highlight elixir >}}
# Pure function
defmodule Math do
  def add(a, b) do
    a + b
  end
end

# Always returns the same result for the same inputs
IO.puts(Math.add(2, 3))  # 5
IO.puts(Math.add(2, 3))  # 5
IO.puts(Math.add(2, 3))  # 5
{{< /highlight >}}

No matter how many times you call `Math.add(2, 3)`, it returns `5`. It doesn't depend on external state, time of day, network conditions, or anything else. It's completely predictable.

## What Are Side Effects?

A **side effect** is any observable change outside the function:

- Modifying a variable outside the function
- Writing to a file or database
- Making an HTTP request
- Printing to the console
- Sending a message to another process
- Generating random numbers
- Reading the current time

Here's an impure function with side effects:

{{< highlight elixir >}}
# Impure function - has side effects
defmodule Logger do
  def log_and_add(a, b) do
    result = a + b
    IO.puts("Adding #{a} + #{b} = #{result}")  # Side effect: prints to console
    result
  end
end

Logger.log_and_add(2, 3)  # Prints and returns 5
{{< /highlight >}}

This function performs I/O (printing), which is a side effect. The function does more than just compute a result - it affects the outside world.

## Contrast: OOP Methods Often Have Hidden Side Effects

In object-oriented programming, methods frequently modify internal state or trigger side effects:

{{< highlight python >}}
# Python - OOP with hidden side effects
class ShoppingCart:
    def __init__(self):
        self.items = []
        self.total = 0
        self.logger = Logger()

    def add_item(self, item):
        self.items.append(item)              # Side effect: mutates internal state
        self.total += item.price             # Side effect: mutates internal state
        self.logger.log(f"Added {item.name}") # Side effect: I/O
        self._send_analytics(item)           # Side effect: network request
        return True

cart = ShoppingCart()
cart.add_item(product)  # What does this do? Hard to tell without reading implementation
{{< /highlight >}}

Looking at `cart.add_item(product)`, you can't tell what side effects it has. Does it mutate state? Make network calls? Write to a database? You have to read the implementation to know.

Compare this to Elixir's functional approach:

{{< highlight elixir >}}
defmodule ShoppingCart do
  # Pure function - just data transformation
  def add_item(cart, item) do
    %{cart |
      items: [item | cart.items],
      total: cart.total + item.price
    }
  end
end

cart = %{items: [], total: 0}
updated_cart = ShoppingCart.add_item(cart, product)

# Side effects happen separately, explicitly
Logger.info("Added #{product.name}")
Analytics.track("item_added", product)
{{< /highlight >}}

The pure `add_item/2` function just transforms data. Side effects (logging, analytics) happen explicitly, separate from the core logic. This makes the code easier to understand, test, and modify.

## Why Pure Functions Matter

### 1. Easy to Test

Pure functions are trivial to test - no mocking, no setup, no cleanup:

{{< highlight elixir >}}
defmodule MathTest do
  use ExUnit.Case

  test "add/2 adds two numbers" do
    assert Math.add(2, 3) == 5
    assert Math.add(-1, 1) == 0
    assert Math.add(0, 0) == 0
  end

  test "multiply/2 multiplies two numbers" do
    assert Math.multiply(2, 3) == 6
    assert Math.multiply(-2, 3) == -6
    assert Math.multiply(0, 5) == 0
  end
end
{{< /highlight >}}

No need to mock databases, stub HTTP clients, or set up test fixtures. Just call the function with inputs and assert the output.

**Contrast with OOP:** Testing methods with side effects requires mocking dependencies, setting up state, and cleaning up after tests:

{{< highlight python >}}
# Python - Testing with mocks
def test_add_item(self):
    mock_logger = Mock()
    mock_analytics = Mock()
    cart = ShoppingCart(logger=mock_logger, analytics=mock_analytics)

    cart.add_item(product)

    # Now verify all the side effects happened
    mock_logger.log.assert_called_once_with("Added Product")
    mock_analytics.track.assert_called_once()
    # And verify state changed
    self.assertEqual(len(cart.items), 1)
{{< /highlight >}}

The test is coupled to implementation details. If you change how logging works, tests break even if the core logic is correct.

### 2. Parallelizable and Cacheable

Pure functions can be called in any order, on any thread, with no synchronization:

{{< highlight elixir >}}
# Pure function - safe to parallelize
defmodule PriceCalculator do
  def calculate_discount(price, discount_rate) do
    price * (1 - discount_rate)
  end

  def apply_tax(price, tax_rate) do
    price * (1 + tax_rate)
  end

  def final_price(original_price, discount_rate, tax_rate) do
    original_price
    |> calculate_discount(discount_rate)
    |> apply_tax(tax_rate)
  end
end

# Can process thousands of items in parallel - no race conditions
products
|> Task.async_stream(fn product ->
  PriceCalculator.final_price(product.price, 0.10, 0.08)
end)
|> Enum.to_list()
{{< /highlight >}}

Since pure functions don't modify shared state, there are no race conditions. You can also cache results - if you call `calculate_discount(100, 0.10)`, the result will always be `90.0`, so you can cache it.

### 3. Referential Transparency

Pure functions have **referential transparency** - you can replace a function call with its result without changing program behavior:

{{< highlight elixir >}}
# These are equivalent
final = Math.add(2, 3) |> Math.multiply(4)
final = 5 |> Math.multiply(4)
final = 20
{{< /highlight >}}

This property makes reasoning about code much easier. You can mentally "inline" function calls and understand what's happening without tracking state changes.

## Identifying Pure vs Impure Functions

Let's look at examples:

{{< highlight elixir >}}
# Pure - same inputs always produce same output
def double(x), do: x * 2

# Pure - works only with its arguments
def full_name(first, last), do: "#{first} #{last}"

# Pure - creates new data, doesn't modify anything
def add_item(list, item), do: [item | list]

# Impure - generates random values (different output each call)
def roll_dice(), do: :rand.uniform(6)

# Impure - depends on external state (current time)
def is_business_hours?() do
  hour = DateTime.utc_now().hour
  hour >= 9 and hour < 17
end

# Impure - performs I/O
def save_to_file(data, filename) do
  File.write(filename, data)
end

# Impure - sends messages to other processes
def notify_user(user_id, message) do
  send(user_id, {:notification, message})
end
{{< /highlight >}}

Notice that impure functions often:
- Have names suggesting action: `save_`, `send_`, `notify_`
- Return `{:ok, result}` or `{:error, reason}` (indicating possible failure from side effects)
- Interact with the outside world

## Making Functions Pure: Dependency Injection

You can often make impure functions pure by passing dependencies as arguments:

**Impure version:**
{{< highlight elixir >}}
# Depends on current time - impure
def is_expired?(expiration_date) do
  DateTime.compare(DateTime.utc_now(), expiration_date) == :gt
end
{{< /highlight >}}

**Pure version:**
{{< highlight elixir >}}
# Pass current time as argument - pure
def is_expired?(current_time, expiration_date) do
  DateTime.compare(current_time, expiration_date) == :gt
end

# Caller handles the side effect of reading current time
current_time = DateTime.utc_now()
expired? = is_expired?(current_time, expiration_date)
{{< /highlight >}}

Now `is_expired?/2` is pure and easy to test:

{{< highlight elixir >}}
test "is_expired?/2 returns true when date is in the past" do
  past = ~U[2025-01-01 00:00:00Z]
  current = ~U[2026-02-13 00:00:00Z]

  assert is_expired?(current, past) == true
end

test "is_expired?/2 returns false when date is in the future" do
  future = ~U[2027-01-01 00:00:00Z]
  current = ~U[2026-02-13 00:00:00Z]

  assert is_expired?(current, future) == false
end
{{< /highlight >}}

No need to mock `DateTime.utc_now()` - just pass different times.

## Separating Pure Logic from Side Effects

A common pattern in Elixir: keep your core logic pure, and push side effects to the edges of your system.

{{< highlight elixir >}}
defmodule OrderProcessor do
  # Pure function - just computes what to do
  def calculate_order_updates(order, payment_result) do
    case payment_result do
      {:ok, transaction_id} ->
        %{
          order: %{order | status: :paid, transaction_id: transaction_id},
          actions: [
            {:send_email, order.customer_email, :payment_confirmation},
            {:update_inventory, order.items},
            {:log, "Order #{order.id} paid successfully"}
          ]
        }

      {:error, reason} ->
        %{
          order: %{order | status: :payment_failed, failure_reason: reason},
          actions: [
            {:send_email, order.customer_email, :payment_failed},
            {:log, "Order #{order.id} payment failed: #{reason}"}
          ]
        }
    end
  end

  # Impure function - performs the side effects
  def execute_actions(actions) do
    Enum.each(actions, fn action ->
      case action do
        {:send_email, email, template} ->
          EmailService.send(email, template)

        {:update_inventory, items} ->
          Inventory.decrease_stock(items)

        {:log, message} ->
          Logger.info(message)
      end
    end)
  end
end

# Usage: pure logic first, side effects second
order = %{id: 123, customer_email: "user@example.com", items: [...], status: :pending}
payment_result = PaymentGateway.charge(order)

%{order: updated_order, actions: actions} =
  OrderProcessor.calculate_order_updates(order, payment_result)

# Now perform side effects
OrderProcessor.execute_actions(actions)
{{< /highlight >}}

The pure `calculate_order_updates/2` function is easy to test - just verify it returns the right data structure. You can test all the business logic without mocking email services, databases, or loggers.

The impure `execute_actions/1` is simple - it just performs the side effects specified by the pure function. Testing can focus on integration rather than complex business logic.

## When Side Effects Are Necessary

Real applications need side effects. The functional approach isn't about eliminating them - it's about:

1. **Isolating them** - Keep pure logic separate from side effects
2. **Making them explicit** - Side effects should be obvious from function names and return types
3. **Deferring them** - Compute what to do (pure), then do it (impure)

{{< highlight elixir >}}
defmodule UserRegistration do
  # Pure: validates and prepares data
  def prepare_registration(params) do
    with {:ok, validated} <- validate_params(params),
         {:ok, user_data} <- build_user_data(validated),
         {:ok, email_data} <- prepare_welcome_email(user_data) do
      {:ok, %{user: user_data, email: email_data}}
    end
  end

  # Impure: performs side effects
  def execute_registration(%{user: user_data, email: email_data}) do
    with {:ok, user} <- Repo.insert(user_data),
         {:ok, _email} <- Mailer.send(email_data) do
      {:ok, user}
    end
  end

  # High-level function coordinates both
  def register_user(params) do
    with {:ok, prepared} <- prepare_registration(params),
         {:ok, user} <- execute_registration(prepared) do
      {:ok, user}
    end
  end
end
{{< /highlight >}}

The validation and business logic (`prepare_registration/1`) is pure and easy to test. The database and email operations (`execute_registration/1`) are isolated and explicit.

## Key Takeaways

- **Pure functions** always return the same output for the same input and have no side effects
- **Side effects** include I/O, mutation, random numbers, reading time - anything that affects or depends on the outside world
- **OOP often hides side effects** in methods, making code harder to understand and test
- **Benefits of pure functions:**
  - Easy to test (no mocking required)
  - Parallelizable and cacheable
  - Referentially transparent
  - Easy to reason about
- **Make functions pure by:**
  - Passing dependencies as arguments
  - Returning what to do instead of doing it
  - Separating pure logic from side effects
- **Real apps need side effects** - the goal is to isolate and make them explicit, not eliminate them

## Try It Yourself

Open up `iex` and practice identifying and refactoring pure vs impure functions.

### Exercise 1: Identify Pure vs Impure

Determine whether each function is pure or impure, and why:

{{< highlight elixir >}}
def greet(name), do: "Hello, #{name}!"

def random_greeting(name) do
  greetings = ["Hi", "Hello", "Hey"]
  greeting = Enum.random(greetings)
  "#{greeting}, #{name}!"
end

def total_price(items) do
  Enum.reduce(items, 0, fn item, acc -> acc + item.price end)
end

def discounted_price(price) do
  if DateTime.utc_now().hour < 12 do
    price * 0.9
  else
    price
  end
end
{{< /highlight >}}

### Exercise 2: Refactor to Pure

Refactor this impure function to be pure:

{{< highlight elixir >}}
def process_order(order) do
  Logger.info("Processing order #{order.id}")

  if order.total > 100 do
    updated = %{order | discount: 10}
    Repo.update(updated)
    {:ok, updated}
  else
    {:ok, order}
  end
end
{{< /highlight >}}

**Hint:** Separate the logic (computing the new order) from the side effects (logging and database update).

## Official Documentation to Help You Learn

- [Elixir Modules and Functions](https://elixir-lang.org/getting-started/modules-and-functions.html)
- [Elixir with Statement](https://elixir-lang.org/getting-started/mix-otp/docs-tests-and-with.html)
- [ExUnit Testing Documentation](https://hexdocs.pm/ex_unit/ExUnit.html)

---

*Part Three | Functional Programming Through Elixir series*

**Previous in series:** [Part Two - Functions as First-Class Citizens](/posts/fp-series-functions-as-first-class-citizens/)

**Next in series:** Part Four - Pattern Matching
