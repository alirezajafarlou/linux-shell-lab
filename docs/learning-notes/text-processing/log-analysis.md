# Day 04 — Log Analysis and Command Composition

## Objective

Apply shell text-processing tools to access logs, count HTTP status codes, extract request methods, and investigate unusual records.

## Tools and Concepts Covered

- `awk` — field extraction, conditions, counters, and the `END` block.
- `sort` and `uniq -c` — count repeated values.
- `head` — display the first lines of output.
- Pipes — compose multiple commands into a processing pipeline.
- Redirection — save standard output and standard error.
- `&&` and `||` — execute commands conditionally.

## Dataset

The exercises used an anonymized access-log dataset from a real web service.

```text
IP272273 - - [03/Feb/2026:00:04:06 +0900] "HEAD / HTTP/1.1" 405 0 "https://doktor.tak-cslab.org/" "Mozilla/5.0+(compatible; UptimeRobot/2.0; http://www.uptimerobot.com/)"
```

The dataset contained **3,184 records**.

The client identifiers are anonymized and should not be interpreted as actual IP addresses.

## Exercises

### 1. Count HTTP status codes

```bash
awk '{print $9}' server.log |
    sort |
    uniq -c |
    sort -nr
```

Extract the ninth whitespace-separated field, count repeated values, and sort the results by descending frequency.

This assumes the status code occupies field 9 in the relevant log format.

### 2. Extract and count HTTP methods

```bash
awk -F'"' '{print $2}' server.log |
    awk '{print $1}' |
    sort |
    uniq -c |
    sort -nr
```

The first `awk` extracts the quoted request field. The second extracts the HTTP method, such as `GET`, `POST`, or `HEAD`.

### 3. Find the most frequent client identifiers

```bash
awk '{print $1}' server.log |
    sort |
    uniq -c |
    sort -nr |
    head -10
```

Identify the ten most frequent values in the first field.

High frequency alone does not establish whether a client is malicious or automated.

### 4. Count HTTP 404 responses

```bash
awk '$9 == 404 {count++} END {print count}' server.log
```

This introduces two useful `awk` concepts:

- `count++` increments a counter.
- `END` executes after all input records have been processed.

### 5. Count HTTP server errors

```bash
awk '$9 >= 500 && $9 <= 599 {count++} END {print count}' server.log
```

Count records whose ninth field contains a numeric value in the HTTP 5xx range.

### 6. Investigate unusual records

```bash
awk '$9 == "-" || $9 == 150 {print NR ": " $0}' server.log
```

Print records where the ninth field is `-` or `150`, including the original line number.

Inspect the complete records before deciding whether the cause is malformed input, unusual traffic, or an incorrect assumption about field positions.

## Observed Status-Field Values

| Value | Count |
|---|---:|
| `404` | 2,203 |
| `200` | 660 |
| `405` | 276 |
| `400` | 25 |
| `-` | 19 |
| `150` | 1 |

These counts sum to 3,184. They describe the observed ninth-field values; not every value necessarily represents a valid HTTP status code.

In the unusual record containing `mstshash=Administr`, the status was `400` and `150` was the response size. The request contained binary-looking data associated with RDP-related traffic, but the record alone does not establish the sender's identity or intent.

## Redirection and Command Chaining

### Output redirection

```bash
command > output.txt
command >> output.txt
command 2> errors.txt
command > output.txt 2>&1
```

- `>` redirects standard output and overwrites the destination.
- `>>` appends standard output.
- `2>` redirects standard error.
- `2>&1` redirects standard error to the current standard-output destination.

Redirection order matters when combining standard output and standard error.

### Conditional execution

```bash
command_a && command_b
command_a || command_b
```

- `&&` runs the second command if the first succeeds.
- `||` runs the second command if the first fails.

## Key Takeaways

- Field positions depend on the log format.
- Whitespace-based parsing can fail when fields contain spaces or records are malformed.
- Quoted-field extraction can help preserve request strings containing spaces.
- `awk` supports conditional filtering and basic aggregation without requiring a separate script.
- Unusual log entries should be investigated using the complete record rather than a single field.

## Next Step

Practice more complex log queries, improve parsing reliability, and turn frequently used command pipelines into reusable shell scripts.
