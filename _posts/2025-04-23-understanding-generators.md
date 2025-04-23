# Understanding Generator Expressions vs List Comprehensions in Python

Python offers powerful tools to manipulate and process data elegantly. In this blog post, we'll walk through an interesting use case involving `tokenized` data and see how generator expressions can offer a clean and efficient alternative to more verbose implementations.

## The Problem

Imagine you have a list of tokenized sentences, where each item is a tuple consisting of a list of words and a float score:

```python
# Sample tokenized data
tokenized = [
    (["check", "whether", "the", "battery", "ran", "down", "please", "<END>"], 0.4),
    (["check", "whether", "the", "program", "ran", "please", "<END>"], 0.6)
]
```

Our goal is to extract all **unique** words from these tokenized sentences and sort them lexicographically to form a vocabulary.

## The Verbose Way

One way to solve this is to use a set to collect unique words and then sort them:

```python
# Initialize an empty set
unique_words = set()

# Loop through each sentence in the tokenized list
for sentence, _ in tokenized:
    for word in sentence:
        unique_words.add(word)

# Convert the set to a sorted list
vocab = sorted(unique_words)
```

This works perfectly, but there's a more concise way.

## The Pythonic Way – Using Generator Expressions

```python
vocab = sorted(set(word for s, _ in tokenized for word in s))
```

### Why Does This Work?

1. `for s, _ in tokenized` — iterates through each tuple in `tokenized`, unpacking the sentence list into `s`.
2. `for word in s` — goes through each word in the sentence.
3. `word` — the expression returned for each loop iteration.
4. `set(...)` — collects all unique words.
5. `sorted(...)` — arranges the unique words in lexicographical order.

### Equivalent Code Using `yield`

You can also achieve the same with a generator function:

```python
def yield_words(tokenized_data):
    for sentence, _ in tokenized_data:
        for word in sentence:
            yield word

vocab = sorted(set(yield_words(tokenized)))
```

This version can be helpful if you're looking to modularize the logic or need to reuse the generator elsewhere.

## List Comprehensions vs Generator Expressions

Here's a quick comparison:

### List Comprehension
```python
squares = [x**2 for x in range(5)]
print(squares)  # [0, 1, 4, 9, 16]
```
- Allocates memory for the full list immediately.
- Faster access if you need to reuse the list multiple times.

### Generator Expression
```python
squares_gen = (x**2 for x in range(5))
print(squares_gen)  # <generator object>
for square in squares_gen:
    print(square)
```
- Lazily evaluated — useful for large datasets.
- More memory efficient.

### Use Case in Summing
```python
result = sum(x**2 for x in range(5))
print(result)  # 30
```
- No intermediate list is created — just yields values on demand.

## Using `yield` to Emit Multiple Values
In some cases it may be necessary to return more than one value. In that case we can use `yield` as generator expressions are useful for only returning one value.
```python
def generate_squares_and_cubes(limit):
    for x in range(limit):
        yield x**2
        yield x**3

for val in generate_squares_and_cubes(5):
    print(val)
```

This shows how `yield` can be used to produce multiple values in sequence.


