# Regular Expressions

Regular expressions (often shortened to **RegEx**) are one of the most powerful tools you can add to your toolkit. They allow you to **search, match, filter, and manipulate text based on patterns**, rather than exact words. Once you are comfortable with them, tasks that would normally require complex logic become simple one-liners.

You can think of RegEx as a set of rules that describe *what you are looking for*, rather than *where you are looking*. Instead of searching for a fixed string, you describe the structure of the text you want to match.

Regular expressions are supported across many tools and languages, including `grep`, `sed`, `awk`, Python, and others. This makes them extremely useful during enumeration, log analysis, input validation, and data extraction.

At a basic level, a regular expression is just a sequence of characters that form a **search pattern**. What makes RegEx powerful are *metacharacters*—special symbols that describe behaviour instead of literal text. These allow you to match digits, ranges of characters, repetitions, or combinations of patterns.

---

## Grouping in Regular Expressions

One of the most important concepts in RegEx is **grouping**. Grouping lets you treat multiple characters or expressions as a single unit and apply rules to them collectively.

Regular expressions mainly use three types of brackets, each with a different purpose.

### Grouping Operators Overview

| Operator | Description                                                          |
| -------- | -------------------------------------------------------------------- |
| `(a)`    | Round brackets group parts of a regex so they are processed together |
| `[a-z]`  | Square brackets define character classes                             |
| `{1,10}` | Curly brackets define how many times a pattern should repeat         |
| `\|`     | OR operator – matches one pattern or another                         |
| `.*`     | Matches any characters in sequence (used for ordered matching)       |

Each of these serves a different role, and combining them is where RegEx becomes truly powerful.

---

## Character Classes `[ ]`

Square brackets define a **set of allowed characters**.

Examples:

* `[abc]` → matches `a`, `b`, or `c`
* `[a-z]` → matches any lowercase letter
* `[0-9]` → matches any digit

Only one character is matched, but it must be one from the defined set.

---

## Quantifiers `{ }`

Curly brackets define **how often something should repeat**.

Examples:

* `{1}` → exactly once
* `{1,3}` → between 1 and 3 times
* `{10}` → exactly 10 times

Quantifiers always apply to the element immediately before them.

---

## OR Operator `|`

The OR operator allows you to match **one pattern or another**.

To use it with `grep`, you must enable extended regular expressions using `-E`.

### Example: OR Matching

Search for lines containing either `my` **or** `false`:

```text
grep -E "(my|false)" /etc/passwd
```

Example output:

```text
lxd:x:105:65534::/var/lib/lxd/:/bin/false
pollinate:x:109:1::/var/cache/pollinate:/bin/false
mysql:x:116:120:MySQL Server,,:/nonexistent:/bin/false
```

Each of these lines contains at least one of the two search terms, so all of them are returned.

---

## Simulating an AND Condition

Regular expressions do not have a dedicated AND operator. Instead, you **control order** using patterns like `.*`, which matches any number of characters in between two expressions.

### Example: AND Matching

Search for lines that contain **both** `my` **and** `false`, in that order:

```text
grep -E "(my.*false)" /etc/passwd
```

Output:

```text
mysql:x:116:120:MySQL Server,,:/nonexistent:/bin/false
```

Here, you are explicitly saying:

* Match `my`
* Followed by any characters
* Followed by `false`

Only one line satisfies that condition.

---

## Alternative AND Approach (Chained Grep)

You can also achieve an AND condition by filtering twice:

```text
grep -E "my" /etc/passwd | grep -E "false"
```

This works because:

1. The first `grep` keeps lines containing `my`
2. The second `grep` removes anything that does not contain `false`

While this approach is sometimes easier to read, learning to express logic directly in RegEx gives you more control and flexibility.

---

## Why This Matters

Regular expressions are not about memorising syntax. They are about **thinking in patterns**.

Once you understand grouping, repetition, and ordering, you can:

* Extract specific fields from large files
* Identify misconfigurations quickly
* Filter logs efficiently
* Build precise enumeration pipelines

You will see RegEx used everywhere in penetration testing. The earlier you become comfortable with it, the more natural your command-line workflow will feel.

Treat RegEx as a skill you refine over time. Start small, experiment often, and always test your patterns against real data.
