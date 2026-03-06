+++
title = "Bash"
date = "2026-03-06T00:00:00-05:00"
tags = ["bash", "scripting"]
description = "Quick reference for bash one-liners and scripting patterns useful in CTFs."
draft = false
+++


## One-liners
```bash
# Hex encode/decode
echo -n "hello" | xxd -p
echo "68656c6c6f" | xxd -r -p

# Base64
echo -n "hello" | base64
echo "aGVsbG8=" | base64 -d

# Reverse a string
echo "hello" | rev

# Extract flag pattern
grep -oP 'flag\{[^}]+\}' file.txt

# Count + sort occurrences
sort file.txt | uniq -c | sort -rn

# URL encode
python3 -c "import urllib.parse; print(urllib.parse.quote('hello world'))"
```

## Loops
```bash
# Range loop
for i in {1..100}; do echo $i; done

# Read file line by line
while IFS= read -r line; do
    echo "$line"
done < file.txt

# Fuzz with wordlist
for word in $(cat wordlist.txt); do
    code=$(curl -s -o /dev/null -w "%{http_code}" "http://target/$word")
    [[ "$code" != "404" ]] && echo "$word: $code"
done

# Background parallel jobs
for i in {1..10}; do (cmd $i &); done; wait
```

## String Manipulation
```bash
str="hello world"
echo ${str:0:5}        # hello (substring)
echo ${str: -5}        # world (from end)
echo ${str/hello/hi}   # hi world (replace)
echo ${#str}           # 11 (length)

# Split on delimiter
IFS=',' read -ra arr <<< "a,b,c"
for item in "${arr[@]}"; do echo $item; done
```

## File Operations
```bash
content=$(<file.txt)                         # read into var
diff <(cmd1) <(cmd2)                         # process substitution
find . -name "*.txt" -print0 | xargs -0 grep "pattern"
```

## Networking
```bash
nc -lvnp 4444                                # listen
nc target 4444 < file                        # send file
nc -zv target 20-1000 2>&1 | grep succeeded  # port scan

curl -v -X POST -d "a=1&b=2" http://target/
curl -b "session=abc" -H "X-Header: val" http://target/
curl -L -o out.bin http://target/file        # follow redirects, save
```

## Scripting Boilerplate
```bash
#!/bin/bash
set -euo pipefail

usage() { echo "Usage: $0 -u URL [-p PORT]"; exit 1; }

while getopts "u:p:h" opt; do
    case $opt in
        u) URL="$OPTARG" ;;
        p) PORT="${OPTARG:-80}" ;;
        *) usage ;;
    esac
done
[[ -z "${URL:-}" ]] && usage

cleanup() { rm -f /tmp/tmp.$$; }
trap cleanup EXIT
```
