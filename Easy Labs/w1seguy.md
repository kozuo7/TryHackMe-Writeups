# TryHackMe W1seGuy Write-Up

## Overview

W1seGuy was a TryHackMe crypto room built around a TCP service running on port `1337`.

At first I thought this was going to be a super simple netcat challenge, but it ended up turning into a full breakdown of XOR, known-plaintext attacks, socket behavior, and building a small Python decoder.

The main issue was that the server used a short repeating XOR key. Since TryHackMe flags follow a predictable format, the key could be recovered from the encrypted output.

> Flags are redacted to follow TryHackMe write-up guidelines.

---

## Initial Connection

The room said the server was listening on TCP port `1337`, so I started with netcat:

```bash
nc <MACHINE_IP> 1337
```

The service returned something like:

```text
This XOR encoded text has flag 1: <HEX_OUTPUT>
What is the encryption key?
```

So the challenge was pretty clear:

1. Figure out the encryption key.
2. Send the key back to the server.
3. Get the second flag.

---

## Source Code Review

The useful part of the source code was this line:

```python
xored += chr(ord(flag[i]) ^ ord(key[i % len(key)]))
```

This showed that the server was XORing each character of the flag with a key.

The `i % len(key)` part was important because it meant the key repeated across the whole flag.

The key generation also showed the key was only 5 characters long:

```python
res = ''.join(random.choices(string.ascii_letters + string.digits, k=5))
key = str(res)
```

So the key would look something like this:

```text
uEEzC
```

And it would repeat across the flag like this:

```text
uEEzCuEEzCuEEzCuEEzC...
```

That made the encryption weak because the key was short, repeated, and the flag format was predictable.

---

## My First Mistake

My first approach was based on the fake flag shown in the source code:

```text
THM{thisisafakeflag}
```

The idea was:

```text
ciphertext XOR known_plaintext = key
```

That sort of worked at first, but only partially. The recovered key started off clean, then turned into garbage after a few characters.

The server also responded with:

```text
Close but no cigar
```

That told me the key I recovered was wrong.

After printing the ciphertext length, I noticed the encrypted output was `40` bytes, while the fake flag was only `20` bytes. So the live server was not encrypting that fake flag. It was encrypting a real flag-shaped string.

That was the point where I had to adjust the attack instead of forcing the fake flag assumption.

---

## Actual Attack

Instead of assuming the full plaintext, I only used the parts I knew for sure.

TryHackMe flags normally start with:

```text
THM{
```

And end with:

```text
}
```

Since the key was 5 characters long, that was enough to recover the full key.

The first four known characters recover the first four key characters:

```text
cipher[0] XOR 'T' = key[0]
cipher[1] XOR 'H' = key[1]
cipher[2] XOR 'M' = key[2]
cipher[3] XOR '{' = key[3]
```

Then the last encrypted byte can recover the final key character because the flag ends with `}`:

```text
cipher[-1] XOR '}' = key[4]
```

So the attack became:

```text
Use THM{ to recover key characters 0-3.
Use } to recover key character 4.
Use the full key to decrypt flag 1.
Send the key back to the server.
Get flag 2.
```

---

## Proof of Concept

This was the final working decoder:

```python
import socket
import re

HOST = "<MACHINE_IP>"
PORT = 1337

s = socket.socket()
s.settimeout(5)
s.connect((HOST, PORT))

data = s.recv(4096).decode()

print("[SERVER MESSAGE]")
print(data)

hex_cipher = re.search(r"flag 1: ([0-9a-fA-F]+)", data).group(1)
cipher = bytes.fromhex(hex_cipher)

print("Cipher length:", len(cipher))

key = [None] * 5

# THM flags start with THM{
known_start = b"THM{"

for i in range(len(known_start)):
    key[i % 5] = cipher[i] ^ known_start[i]

# THM flags end with }
last_index = len(cipher) - 1
key[last_index % 5] = cipher[last_index] ^ ord("}")

key = bytes(key).decode()
print("[RECOVERED KEY]", key)

flag1 = ''.join(
    chr(cipher[i] ^ ord(key[i % len(key)]))
    for i in range(len(cipher))
)

print("[DECODED FLAG 1]", flag1)

s.sendall((key + "\n").encode())

print("[SERVER RESPONSE]")

while True:
    try:
        chunk = s.recv(4096)

        if not chunk:
            break

        print(chunk.decode(), end="")

    except socket.timeout:
        break

s.close()
```

---

## Code Breakdown

The script connects to the server the same way netcat does:

```python
s = socket.socket()
s.connect((HOST, PORT))
```

Then it receives the server message:

```python
data = s.recv(4096).decode()
```

The server sends the encrypted flag as hex, so I used regex to pull only the hex string out:

```python
hex_cipher = re.search(r"flag 1: ([0-9a-fA-F]+)", data).group(1)
```

Then I converted the hex back into bytes:

```python
cipher = bytes.fromhex(hex_cipher)
```

After that, I created an empty 5-character key:

```python
key = [None] * 5
```

Then I used the known `THM{` flag prefix to recover the first four key characters:

```python
known_start = b"THM{"

for i in range(len(known_start)):
    key[i % 5] = cipher[i] ^ known_start[i]
```

Then I used the known ending `}` to recover the final key character:

```python
last_index = len(cipher) - 1
key[last_index % 5] = cipher[last_index] ^ ord("}")
```

Once the key was recovered, I decrypted flag 1 locally:

```python
flag1 = ''.join(
    chr(cipher[i] ^ ord(key[i % len(key)]))
    for i in range(len(cipher))
)
```

Finally, I sent the recovered key back to the server:

```python
s.sendall((key + "\n").encode())
```

That is basically the Python version of typing the key into netcat and pressing Enter.

---

## Debugging Issues

This room had a few annoying issues outside of the actual crypto.

At one point, the port showed as filtered even though the machine still responded to ping. That threw me off for a bit.

The useful checks were:

```bash
nc -nv -w 5 <MACHINE_IP> 1337
```

```bash
nmap -Pn -p 1337 --reason <MACHINE_IP>
```

The main lesson there was that ping only proves the host is alive. It does not prove the TCP service is reachable.

Another issue was socket timing. At first, the script recovered the key, but the output only showed:

```text
What is the encryption key?
```

The key had been sent, but the script only read one response chunk. The fix was to keep reading from the socket in a loop until the server stopped sending data or the timeout hit.

That is why the final script has this part:

```python
while True:
    try:
        chunk = s.recv(4096)

        if not chunk:
            break

        print(chunk.decode(), end="")

    except socket.timeout:
        break
```

---

## Result

The final script successfully:

- Connected to the TCP service.
- Extracted the encrypted hex output.
- Recovered the random 5-character XOR key.
- Decrypted flag 1 locally.
- Sent the key back to the server.
- Received flag 2.

Example output format:

```text
[SERVER MESSAGE]
This XOR encoded text has flag 1: <HEX_OUTPUT>

Cipher length: 40
[RECOVERED KEY] <REDACTED>
[DECODED FLAG 1] THM{REDACTED}
[SERVER RESPONSE]
What is the encryption key? Congrats! That is the correct key! Here is flag 2: THM{REDACTED}
```

---

## Takeaway

The main lesson from this room was that XOR is reversible, and short repeating XOR keys are weak when the plaintext format is predictable.

The important relationship is:

```text
ciphertext XOR known_plaintext = key
```

This was not a beta break or unintended shortcut. It was the intended XOR weakness, but instead of only solving it manually with a tool like CyberChef, I built a full Python proof of concept.

For an “easy” room, it still made me work through a lot:

- reading source code
- understanding XOR
- recognizing a known-plaintext attack
- debugging bad assumptions
- dealing with socket timing
- troubleshooting filtered ports

So the concept was simple, but the process was actually solid practice.
