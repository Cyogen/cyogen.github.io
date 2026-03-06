+++
title = "Regex"
date = "2026-03-06T00:00:00-05:00"
tags = ["regex"]
description = "Regex syntax and patterns for CTF grep, parsing, and flag extraction."
draft = false
+++


## Anchors & Boundaries
```
^           start of string/line
$           end of string/line
\b          word boundary
\B          non-word boundary
```

## Character Classes
```
.           any char except newline
\d  [0-9]   digit
\D  [^0-9]  non-digit
\w  [a-zA-Z0-9_]    word char
\W          non-word char
\s  [ \t\n\r\f\v]   whitespace
\S          non-whitespace
[abc]       a, b, or c
[^abc]      not a, b, or c
[a-z]       lowercase letter
```

## Quantifiers
```
*           0 or more (greedy)
+           1 or more (greedy)
?           0 or 1
{n}         exactly n
{n,}        n or more
{n,m}       between n and m
*?  +?  ??  non-greedy versions
```

## Groups & Lookaround
```
(abc)       capturing group
(?:abc)     non-capturing group
(?P<name>)  named group
(?=abc)     positive lookahead
(?!abc)     negative lookahead
(?<=abc)    positive lookbehind
(?<!abc)    negative lookbehind
```

## Common CTF Patterns
```python
import re

# Extract flag
re.findall(r'flag\{[^}]+\}', text)
re.findall(r'HTB\{[^}]+\}', text)
re.findall(r'[A-Z]{2,5}\{[^}]+\}', text)   # any CTF format

# IP address
re.findall(r'\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b', text)

# Email
re.findall(r'[\w.+-]+@[\w-]+\.[a-zA-Z]+', text)

# URLs
re.findall(r'https?://[^\s"\'<>]+', text)

# Base64 blob
re.findall(r'[A-Za-z0-9+/]{20,}={0,2}', text)

# Hex string
re.findall(r'(?:0x)?[0-9a-fA-F]{8,}', text)

# Extract between tags
re.findall(r'<tag>(.*?)</tag>', text, re.DOTALL)
```

## Python Usage
```python
import re

# Search (first match)
m = re.search(r'(\d+)', text)
if m:
    print(m.group(0))   # full match
    print(m.group(1))   # first group

# Find all
matches = re.findall(r'\d+', text)

# Sub
result = re.sub(r'\s+', ' ', text)

# Named groups
m = re.search(r'(?P<user>\w+):(?P<pass>\w+)', text)
m.group('user')
m.group('pass')

# Flags
re.IGNORECASE  # re.I
re.MULTILINE   # re.M  (^ $ match per line)
re.DOTALL      # re.S  (. matches \n)
re.VERBOSE     # re.X  (allow comments/whitespace)
```

## Grep with Regex
```bash
grep -oP 'flag\{[^}]+\}' file.txt     # Perl regex, print match only
grep -oE '[0-9a-f]{32}' file.txt       # MD5 hashes
grep -P '(?<=secret=)\w+' file.txt     # lookbehind
```
