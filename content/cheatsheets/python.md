+++
title = "Python (CTF)"
date = "2026-03-06T00:00:00-05:00"
tags = ["python", "scripting"]
description = "Python snippets for encoding, crypto, pwntools, and CTF scripting."
draft = false
+++


## Encoding / Decoding
```python
import base64, binascii

# Base64
base64.b64encode(b"hello")           # b'aGVsbG8='
base64.b64decode("aGVsbG8=")        # b'hello'

# Hex
binascii.hexlify(b"hello")           # b'68656c6c6f'
binascii.unhexlify("68656c6c6f")    # b'hello'
bytes.fromhex("68656c6c6f")         # b'hello'
b"hello".hex()                       # '68656c6c6f'

# URL encode/decode
from urllib.parse import quote, unquote
quote("hello world")                 # 'hello%20world'
unquote("hello%20world")            # 'hello world'

# XOR
def xor(data, key):
    return bytes(a ^ b for a, b in zip(data, key * (len(data)//len(key)+1)))
```

## Integers & Bytes
```python
# int <-> bytes
n = int.from_bytes(b'\x00\xff', 'big')   # 255
b = n.to_bytes(2, 'big')                  # b'\x00\xff'

# int <-> hex string
hex(255)                # '0xff'
int('ff', 16)          # 255
int('0xff', 16)        # 255

# Bit operations
bin(42)                # '0b101010'
42 >> 1                # 21
42 << 1                # 84
42 & 0xFF              # 42
42 ^ 0xFF              # 213
```

## Crypto (PyCryptodome)
```python
from Crypto.Util.number import long_to_bytes, bytes_to_long, getPrime
from Crypto.Cipher import AES

# RSA helpers
n = p * q
e = 65537
phi = (p-1)*(q-1)
d = pow(e, -1, phi)    # modular inverse (Python 3.8+)
m = pow(c, d, n)       # decrypt
flag = long_to_bytes(m)

# AES CBC
cipher = AES.new(key, AES.MODE_CBC, iv)
ct = cipher.encrypt(pad(pt, 16))
cipher2 = AES.new(key, AES.MODE_CBC, iv)
pt = unpad(cipher2.decrypt(ct), 16)
```

## pwntools Quick Reference
```python
from pwn import *

# Connect
p = process('./binary')
p = remote('host', 1337)

# Send / Receive
p.sendline(b"input")
p.send(b"no newline")
p.recvuntil(b"prompt: ")
p.recvline()
p.recv(64)
data = p.recvall()

# Pack addresses
p64(0xdeadbeef)         # little-endian 8 bytes
p32(0xdeadbeef)         # little-endian 4 bytes
u64(b"\xef\xbe\xad\xde\x00\x00\x00\x00")  # unpack

# ROP
elf = ELF('./binary')
rop = ROP(elf)
rop.call(elf.plt['puts'], [elf.got['printf']])

# Cyclic pattern (for offset finding)
cyclic(200)
cyclic_find(0x6161616c)

# GDB attach
gdb.attach(p, gdbscript="b *main+50\nc")
```

## File / IO
```python
# Read bytes
data = open('file', 'rb').read()

# Write bytes
open('out', 'wb').write(data)

# Iterate lines
for line in open('file'):
    line = line.strip()
```

## Requests (web)
```python
import requests

s = requests.Session()
r = s.get('http://target/', params={'id': 1})
r = s.post('http://target/login', data={'user':'admin','pass':'pw'})
r = s.post('http://target/', json={'key': 'value'})

print(r.status_code, r.text)
s.cookies.set('session', 'abc123')
s.headers.update({'X-Custom': 'value'})
```

## Misc Useful
```python
from itertools import product
from string import printable

# Brute force single-char XOR key
ciphertext = bytes.fromhex("...")
for key in range(256):
    pt = bytes(b ^ key for b in ciphertext)
    if b'flag' in pt:
        print(key, pt)

# Generate all pairs
for a, b in product('abc', repeat=2):
    print(a, b)
```
