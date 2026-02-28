---
title: "Part Five | Functional Programming Through Elixir: Recursion Over Iteration"
date: 2026-02-27
draft: false
tags: ["elixir", "functional-programming", "fp-series", "recursion", "tail-call-optimization", "learning"]
description: "Fifth post in a series exploring functional programming concepts through Elixir. Learn how recursion replaces loops and why tail-call optimization makes it practical."
---

## A Different Way to Loop

In [Part Four](/posts/fp-series-pattern-matching/), we learned how pattern matching enables elegant data handling. Now we'll tackle another fundamental shift: **in functional programming, you don't use loops - you use recursion**.

If you're coming from an object-oriented background, loops are probably second nature. `for`, `while`, `forEach` - these are the tools you reach for when you need to process multiple items. But in Elixir, immutability means you can't have traditional loop counters that increment. Instead, you use recursion: functions that call themselves.

Once you understand recursion, you'll find it more expressive than loops. It naturally handles complex iteration patterns, and Elixir's tail-call optimization makes it just as efficient as loops in other languages.

<!--more-->

## What is Recursion?

**Recursion** is when a function calls itself. Every recursive function has two parts:

1. **Base case** - When to stop recursing (the exit condition)
2. **Recursive case** - How to break down the problem and call yourself again

Let's start with the classic example - calculating factorial:

{{< highlight elixir >}}
defmodule Math do
  # Base case: factorial of 0 is 1
  def factorial(0), do: 1

  # Recursive case: n! = n * (n-1)!
  def factorial(n) when n > 0 do
    n * factorial(n - 1)
  end
end

IO.puts(Math.factorial(5))  # 120
# How it works:
# factorial(5) = 5 * factorial(4)
# factorial(4) = 4 * factorial(3)
# factorial(3) = 3 * factorial(2)
# factorial(2) = 2 * factorial(1)
# factorial(1) = 1 * factorial(0)
# factorial(0) = 1  (base case!)
# Now unwind: 1 * 1 * 2 * 3 * 4 * 5 = 120
{{< /highlight >}}

Notice how pattern matching separates the base case from the recursive case. When `n` reaches 0, we stop recursing and return 1. Otherwise, we multiply `n` by the factorial of `n - 1`.

## Contrast: Loops in OOP

In most languages, you'd use a loop:

{{< highlight python >}}
# Python - imperative loop
def factorial(n):
    result = 1
    for i in range(1, n + 1):
        result *= i  # Mutate result
    return result

print(factorial(5))  # 120
{{< /highlight >}}

{{< highlight javascript >}}
// JavaScript - imperative loop
function factorial(n) {
    let result = 1;
    for (let i = 1; i <= n; i++) {
        result *= i;  // Mutate result
    }
    return result;
}

console.log(factorial(5));  // 120
{{< /highlight >}}

The loop approach:
- Requires mutable variables (`result`, `i`)
- Mixes control flow (the loop) with logic (the multiplication)
- Harder to reason about because state changes on each iteration

The recursive approach:
- No mutable state - each call has its own immutable `n`
- Separates base case from recursive logic clearly
- Each step is self-contained and easier to understand

## Recursion with Lists

Recursion really shines when working with lists. The pattern is always the same:
- Base case: empty list
- Recursive case: process the head, recurse on the tail

### Example: Summing a List

{{< highlight elixir >}}
defmodule ListMath do
  # Base case: sum of empty list is 0
  def sum([]), do: 0

  # Recursive case: head + sum of tail
  def sum([head | tail]) do
    head + sum(tail)
  end
end

IO.puts(ListMath.sum([1, 2, 3, 4, 5]))  # 15

# How it works:
# sum([1, 2, 3, 4, 5]) = 1 + sum([2, 3, 4, 5])
# sum([2, 3, 4, 5])    = 2 + sum([3, 4, 5])
# sum([3, 4, 5])       = 3 + sum([4, 5])
# sum([4, 5])          = 4 + sum([5])
# sum([5])             = 5 + sum([])
# sum([])              = 0  (base case!)
# Now unwind: 0 + 5 + 4 + 3 + 2 + 1 = 15
{{< /highlight >}}

**Compare with OOP:**

{{< highlight java >}}
// Java - imperative loop
public int sum(List<Integer> numbers) {
    int total = 0;
    for (int num : numbers) {
        total += num;  // Mutate total
    }
    return total;
}
{{< /highlight >}}

### Example: Mapping Over a List

Let's implement our own `map` function using recursion:

{{< highlight elixir >}}
defmodule MyEnum do
  # Base case: mapping over empty list returns empty list
  def map([], _function), do: []

  # Recursive case: apply function to head, recurse on tail
  def map([head | tail], function) do
    [function.(head) | map(tail, function)]
  end
end

numbers = [1, 2, 3, 4, 5]
doubled = MyEnum.map(numbers, fn x -> x * 2 end)
IO.inspect(doubled)  # [2, 4, 6, 8, 10]

# How it works:
# map([1,2,3,4,5], fn) = [fn(1) | map([2,3,4,5], fn)]
# map([2,3,4,5], fn)   = [fn(2) | map([3,4,5], fn)]
# map([3,4,5], fn)     = [fn(3) | map([4,5], fn)]
# map([4,5], fn)       = [fn(4) | map([5], fn)]
# map([5], fn)         = [fn(5) | map([], fn)]
# map([], fn)          = []  (base case!)
# Result: [2, 4, 6, 8, 10]
{{< /highlight >}}

This is essentially how `Enum.map` works internally! Recursion naturally expresses the idea: "transform the first item, then transform the rest."

## The Problem: Stack Overflow

There's a catch with the recursive examples so far. Each function call adds a frame to the call stack. For large lists, you'll run out of stack space:

{{< highlight elixir >}}
# This will crash with a huge list!
defmodule BadSum do
  def sum([]), do: 0
  def sum([head | tail]), do: head + sum(tail)
end

# Trying to sum a million numbers
big_list = Enum.to_list(1..1_000_000)
BadSum.sum(big_list)  # ** (SystemStackError) stack overflow
{{< /highlight >}}

Why? Because each recursive call must wait for the next call to complete before it can add:

```
sum([1,2,3]) waits for...
  sum([2,3]) waits for...
    sum([3]) waits for...
      sum([])  # Finally returns 0
    Now add 3: returns 3
  Now add 2: returns 5
Now add 1: returns 6
```

The stack grows with the size of the list. For large lists, this crashes.

## The Solution: Tail-Call Optimization

**Tail-call optimization (TCO)** is a compiler optimization where if a function's last action is to call itself, the compiler can reuse the current stack frame instead of creating a new one.

A **tail-recursive** function is one where the recursive call is the very last thing the function does - no operations happen after the recursive call returns.

Let's rewrite our `sum` function to be tail-recursive:

{{< highlight elixir >}}
defmodule TailSum do
  # Public function: starts the recursion with accumulator 0
  def sum(list), do: sum_helper(list, 0)

  # Private helper with accumulator
  # Base case: empty list, return accumulator
  defp sum_helper([], acc), do: acc

  # Recursive case: add head to accumulator, recurse on tail
  defp sum_helper([head | tail], acc) do
    sum_helper(tail, acc + head)  # Last operation is the recursive call!
  end
end

IO.puts(TailSum.sum([1, 2, 3, 4, 5]))  # 15

# How it works (with accumulator):
# sum_helper([1,2,3,4,5], 0)
# sum_helper([2,3,4,5], 1)    # 0 + 1
# sum_helper([3,4,5], 3)      # 1 + 2
# sum_helper([4,5], 6)        # 3 + 3
# sum_helper([5], 10)         # 6 + 4
# sum_helper([], 15)          # 10 + 5
# Returns 15 (base case)
{{< /highlight >}}

**Key differences:**
- We use an **accumulator** (`acc`) to build up the result as we go
- The recursive call is the **last operation** - we don't do anything with its return value
- Because of TCO, Elixir reuses the same stack frame for each call

This can now handle lists of any size:

{{< highlight elixir >}}
big_list = Enum.to_list(1..1_000_000)
TailSum.sum(big_list)  # Works perfectly! No stack overflow
{{< /highlight >}}

## Tail-Recursive vs Non-Tail-Recursive

Let's compare the two approaches side by side:

### Non-Tail-Recursive (Builds Up Stack)

{{< highlight elixir >}}
def factorial(0), do: 1
def factorial(n), do: n * factorial(n - 1)
#                       ↑
#                       Operation AFTER recursive call (multiply)
{{< /highlight >}}

After the recursive call returns, we still have work to do (multiply by `n`). The stack must remember `n` for each level.

### Tail-Recursive (Constant Stack)

{{< highlight elixir >}}
def factorial(n), do: factorial_helper(n, 1)

defp factorial_helper(0, acc), do: acc
defp factorial_helper(n, acc), do: factorial_helper(n - 1, n * acc)
#                                  ↑
#                                  Recursive call is the LAST thing
{{< /highlight >}}

The multiplication happens **before** the recursive call (in the accumulator). Nothing happens after the recursive call, so the stack frame can be reused.

## Common Recursive Patterns with Accumulators

### Pattern 1: Reversing a List

{{< highlight elixir >}}
defmodule MyList do
  def reverse(list), do: reverse_helper(list, [])

  # Base case: empty list, return accumulator
  defp reverse_helper([], acc), do: acc

  # Recursive case: prepend head to accumulator
  defp reverse_helper([head | tail], acc) do
    reverse_helper(tail, [head | acc])
  end
end

IO.inspect(MyList.reverse([1, 2, 3, 4, 5]))  # [5, 4, 3, 2, 1]

# How it works:
# reverse_helper([1,2,3,4,5], [])
# reverse_helper([2,3,4,5], [1])
# reverse_helper([3,4,5], [2, 1])
# reverse_helper([4,5], [3, 2, 1])
# reverse_helper([5], [4, 3, 2, 1])
# reverse_helper([], [5, 4, 3, 2, 1])
# Returns [5, 4, 3, 2, 1]
{{< /highlight >}}

### Pattern 2: Filtering a List

{{< highlight elixir >}}
defmodule MyFilter do
  def filter(list, predicate), do: filter_helper(list, predicate, [])

  # Base case: empty list, reverse accumulator (we built it backwards)
  defp filter_helper([], _predicate, acc), do: Enum.reverse(acc)

  # If head passes predicate, add to accumulator
  defp filter_helper([head | tail], predicate, acc) do
    if predicate.(head) do
      filter_helper(tail, predicate, [head | acc])
    else
      filter_helper(tail, predicate, acc)
    end
  end
end

numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
evens = MyFilter.filter(numbers, fn x -> rem(x, 2) == 0 end)
IO.inspect(evens)  # [2, 4, 6, 8, 10]
{{< /highlight >}}

### Pattern 3: Counting Items

{{< highlight elixir >}}
defmodule Counter do
  def count_matching(list, predicate), do: count_helper(list, predicate, 0)

  # Base case: empty list, return count
  defp count_helper([], _predicate, count), do: count

  # If head matches, increment count
  defp count_helper([head | tail], predicate, count) do
    if predicate.(head) do
      count_helper(tail, predicate, count + 1)
    else
      count_helper(tail, predicate, count)
    end
  end
end

numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
even_count = Counter.count_matching(numbers, fn x -> rem(x, 2) == 0 end)
IO.puts(even_count)  # 5
{{< /highlight >}}

## When to Use Recursion

### Perfect for Recursion:
- **Processing lists** - Natural fit with `[head | tail]` pattern matching
- **Tree traversal** - Each subtree is handled recursively
- **Divide-and-conquer algorithms** - Break problem into smaller pieces
- **State machines** - Each state transition is a recursive call

### When Libraries Are Better:
In practice, you'll rarely write recursive functions from scratch. Elixir's `Enum` and `Stream` modules already provide optimized implementations:

{{< highlight elixir >}}
# Instead of writing recursive sum:
defmodule MySum do
  def sum(list), do: sum_helper(list, 0)
  defp sum_helper([], acc), do: acc
  defp sum_helper([h | t], acc), do: sum_helper(t, acc + h)
end

# Just use Enum.sum:
Enum.sum([1, 2, 3, 4, 5])  # 15

# Instead of recursive filter:
defmodule MyFilter do
  # ... complex recursive implementation ...
end

# Just use Enum.filter:
Enum.filter([1, 2, 3, 4, 5], fn x -> rem(x, 2) == 0 end)  # [2, 4]
{{< /highlight >}}

**When to write your own recursion:**
- Custom business logic that doesn't fit standard patterns
- Complex tree or graph traversal
- State machines or process loops (we'll cover this in a later post)
- Learning and understanding FP concepts!

## Real-World Example: Flattening Nested Lists

Let's implement a function to flatten arbitrarily nested lists - a problem that's elegant with recursion:

{{< highlight elixir >}}
defmodule Flatten do
  def flatten(list), do: flatten_helper(list, [])

  # Base case: empty list
  defp flatten_helper([], acc), do: Enum.reverse(acc)

  # If head is a list, flatten it first
  defp flatten_helper([head | tail], acc) when is_list(head) do
    flattened_head = flatten_helper(head, [])
    flatten_helper(tail, Enum.reverse(flattened_head) ++ acc)
  end

  # If head is not a list, add to accumulator
  defp flatten_helper([head | tail], acc) do
    flatten_helper(tail, [head | acc])
  end
end

nested = [1, [2, [3, 4], 5], [6, [7, 8]], 9]
IO.inspect(Flatten.flatten(nested))  # [1, 2, 3, 4, 5, 6, 7, 8, 9]
{{< /highlight >}}

**OOP Comparison:** In OOP, you'd use nested loops or maintain a stack manually. The recursive approach naturally mirrors the structure of the data.

## Thinking Recursively: A Mental Model

When approaching a problem recursively:

1. **Identify the base case** - What's the simplest input? (Usually empty list, zero, etc.)
2. **Break down the problem** - How can you make the input smaller?
3. **Assume the recursion works** - Trust that your function works for the smaller input
4. **Combine the results** - How do you use the recursive result to solve the current input?

Example: "Reverse a list"
1. Base case: Empty list is already reversed → `[]`
2. Break down: Take the first element and the rest: `[head | tail]`
3. Assume: `reverse(tail)` correctly reverses the tail
4. Combine: Put `head` at the end of the reversed tail

{{< highlight elixir >}}
def reverse([]), do: []
def reverse([head | tail]), do: reverse(tail) ++ [head]
{{< /highlight >}}

(Note: This version is simpler but less efficient - appending to lists is O(n). The accumulator version we saw earlier is O(1) per step.)

## Key Takeaways

- **Recursion replaces loops** in functional programming - functions call themselves instead of using for/while
- **Every recursive function has a base case** (when to stop) and a **recursive case** (how to continue)
- **Pattern matching** makes base and recursive cases clean and explicit
- **Tail-call optimization** prevents stack overflow by reusing stack frames
- **Tail-recursive functions** use accumulators to build results and make the recursive call the last operation
- **Common patterns** include summing, filtering, mapping, and counting with accumulators
- **In practice**, use `Enum` and `Stream` modules - they're already optimized
- **Write recursion for** custom logic, tree traversal, and learning FP concepts

## Try It Yourself

Open `iex` and practice these recursive exercises.

### Exercise 1: Length of a List

Write a recursive function to find the length of a list:

{{< highlight elixir >}}
defmodule MyList do
  def length([]), do: ???
  def length([_head | tail]), do: ???
end

# Test it:
MyList.length([1, 2, 3, 4, 5])  # Should return 5
{{< /highlight >}}

**Bonus:** Make it tail-recursive with an accumulator.

### Exercise 2: Maximum Value

Find the maximum value in a list using recursion:

{{< highlight elixir >}}
defmodule MyMax do
  # Hint: You'll need a base case for single-element lists
  def max([single]), do: ???

  def max([head | tail]) do
    # Compare head with max of tail
    ???
  end
end

# Test it:
MyMax.max([3, 7, 2, 9, 1, 5])  # Should return 9
{{< /highlight >}}

### Exercise 3: Range Generator

Generate a list from `start` to `stop`:

{{< highlight elixir >}}
defmodule MyRange do
  def range(start, stop) when start > stop, do: ???
  def range(start, stop), do: ???
end

# Test it:
MyRange.range(1, 5)  # Should return [1, 2, 3, 4, 5]
{{< /highlight >}}

### Exercise 4: Make It Tail-Recursive

Take your solutions above and convert them to tail-recursive versions using accumulators.

## Official Documentation to Help You Learn

- [Elixir Recursion Guide](https://elixir-lang.org/getting-started/recursion.html)
- [Tail Call Optimization Explained](https://en.wikipedia.org/wiki/Tail_call)
- [Enum Module Documentation](https://hexdocs.pm/elixir/Enum.html)

---

*Part Five | Functional Programming Through Elixir series*

**Previous in series:** [Part Four - Pattern Matching](/posts/fp-series-pattern-matching/)

**Next in series:** Part Six - The Pipe Operator (Coming Soon)
