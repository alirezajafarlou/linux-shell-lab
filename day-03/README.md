# Day 03 — Shell Text Processing

## Objective

Practice processing text from files and command output using standard Unix utilities and pipelines.

## Tools Covered

- `grep` — filter lines matching a pattern.
- `cut` — extract fields using a delimiter.
- `awk` — extract fields and evaluate conditions.
- `sort` — sort lines alphabetically or numerically.
- `uniq` — identify adjacent duplicate lines.
- `wc` — count lines and other input statistics.
- `sed` — transform text using substitution.
- `tee` — display output while writing it to a file.

## Exercises

### 1. Filter log entries

```bash
grep "500" server.log
```

Display lines containing `500`.

```bash
grep -v "500" server.log
```

Display lines that do not contain `500`.

**Note:** These commands perform text matching, not structured HTTP status-code parsing.

### 2. Extract fields

```bash
awk '{print $1}' server.log
```

Print the first whitespace-separated field from each line.

```bash
cut -d' ' -f1 server.log
```

Extract the first field using a literal space as the delimiter.

These commands may behave differently when input spacing is inconsistent.

### 3. Count repeated values

```bash
awk '{print $1}' server.log |
    sort |
    uniq -c |
    sort -nr
```

This pipeline:

1. Extracts the first field.
2. Sorts the values so identical entries are adjacent.
3. Counts each group with `uniq -c`.
4. Sorts the counts numerically in descending order.

`uniq` only combines adjacent identical lines, which makes sorting first important.

### 4. Substitute text

```bash
sed 's/500/ERROR/' server.log
```

Replace the first occurrence of `500` on each line in the output.

```bash
sed 's/500/ERROR/g' server.log
```

Replace every occurrence on each line.

Without `-i`, these commands do not modify the original file.

### 5. Display and save output

```bash
grep "404" server.log | tee matches.log
```

Display matching lines and save them to `matches.log`.

```bash
grep "404" server.log | tee -a matches.log
```

Append matching lines instead of overwriting the existing file.

## Key Takeaways

- Pipes connect commands into reusable data-processing workflows.
- `awk` is useful for field extraction and basic data selection.
- `sort` and `uniq -c` work together to count repeated values.
- `sed` can transform output without modifying the source file.
- Understanding delimiters and whitespace is essential when parsing text.

## Next Step

Apply these tools to real access logs and use conditional expressions to analyze specific fields.
