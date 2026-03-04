---
title: "Part Six | Functional Programming Through Elixir: The Pipe Operator"
date: 2026-03-03
draft: false
tags: ["elixir", "functional-programming", "fp-series", "pipe-operator", "composability", "learning"]
description: "Sixth post in a series exploring functional programming concepts through Elixir. How the pipe operator turns nested function calls into readable data pipelines."
---

## Unreadable Nesting

In [Part Five](/posts/fp-series-recursion-over-iteration/), we saw how recursion replaces loops. Now let's look at a different readability problem: **nested function calls**.

Say you need to take a user's input, trim whitespace, convert it to lowercase, and capitalize the first letter. In Elixir, without the pipe operator, you'd write:

{{< highlight elixir >}}
String.capitalize(String.downcase(String.trim("  HELLO WORLD  ")))
# "Hello world"
{{< /highlight >}}

You have to read this from the inside out. The first operation (`trim`) is buried in the middle, and the last operation (`capitalize`) is on the outside. The more steps you add, the worse it gets.

If you're coming from OOP, you might be used to method chaining, which reads left to right:

{{< highlight python >}}
# Python - method chaining
"  HELLO WORLD  ".strip().lower().capitalize()
# "Hello world"
{{< /highlight >}}

{{< highlight javascript >}}
// JavaScript - method chaining
"  HELLO WORLD  ".trim().toLowerCase()
// "hello world"
// (JavaScript doesn't have a direct capitalize method)
{{< /highlight >}}

Method chaining reads nicely, but it only works because each method is attached to the object. In functional programming, functions aren't bound to objects. They're standalone. So how do we get the same readability?

<!--more-->

## The Pipe Operator

Elixir's **pipe operator** (`|>`) solves this. It takes the result of the expression on the left and passes it as the **first argument** to the function on the right:

{{< highlight elixir >}}
"  HELLO WORLD  "
|> String.trim()
|> String.downcase()
|> String.capitalize()
# "Hello world"
{{< /highlight >}}

Now you can read it top to bottom, in the order things actually happen:

1. Start with `"  HELLO WORLD  "`
2. Trim whitespace
3. Convert to lowercase
4. Capitalize first letter

Each line is one clear step. No nesting, no reading inside-out.

## How `|>` Works Under the Hood

The pipe operator is simple - it rewrites your code at compile time. This:

{{< highlight elixir >}}
value |> function(arg2, arg3)
{{< /highlight >}}

Becomes this:

{{< highlight elixir >}}
function(value, arg2, arg3)
{{< /highlight >}}

The left side always becomes the **first argument**. That's all it does. This is why Elixir's standard library consistently puts the "data" argument first. It makes everything pipeable.

A few more examples to make this concrete:

{{< highlight elixir >}}
# These two are identical:
String.split("hello world", " ")
"hello world" |> String.split(" ")

# These two are also identical:
Enum.map([1, 2, 3], fn x -> x * 2 end)
[1, 2, 3] |> Enum.map(fn x -> x * 2 end)
{{< /highlight >}}

## Building Data Transformation Pipelines

The pipe operator is at its best when you chain multiple transformations together.

### Example 1: Cleaning User Input

A user types in their email with extra spaces and mixed casing:

{{< highlight elixir >}}
raw_email = "   Jose.Reyes@Example.COM   "

clean_email =
  raw_email
  |> String.trim()
  |> String.downcase()

IO.puts(clean_email)  # "jose.reyes@example.com"
{{< /highlight >}}

**Without the pipe:**

{{< highlight elixir >}}
clean_email = String.downcase(String.trim(raw_email))
{{< /highlight >}}

With two steps, the nested version is still manageable. But pipelines make the intent clearer: "take the raw email, trim it, then downcase it."

### Example 2: Processing a List of Numbers

You want to take a list of numbers, keep only the even ones, double them, and sort the result:

{{< highlight elixir >}}
[5, 3, 8, 1, 4, 7, 2, 6]
|> Enum.filter(fn x -> rem(x, 2) == 0 end)
|> Enum.map(fn x -> x * 2 end)
|> Enum.sort()
# [4, 8, 12, 16]
{{< /highlight >}}

Each step is a small operation. You can read what happens at each stage.

**Compare with OOP:**

{{< highlight python >}}
# Python
numbers = [5, 3, 8, 1, 4, 7, 2, 6]
evens = [x for x in numbers if x % 2 == 0]
doubled = [x * 2 for x in evens]
result = sorted(doubled)
# [4, 8, 12, 16]
{{< /highlight >}}

{{< highlight javascript >}}
// JavaScript - method chaining on arrays
[5, 3, 8, 1, 4, 7, 2, 6]
  .filter(x => x % 2 === 0)
  .map(x => x * 2)
  .sort((a, b) => a - b)
// [4, 8, 12, 16]
{{< /highlight >}}

The JavaScript version looks similar to Elixir's pipes. The key difference is that JavaScript's chaining only works with methods that belong to the Array prototype. Elixir's pipe operator works with **any function**, even ones you write yourself.

### Example 3: Transforming a List of Strings

Here we process a list of names. Trim whitespace, capitalize each name, and remove any that are empty:

{{< highlight elixir >}}
["  alice ", "", "BOB", "  charlie  ", "  ", "diana"]
|> Enum.map(&String.trim/1)
|> Enum.reject(fn name -> name == "" end)
|> Enum.map(&String.capitalize/1)
# ["Alice", "Bob", "Charlie", "Diana"]
{{< /highlight >}}

Notice the `&String.trim/1` syntax. This is a **function capture**, shorthand for `fn x -> String.trim(x) end`. Since each element just needs to be passed directly to the function, we can use this shorter form.

## Contrast: Method Chaining vs Pipe Operator

OOP's method chaining and Elixir's pipe operator look similar but work differently:

**Method chaining (OOP):** Methods live on the object. You can only chain methods that the object knows about.

{{< highlight java >}}
// Java - chaining only works with the object's own methods
String result = "  HELLO  "
    .trim()
    .toLowerCase()
    .replace("hello", "hi");
// Can only call String methods
{{< /highlight >}}

**Pipe operator (FP):** Functions are independent. You can pipe into **any function** that takes the data as its first argument.

{{< highlight elixir >}}
# Elixir - pipe into any function, even your own
defmodule MyString do
  def add_greeting(name), do: "Hello, #{name}!"
end

"  alice  "
|> String.trim()
|> String.capitalize()
|> MyString.add_greeting()
# "Hello, Alice!"
{{< /highlight >}}

This matters. In OOP, if you want to add a new transformation, you need to extend the class or create a wrapper. In Elixir, you just write a function and pipe into it.

## Why Small, Composable Functions Matter

The pipe operator pushes you toward writing **small, focused functions** that do one thing well. Each function takes data in and returns data out. This makes them:

- **Easy to test** - each function can be tested on its own
- **Easy to reuse** - compose them differently for different tasks
- **Easy to read** - each function has a single responsibility

{{< highlight elixir >}}
defmodule TextProcessor do
  def remove_punctuation(text) do
    String.replace(text, ~r/[^\w\s]/, "")
  end

  def normalize_whitespace(text) do
    text
    |> String.trim()
    |> String.replace(~r/\s+/, " ")
  end

  def word_count(text) do
    text
    |> String.split(" ")
    |> length()
  end
end

# Now compose them freely:
"  Hello,   world!  How   are you?  "
|> TextProcessor.remove_punctuation()
|> TextProcessor.normalize_whitespace()
|> TextProcessor.word_count()
# 5
{{< /highlight >}}

Each function is simple on its own. The pipe operator lets you combine them in whatever order you need. You can rearrange, add, or remove steps without rewriting everything.

## Common Patterns and Gotchas

### Debugging with `IO.inspect`

You can insert `IO.inspect` anywhere in a pipeline to see intermediate values. Since `IO.inspect` returns the value it receives, it doesn't break the pipeline:

{{< highlight elixir >}}
[5, 3, 8, 1, 4, 7, 2, 6]
|> Enum.filter(fn x -> rem(x, 2) == 0 end)
|> IO.inspect(label: "after filter")
|> Enum.map(fn x -> x * 2 end)
|> IO.inspect(label: "after map")
|> Enum.sort()

# Output:
# after filter: [8, 4, 2, 6]
# after map: [16, 8, 4, 12]
# [4, 8, 12, 16]
{{< /highlight >}}

Very useful when you're trying to figure out what's happening at each step.

### When the First Argument Doesn't Fit

Sometimes the data argument isn't in the first position. For example, `String.replace/3` takes the string first (pipe-friendly), but what if you encounter a function where the data goes in a different position?

You can use an anonymous function:

{{< highlight elixir >}}
# If a function takes data as the second argument:
some_value
|> then(fn x -> some_function("fixed_arg", x) end)
{{< /highlight >}}

The `then/2` function exists for this. It passes the piped value into an anonymous function.

### Keep Pipelines Focused

If a pipeline grows beyond 5-7 steps, consider breaking it into named functions:

{{< highlight elixir >}}
# Instead of one long pipeline:
data
|> step1()
|> step2()
|> step3()
|> step4()
|> step5()
|> step6()
|> step7()

# Break it into meaningful chunks:
data
|> validate_input()
|> transform_data()
|> format_output()

# Where each function is its own small pipeline
defp validate_input(data) do
  data
  |> check_required_fields()
  |> normalize_values()
end
{{< /highlight >}}

## Key Takeaways

- **The pipe operator (`|>`)** passes the result of the left side as the first argument to the right side
- **It eliminates nested function calls**, making code read top-to-bottom instead of inside-out
- **Elixir's standard library** puts the data argument first so functions work naturally with pipes
- **Unlike method chaining**, pipes work with any function, including your own
- **Pipes push you toward small, composable functions** that do one thing well
- **Use `IO.inspect`** to debug intermediate values without breaking the pipeline
- **Use `then/2`** when you need to pipe into a function where the data isn't the first argument

## Try It Yourself

Open `iex` and practice these pipe operator exercises.

### Exercise 1: String Pipeline

Write a pipeline that takes a sentence, splits it into words, reverses the list of words, and joins them back with spaces:

{{< highlight elixir >}}
"the quick brown fox"
|> ???
|> ???
|> ???
# Should return "fox brown quick the"

# Hint: String.split/2, Enum.reverse/1, Enum.join/2
{{< /highlight >}}

### Exercise 2: Number Crunching Pipeline

Write a pipeline that takes a list of numbers from 1 to 10, keeps only odd numbers, squares each one, and sums them:

{{< highlight elixir >}}
1..10
|> Enum.to_list()
|> ???
|> ???
|> ???
# Should return 165 (1 + 9 + 25 + 49 + 81)

# Hint: Enum.filter/2, Enum.map/2, Enum.sum/1
{{< /highlight >}}

### Exercise 3: Build Your Own Pipeline Functions

Create a module with small functions that you can compose with pipes. Write functions to process a list of raw product names:

{{< highlight elixir >}}
defmodule ProductCleaner do
  def trim_all(products), do: ???
  def remove_empty(products), do: ???
  def capitalize_all(products), do: ???
  def sort_alphabetically(products), do: ???
end

["  widget ", "", " GADGET", "  gizmo  ", "  ", "Doohickey"]
|> ProductCleaner.trim_all()
|> ProductCleaner.remove_empty()
|> ProductCleaner.capitalize_all()
|> ProductCleaner.sort_alphabetically()
# Should return ["Doohickey", "Gadget", "Gizmo", "Widget"]
{{< /highlight >}}

## Official Documentation to Help You Learn

- [Elixir Pipe Operator Guide](https://elixir-lang.org/getting-started/enumerables-and-streams.html#the-pipe-operator)
- [Kernel.|>/2 Documentation](https://hexdocs.pm/elixir/Kernel.html#%7C%3E/2)
- [Enum Module Documentation](https://hexdocs.pm/elixir/Enum.html)

---

*Part Six | Functional Programming Through Elixir series*

**Previous in series:** [Part Five - Recursion Over Iteration](/posts/fp-series-recursion-over-iteration/)

**Next in series:** Part Seven - Higher-Order Functions (Coming Soon)
