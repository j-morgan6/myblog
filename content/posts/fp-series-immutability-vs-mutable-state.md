---
title: "Part One | Functional Programming Through Elixir: Immutability vs Mutable State"
date: 2026-02-02
draft: false
tags: ["elixir", "functional-programming", "fp-series", "immutability", "learning"]
description: "First post in a series exploring functional programming concepts through Elixir. Learn why immutability is fundamental to FP and how it differs from mutable state in object-oriented programming."
---

## The Foundation of Functional Programming

If you're coming from an object-oriented programming background, one of the most fundamental shifts in thinking when learning functional programming is **immutability**. In Elixir and other functional languages, data never changes once it's created. This might sound limiting at first, but it's actually a superpower that unlocks predictability, thread safety, and easier debugging.

This is the first post in a series exploring functional programming concepts through Elixir, aimed at developers transitioning from OOP paradigms.

<!--more-->

## What is Immutability?

**Immutability** means that once a data structure is created, it cannot be modified. Any operation that appears to "change" the data actually creates a new copy with the desired modifications, leaving the original untouched.

Let's see this in action with Elixir:

{{< highlight elixir >}}
# Create a list
original_list = [1, 2, 3]

# "Add" an element using the cons operator [head | tail]
# This prepends 0 to the list, creating a new list
new_list = [0 | original_list]

IO.inspect(original_list)  # [1, 2, 3] - unchanged!
IO.inspect(new_list)       # [0, 1, 2, 3]
{{< /highlight >}}

The `original_list` remains exactly as it was. The `[0 | original_list]` syntax creates an entirely new list with 0 at the front, leaving the original untouched.

## Contrast: Mutable State in OOP

In most object-oriented languages, objects maintain internal state that can be modified over time:

{{< highlight python >}}
# Python example
class BankAccount:
    def __init__(self, balance):
        self.balance = balance

    def deposit(self, amount):
        self.balance += amount  # Mutates internal state
        return self.balance

account = BankAccount(100)
account.deposit(50)
print(account.balance)  # 150 - the object changed!
{{< /highlight >}}

The `account` object's internal state changed. The object you created is no longer the same object, its `balance` property now has a different value.

## The Elixir Way: Data Transformation

In Elixir, instead of mutating objects, we transform data:

{{< highlight elixir >}}
defmodule BankAccount do
  # defstruct defines a struct (similar to a class, but just data)
  defstruct balance: 0

  def deposit(account, amount) do
    # The %{struct | field: new_value} syntax creates a NEW struct
    # with the updated field, leaving the original unchanged
    %{account | balance: account.balance + amount}
  end
end

# Create a new BankAccount struct with balance: 100
account = %BankAccount{balance: 100}
updated_account = BankAccount.deposit(account, 50)

IO.inspect(account)         # %BankAccount{balance: 100}
IO.inspect(updated_account) # %BankAccount{balance: 150}
{{< /highlight >}}

Notice that `account` still has a balance of 100. The `deposit/2` function didn't change it - it returned a **new** account struct with the updated balance. If you want to work with the new value, you must explicitly use `updated_account`.

**Elixir Syntax Note:** The `%{struct | field: value}` syntax is how you "update" a map or struct in Elixir. It doesn't actually update - it creates a new copy with the specified fields changed.

## Why Immutability Matters

### 1. Thread Safety Without Locks

One of the biggest challenges in concurrent programming is dealing with shared mutable state. When multiple threads can modify the same object, you need locks, mutexes, and complex synchronization logic to prevent race conditions.

With immutability, this entire class of problems disappears:

{{< highlight elixir >}}
# Multiple processes can safely work with the same data
data = %{count: 0, items: []}

# Process 1
Task.async(fn ->
  updated = %{data | count: data.count + 1}
  IO.inspect(updated)
end)

# Process 2
Task.async(fn ->
  updated = %{data | items: ["new_item" | data.items]}
  IO.inspect(updated)
end)

# Original data is never modified - no race conditions!
IO.inspect(data)  # %{count: 0, items: []}
{{< /highlight >}}

Each process gets its own copy of the data. There's no shared mutable state to synchronize.

### 2. Predictability and Easier Debugging

When data can't change unexpectedly, your code becomes dramatically easier to reason about:

{{< highlight elixir >}}
defmodule ShoppingCart do
  def calculate_total(items) do
    items
    # The |> pipe operator passes the result to the next function
    # This extracts the price from each item
    |> Enum.map(fn item -> item.price end)
    |> Enum.sum()
  end

  def apply_discount(items, discount_percentage) do
    Enum.map(items, fn item ->
      discount = item.price * discount_percentage
      # Create a new item map with updated price
      %{item | price: item.price - discount}
    end)
  end
end

cart_items = [
  %{name: "Book", price: 20.00},
  %{name: "Pen", price: 5.00}
]

total = ShoppingCart.calculate_total(cart_items)
IO.puts("Total: $#{total}")  # Total: $25.00

# Apply a 10% discount
discounted_items = ShoppingCart.apply_discount(cart_items, 0.10)
discounted_total = ShoppingCart.calculate_total(discounted_items)
IO.puts("Discounted Total: $#{discounted_total}")  # Discounted Total: $22.50

# Original cart_items unchanged!
original_total = ShoppingCart.calculate_total(cart_items)
IO.puts("Original Total: $#{original_total}")  # Original Total: $25.00
{{< /highlight >}}

You can call `apply_discount/2` and know with certainty that `cart_items` will remain unchanged. No hidden side effects, no unexpected modifications.

Compare this to an OOP approach where the discount method might mutate the items in place:

{{< highlight javascript >}}
// JavaScript example
class ShoppingCart {
  constructor(items) {
    this.items = items;
  }

  applyDiscount(percentage) {
    this.items.forEach(item => {
      item.price = item.price * (1 - percentage);  // Mutates!
    });
  }

  calculateTotal() {
    return this.items.reduce((sum, item) => sum + item.price, 0);
  }
}

const cart = new ShoppingCart([
  { name: "Book", price: 20.00 },
  { name: "Pen", price: 5.00 }
]);

console.log(cart.calculateTotal());  // 25.00
cart.applyDiscount(0.10);
console.log(cart.calculateTotal());  // 22.50

// Original data is gone - we can't get back to $25.00!
{{< /highlight >}}

Once you call `applyDiscount()`, the original prices are lost forever. In complex applications, this kind of mutation makes debugging significantly harder because you can't easily track how data changed over time.

### 3. Time-Travel Debugging and Undo/Redo

Because immutability preserves history naturally, implementing features like undo/redo becomes trivial:

{{< highlight elixir >}}
defmodule Document do
  defstruct content: "", history: []

  def edit(doc, new_content) do
    %{doc |
      content: new_content,
      history: [doc.content | doc.history]
    }
  end

  def undo(doc) do
    case doc.history do
      [previous | rest] ->
        %{doc | content: previous, history: rest}
      [] ->
        doc  # Nothing to undo
    end
  end
end

doc = %Document{content: "Hello"}
doc = Document.edit(doc, "Hello World")
doc = Document.edit(doc, "Hello World!")

IO.inspect(doc.content)  # "Hello World!"

doc = Document.undo(doc)
IO.inspect(doc.content)  # "Hello World"

doc = Document.undo(doc)
IO.inspect(doc.content)  # "Hello"
{{< /highlight >}}

Each edit creates a new document while preserving the old one in history. Undo is just retrieving the previous version.

## Performance: Isn't Copying Expensive?

You might be thinking: "Creating new copies of data for every change sounds incredibly expensive!"

Elixir (and Erlang, which it runs on) use **structural sharing** to make this efficient. When you create a "new" data structure, the runtime shares as much of the old structure as possible:

{{< highlight elixir >}}
# When prepending to a list
list1 = [2, 3, 4]
list2 = [1 | list1]

# list2 doesn't copy [2, 3, 4]
# It creates a new head node [1] that points to the existing list
# Memory layout:
# list1: [2, 3, 4]
# list2: [1] -> [2, 3, 4]  (reuses list1)
{{< /highlight >}}

For maps and other structures, only the changed parts are copied. The unchanged portions are shared between the old and new versions.

## Embracing Immutability: A Mental Shift

If you're used to OOP, immutability requires a shift in thinking:

**OOP mindset:** "I have an object. I'll modify its properties to represent the new state."

{{< highlight java >}}
// Java
user.setName("Alice");
user.setAge(30);
user.setEmail("alice@example.com");
{{< /highlight >}}

**FP mindset:** "I have data. I'll transform it into new data that represents the new state."

{{< highlight elixir >}}
# Elixir - the |> pipe operator passes data through transformations
# Each step creates a new map, leaving previous versions unchanged
user
|> Map.put(:name, "Alice")
|> Map.put(:age, 30)
|> Map.put(:email, "alice@example.com")
{{< /highlight >}}

The functional approach explicitly shows data flowing through transformations, making the code easier to trace and understand. The `|>` pipe operator takes the result from the left and passes it as the first argument to the function on the right.

## When You Actually Need State

You might wonder: "If nothing can change, how do I build stateful applications?"

The key insight is that **immutability applies to data structures, not to your entire system**. Elixir handles stateful applications through **processes** - isolated units that maintain state by passing immutable data through recursive function calls. We'll cover this in detail in a later post on the Actor Model, but here's the concept:

Instead of mutating a variable, a process maintains state by calling itself recursively with new immutable data:

{{< highlight elixir >}}
# Simplified concept (not actual implementation code)
def loop(current_state) do
  # Receive a message, create new state (immutable)
  new_state = transform(current_state)

  # Call ourselves again with the new state
  loop(new_state)
end
{{< /highlight >}}

Each "update" creates new data and passes it to the next iteration. The data itself never changes - we just move through a sequence of immutable snapshots. This gives you stateful behavior while keeping all data immutable.

## Key Takeaways

- **Immutability means data never changes** - operations create new copies instead of modifying originals
- **OOP mutates objects** - state changes happen in place, often with hidden side effects
- **Benefits of immutability:**
  - Thread safety without locks
  - Predictable code that's easier to debug
  - Natural time-travel and undo/redo capabilities
  - Fewer bugs from unexpected state changes
- **Performance is optimized** through structural sharing - copying isn't as expensive as it seems
- **State is still possible** - it's managed through processes, not mutable data structures

## Try It Yourself

Open up `iex` and follow along with this exercise to experience immutability firsthand.

**Walkthrough: Tracking Multiple Versions**

{{< highlight elixir >}}
# Start with a shopping cart
cart_v1 = %{items: ["apple"], total: 1.50}

# Add another item - creates a new version
cart_v2 = %{cart_v1 | items: ["apple", "banana"], total: 2.75}

# Add a third item
cart_v3 = %{cart_v2 | items: ["apple", "banana", "orange"], total: 4.00}

# All three versions still exist!
IO.inspect(cart_v1)  # %{items: ["apple"], total: 1.5}
IO.inspect(cart_v2)  # %{items: ["apple", "banana"], total: 2.75}
IO.inspect(cart_v3)  # %{items: ["apple", "banana", "orange"], total: 4.0}
{{< /highlight >}}

Notice how each transformation created a new version while preserving the old ones. This is immutability in action.

**Now You Try:**

Build a simple "undo" system for a text editor. Start with `doc = %{content: ""}` and perform several edits, storing each version in a list. Then implement an undo function that goes back through your versions. Can you access any previous state of the document?

## Official Documentation to Help You Learn

- [Official Elixir Documentation](https://elixir-lang.org/docs.html)
- [Elixir Getting Started Guide](https://elixir-lang.org/getting-started/introduction.html)

---

*Part One | Functional Programming Through Elixir series*

**Next in series:** [Part Two - Functions as First-Class Citizens](/posts/fp-series-functions-as-first-class-citizens/)